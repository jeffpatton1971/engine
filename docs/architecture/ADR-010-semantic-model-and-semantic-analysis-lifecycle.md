# ADR-010: Semantic Model and Semantic Analysis Lifecycle

- **Status:** Proposed
- **Date:** 2026-09-05
- **Decision Type:** Architecture exploration

## Context

Engine needs to transform source-neutral Parsed Intent into a deterministic Infrastructure IR / Resource Graph without becoming aware of the infrastructure-specific semantics of Azure, AWS, GCP, VCFA, Kubernetes, or future platforms.

Infrastructure Integrations own those domain semantics and publish Domain Abstractions containing the concrete strongly typed resources that ultimately participate in the Resource Graph.

The Semantic Model is therefore the versioned semantic contract an Integration exposes to Engine. It gives Parsed Intent domain meaning, but it does not own graph mechanics and it does not require Engine to understand Integration-specific CLR resource implementations.

A key design risk is making the Semantic Model a universal declarative schema capable of describing every infrastructure platform. That would move domain complexity into Engine and create a new meta-language that every Integration would have to fit.

Another risk is allowing Integrations to construct their own graph machinery. If Integrations perform identity registration, reference lookup, dependency traversal, cycle detection, or graph ordering themselves, Engine would no longer have one authoritative graph implementation and deterministic behavior would become difficult to guarantee.

## Proposed decision

Engine defines the **semantic operations and lifecycle** an Integration must provide, not the Integration's internal representation of its infrastructure domain.

An Integration MAY implement its Semantic Model using:

- hand-written code;
- declarative definitions;
- generated definitions;
- reflection;
- source generators;
- upstream API schemas;
- provider metadata;
- internal libraries;
- or any combination of these techniques.

Engine SHALL depend only on the published semantic contract and observable behavior.

The initial semantic responsibilities are intentionally small:

1. recognize an infrastructure resource type;
2. materialize a concrete typed domain resource;
3. construct its canonical `ResourceIdentity`;
4. construct typed `ResourceReference<TResource>` values where the domain resource refers to other resources;
5. validate resource-local and domain-local semantics;
6. describe semantic relationships after references can be resolved;
7. derive prerequisite dependency semantics.

The exact C# interface is intentionally deferred. Required behavior should be pressure-tested with real infrastructure models before API shape is frozen.

## Semantic analysis is multi-phase

Semantic Analysis SHALL be modeled as an ordered multi-phase lifecycle rather than a single opaque `Analyze()` operation.

Conceptually:

```text
Parsed Intent
     |
     v
1. Materialization
   Integration / Semantic Model
     |
     +-- recognize resource type
     +-- construct concrete typed resource
     +-- construct canonical ResourceIdentity
     +-- construct ResourceReference<T>
     +-- validate resource-local/domain-local semantics
     |
     v
2. Identity Registration
   Engine
     |
     +-- collect all materialized resources
     +-- enforce ResourceIdentity uniqueness
     |
     v
3. Reference Resolution
   Engine
     |
     +-- resolve ResourceReference<T>
     +-- verify target exists
     +-- verify expected resource type
     |
     v
4. Semantic Analysis
   Integration / Semantic Model
     |
     +-- interpret resolved resources/references
     +-- declare semantic relationships
     +-- declare prerequisite dependencies
     +-- provide domain provenance/explanation when appropriate
     |
     v
5. Graph Construction and Validation
   Engine
     |
     +-- construct resolved relationship edges
     +-- construct and deduplicate dependency edges
     +-- validate graph integrity
     +-- detect cycles
     +-- establish deterministic ordering
     |
     v
Infrastructure IR / Resource Graph
```

The phases describe ownership and observable behavior. They do not yet require each phase to map one-to-one to a public interface method.

## Phase 1: Materialization

The Integration / Semantic Model owns interpretation of domain-specific Intent and creation of concrete Integration-owned resource instances.

For example, Parsed Intent might contain a source-neutral declaration conceptually resembling:

```yaml
type: azure.virtual-machine
name: web01
properties:
  size: Standard_D4s_v5
  resourceGroup: production
  networkInterface: web01-nic
```

The Azure Integration may materialize this as a strongly typed resource such as:

```csharp
VirtualMachineResource
{
    Identity = ...,
    Size = ...,
    ResourceGroup = ResourceReference<ResourceGroupResource>(...),
    NetworkInterface = ResourceReference<NetworkInterfaceResource>(...)
}
```

Engine SHALL NOT contain Azure-specific materialization logic or compile-time knowledge of `VirtualMachineResource`.

The Integration creates typed references because it understands what a property such as `networkInterface` means. It does not resolve those references against the graph during materialization.

## Validation ownership

Validation is divided by architectural ownership rather than collected into a single generic validation stage.

### Integration / Semantic Model validation

The Integration validates the resource itself and domain-local semantics.

Examples include:

- required fields have values;
- property values have valid domain types;
- numeric or size ranges are valid;
- enum-like values are supported;
- mutually exclusive fields are not supplied together;
- required combinations of fields are present;
- domain-specific invariants that can be evaluated without graph resolution are satisfied.

The governing rule is:

> Integrations validate meaning and domain correctness.

### Engine validation

Engine validates identity, resolution, and graph integrity.

Examples include:

- `ResourceIdentity` values are unique;
- referenced resources exist in the Intent-derived resource set;
- a resolved reference targets the expected resource type;
- relationship endpoints exist;
- dependency prerequisites exist;
- dependency cycles are detected;
- graph traversal and ordering are deterministic.

The governing rule is:

> Engine validates identity, resolution, and graph correctness.

### Target validation

Target-specific validation occurs after a Backend lowers the Resource Graph into the Target model.

A Target validates concerns that belong to its deployment technology, such as target expression validity, target model constraints, or emission requirements.

Therefore validation has three distinct ownership layers:

```text
Integration validation
    domain correctness

Engine validation
    graph correctness

Target validation
    target-model correctness
```

A component SHALL NOT take ownership of validation merely because it happens to have enough information to perform it when that validation belongs to another architectural boundary.

## Validation aggregation and fail-fast behavior

Validation SHOULD report the maximum useful set of independent failures that can be discovered safely in a single compilation pass.

A validation phase SHALL NOT stop after the first ordinary validation failure when additional independent checks can still be evaluated without relying on invalid state.

For example, if an Azure VM is missing `size`, has an invalid local naming value, and violates an unrelated disk-size rule, Azure Integration SHOULD return all applicable resource-local diagnostics from that validation pass rather than forcing the engineer through repeated compile/fix/compile cycles.

Conceptually:

```text
Integration validation
    check required fields
    check value constraints
    check cross-field rules
    collect diagnostics
        |
        v
return resource result + diagnostic set
```

The same principle applies to Engine graph validation where safe. Engine SHOULD collect multiple unresolved references, duplicate identities, or other independent graph failures rather than reporting only the first one encountered.

This does not mean every phase must continue after every error. A phase MAY stop processing a particular resource, subgraph, or downstream phase when continuing would create misleading diagnostics, require invalid state, or risk nondeterministic behavior.

Examples of legitimate early termination include:

- source or Parsed Intent is structurally unreadable;
- a resource cannot be materialized sufficiently to identify its type or identity;
- duplicate identity makes a specific reference inherently ambiguous;
- graph construction cannot proceed meaningfully because required structural invariants are absent;
- a fatal internal contract violation indicates an Integration or Engine defect rather than user-correctable Intent.

The governing rule is:

> Fail fast on unrecoverable structural conditions; aggregate independent user-correctable validation failures whenever safe.

Validation APIs SHOULD therefore prefer structured result/diagnostic collections over using exceptions as ordinary validation-flow control. Exceptions remain appropriate for unexpected runtime failures and broken implementation contracts.

## Reference creation versus reference resolution

The Integration and Engine have intentionally different responsibilities for resource references.

The Integration knows that a domain value refers to a particular kind of resource and therefore constructs the typed reference:

```csharp
ResourceReference<NetworkResource>
```

Engine owns resolving that identity against the complete resource set and determining whether the target exists and is of the expected type.

Therefore:

> The Integration supplies the meaning of a reference; Engine supplies reference-resolution mechanics.

This avoids requiring the Integration to maintain its own resource registry or graph lookup implementation.

## Relationship and dependency derivation

Relationship and dependency semantics are evaluated after Engine has collected resources and can resolve references.

This is important because source order SHALL NOT affect semantic meaning.

For example, a virtual machine may reference a network declared before or after it in YAML, JSON, an API payload, or another Adapter representation. Equivalent Intent MUST produce an equivalent Resource Graph regardless of declaration order.

The Integration / Semantic Model owns:

- what relationships mean in its infrastructure domain;
- which resolved references imply semantic relationships;
- whether a semantic relationship implies prerequisite behavior;
- domain-specific prerequisite rules that create justified dependency-only edges;
- domain-facing provenance explaining why those facts were derived.

Engine owns:

- resolving referenced identities;
- constructing the resulting graph edges;
- deduplicating structural dependency edges;
- retaining separate provenance where diagnostics or explainability require it;
- graph integrity and deterministic analysis.

The governing rule is:

> The Integration describes domain semantics; it does not construct or own the Resource Graph.

## Source-order independence

The order in which resources appear in source Intent SHALL NOT affect the resulting semantic meaning or Resource Graph.

Adapters may preserve source location and ordering metadata for diagnostics where useful, but source declaration order SHALL NOT be used as an implicit dependency mechanism.

All resources are materialized and registered before graph-level reference resolution and semantic relationship/dependency analysis are considered complete.

This permits the same infrastructure Intent to be represented through different Adapters while preserving equivalent semantic output.

## Semantic Model versus universal infrastructure schema

Engine SHALL NOT require Integrations to encode their complete domain into an Engine-owned universal resource taxonomy or universal declarative infrastructure schema.

Engine defines the semantic protocol necessary to perform analysis. The Integration owns how it represents and implements its domain internally.

This distinction allows an Integration to use rich strongly typed CLR resources, generated models, upstream schemas, or other implementation strategies without requiring changes to Engine Core merely because the infrastructure platform evolves.

Adding a new resource type to an Integration SHOULD NOT require an Engine Core change when the existing generic semantic and graph contracts remain sufficient.

## Guardrails

- Engine SHALL NOT contain infrastructure-platform-specific materialization logic.
- Integration implementations SHALL NOT own an independent Resource Graph implementation.
- Integration implementations SHALL NOT perform source-order-dependent reference resolution.
- Resource-local/domain-local validation belongs to the Integration.
- Identity, reference-resolution, and graph-integrity validation belongs to Engine.
- Target-model validation belongs to the Target.
- Integrations create typed resource references; Engine resolves them.
- Integrations define relationship and dependency semantics; Engine constructs and analyzes the resulting graph edges.
- Source declaration order SHALL NOT imply dependency order.
- Equivalent Intent expressed through different Adapters SHOULD produce equivalent semantic resources and graph structure.
- Independent user-correctable validation failures SHOULD be aggregated within a phase when continuing is safe.
- Exceptions SHOULD NOT be the ordinary mechanism for reporting expected validation failures.
- Engine SHALL define required semantic behavior before freezing a detailed Semantic Model API.
- Engine SHALL NOT introduce a universal infrastructure taxonomy merely to simplify Semantic Model implementation.

## Consequences

### Positive

- Engine remains infrastructure-domain neutral.
- Integration developers retain control over how their domain models are authored or generated.
- Strongly typed Domain Abstractions can evolve without requiring Engine to understand each concrete resource type.
- One Engine-owned graph implementation remains authoritative for resolution, cycles, ordering, and traversal.
- Validation ownership is explicit and testable.
- Engineers can receive multiple independent validation failures in one run rather than repeatedly fixing one error at a time.
- Forward references and arbitrary source declaration order are naturally supported.
- Semantic Models can evolve with their upstream platforms without turning Engine into a universal cloud schema registry.
- The architecture remains suitable for third-party Integrations built without references to Engine implementation assemblies.

### Negative / risks

- The semantic lifecycle requires multiple phases and therefore more orchestration than a single `Analyze()` method.
- Integration authors must distinguish local semantic validation from graph-level validation.
- Aggregated validation requires careful suppression of cascading or misleading diagnostics when prerequisite state is invalid.
- Engine must expose enough resolved-resource context for semantic rules without leaking graph ownership back into the Integration.
- The eventual Semantic Model API needs careful design to preserve compile-time ergonomics without over-generalizing the contract.
- Diagnostics may span multiple ownership layers and will require a coherent common diagnostic/provenance model.

## Open questions

- What is the minimum public Semantic Model interface that supports this lifecycle without exposing unnecessary graph internals?
- How does Engine present resolved resource/reference context to Integration semantic rules during Phase 4?
- Are semantic rules resource-scoped, model-scoped, or both?
- How are relationship kinds represented while remaining Integration-owned and Engine-neutral?
- What is the common structured diagnostic contract used to aggregate Integration, Engine, Backend, and Target validation failures?
- How does semantic-rule provenance integrate with graph diagnostics and the Artifact Bundle?
- How should semantic analysis behave when some resources fail materialization or local validation?
- What rules suppress cascading diagnostics when prerequisite validation has already failed?
- What portions of the Semantic Model contract need independent versioning or conformance testing?

## Next validation step

Before freezing the public API, pressure-test this lifecycle using a small real infrastructure model containing multiple resource types and references.

A useful first scenario is an Azure model containing:

```text
Resource Group
Network / Virtual Network
Network Interface
Managed Disk
Virtual Machine
```

The scenario should exercise all five phases, forward references, relationship derivation, dependency derivation, missing references, wrong-type references, local validation failures, and deterministic graph ordering.
