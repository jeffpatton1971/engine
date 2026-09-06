# ADR-008: Domain Abstractions and Typed Resource Graphs

- **Status:** Proposed
- **Date:** 2026-09-05
- **Decision Type:** Architecture exploration

## Context

Engine must construct a deterministic Infrastructure IR from Parsed Intent and an Integration-owned Semantic Model. Integration Backends must then consume that IR with enough domain information and type safety to map infrastructure semantics into a Target contract.

A purely generic property bag such as `Dictionary<string, object>` would keep Engine independent from individual infrastructure domains, but would move many structural errors to runtime and make Backend development unnecessarily weakly typed. At the opposite extreme, making Engine compile directly against every Integration's concrete CLR resource model would couple Engine releases to infrastructure-domain releases and defeat independent extension ownership.

The architecture needs a boundary that permits domain-owned strongly typed resources while allowing Engine to operate through stable common contracts.

## Proposed decision

Each Infrastructure Integration SHOULD publish a separately consumable **Domain Abstractions** contract containing the public resource types and domain contracts shared between the Integration and its Backends.

For example:

```text
SddcFlex.Abstractions
SddcFlex.Integration
SddcFlex.Backend.Terraform
```

Domain Abstractions SHALL implement or compose the common resource and graph contracts published by Engine Abstractions.

Engine SHALL construct and operate on the Resource Graph through Engine-level abstractions. Engine SHALL NOT take a compile-time dependency on a specific Integration's Domain Abstractions.

Integration Backends MAY reference their Integration's Domain Abstractions directly so they can consume resolved domain resources with strong typing.

The Resource Graph SHALL preserve the concrete Integration-owned typed resource instances produced by Semantic Analysis. Engine's minimal resource contract defines only the surface Engine itself requires; it does not flatten, discard, or replace domain-specific properties.

Conceptually:

```text
                    Engine.Abstractions
                     ^            ^
                     |            |
          SddcFlex.Abstractions   |
              ^          ^        |
              |          |        |
SddcFlex.Integration     SddcFlex.Backend.Terraform
                                  |
                                  v
                       Terraform.Target.Abstractions
```

The dependency arrows represent compile-time contract dependencies. Engine discovers and interacts with Integration resources through Engine abstractions rather than referencing `SddcFlex.Abstractions` directly.

## Ownership

| Component | Responsibility |
| --- | --- |
| Adapter | Translate an external Source representation into Parsed Intent. |
| Domain Abstractions | Define the Integration's public resource types and stable domain contracts shared with Backends. |
| Integration / Semantic Model | Materialize typed resources, construct canonical identities/references, validate domain-local semantics, and define relationship/prerequisite semantics. |
| Engine | Orchestrate Semantic Analysis, enforce graph identity uniqueness, resolve references, construct relationships/dependency edges, validate graph integrity, and produce the deterministic Resource Graph. |
| Integration Backend | Lower resolved domain resources from the Resource Graph into one Target contract. |
| Target Abstractions | Define the public Target model and compatibility contract. |
| Target | Validate and emit the Target representation. |

The governing distinction is:

> The Integration defines and applies domain meaning; Engine resolves that meaning into a Resource Graph; the Backend lowers those resolved domain resources into a Target contract.

Engine owns graph structure and resolution mechanics. Integrations own the typed resource payloads and domain semantics represented within that graph. Backends own translation from those typed resources into Target-specific models.

## Semantic Model and Resource Graph

The Semantic Model and Resource Graph are related but distinct.

A Semantic Model defines and applies resource semantics. The Resource Graph contains resolved resource instances and Engine-owned resolved edges.

The lifecycle is deliberately multi-phase as defined by ADR-010:

```text
Parsed Intent
    |
    v
Integration materialization + local validation
    |
    v
Engine identity registration + reference resolution
    |
    v
Integration relationship/prerequisite analysis
    |
    v
Engine graph construction + validation
    |
    v
Resource Graph
```

For example, an Integration may materialize a `VirtualMachineResource` containing a typed `ResourceReference<NetworkResource>`. The Integration understands that the property is a network reference and creates the typed reference; Engine resolves its identity against the complete resource set. After resolution, Integration semantics determine whether the reference represents `uses-network` and whether that relationship implies a prerequisite. Engine constructs the resulting graph edges.

This division prevents the Integration from becoming a second graph implementation while keeping domain meaning outside Engine.

## Typed Resource Graph

The Resource Graph SHALL have a common Engine-owned structural contract while containing Integration-owned strongly typed resource instances.

The minimum Engine resource contract SHOULD remain deliberately small. An illustrative shape is:

```csharp
public interface IResource
{
    ResourceIdentity Identity { get; }
    ResourceType Type { get; }
}
```

`ResourceIdentity` is defined semantically by ADR-009 as the tuple of Integration identity, Resource Type, and Integration-owned canonical Resource Key. Engine enforces uniqueness of that tuple without understanding cloud-specific scoping rules.

An Integration's Domain Abstractions may expose a complete strongly typed resource such as:

```csharp
public sealed record VirtualMachineResource : IResource
{
    public required ResourceIdentity Identity { get; init; }
    public ResourceType Type => SddcFlexResourceTypes.VirtualMachine;

    public required string Name { get; init; }
    public required int CpuCount { get; init; }
    public required MemorySize Memory { get; init; }
    public required ResourceReference<NetworkResource> Network { get; init; }
    public required StorageProfile StorageProfile { get; init; }
}
```

These interfaces and types are illustrative rather than accepted API contracts.

Engine may enumerate and reason about the object through `IResource`, while an SDDC Flex Backend can consume the same graph node as `VirtualMachineResource` because it references `SddcFlex.Abstractions`.

The narrow Engine interface does not imply a narrow or lossy graph. The full concrete resource instance remains available to the Backend with all domain properties produced during Semantic Analysis.

## Lossless Infrastructure IR

The Infrastructure IR SHALL preserve all resolved information required by a conformant Backend to lower the resource into a supported Target.

A Backend SHALL NOT need to return to raw or Parsed Intent merely to recover a property that was available before Semantic Analysis.

If Parsed Intent provides CPU, memory, storage profile, and a network reference, and the Semantic Model accepts those values, the resulting typed resource and graph SHALL preserve the corresponding domain state and resolved graph semantics required for Backend consumption.

## Resource references and graph connectivity

Domain resource objects MAY contain typed resource references, but graph connectivity and resolved edges remain Engine-owned concerns.

The preferred minimum reference shape is defined in ADR-009 and is conceptually:

```csharp
public readonly record struct ResourceReference<TResource>(
    ResourceIdentity Identity)
    where TResource : IResource;
```

The Integration creates the reference because it understands domain meaning. The reference identifies the expected target resource type and identity, but it does not hold a live target object, define relationship meaning, or automatically imply a dependency.

Engine resolves the reference against the complete registered resource set and validates target existence/type. A Backend may later ask the Resource Graph to resolve the typed reference to the concrete target resource through Engine-owned graph APIs.

## Graph-owned relationships and dependencies

Semantic relationships and dependency edges SHOULD be represented by the Resource Graph rather than being required members of each resource object.

An illustrative graph contract may resemble:

```csharp
public interface IResourceGraph
{
    IReadOnlyCollection<IResource> Resources { get; }
    IReadOnlyCollection<ResourceRelationship> Relationships { get; }
    IReadOnlyCollection<ResourceDependency> Dependencies { get; }
}
```

This keeps graph state, cycle detection, dependency ordering, traversal, and reference resolution under Engine ownership while allowing resource classes to focus on domain state.

ADR-009 defines the semantic distinction:

- a **relationship** expresses domain meaning, such as a virtual machine using a network or disk;
- a **dependency** expresses prerequisite or ordering behavior, such as a virtual disk needing to exist before a virtual machine can use it.

A relationship may produce a dependency according to Integration semantics, but the concepts are not interchangeable. A genuine prerequisite may also create a dependency without a semantic relationship. Integrations SHALL NOT invent semantic relationships or synthetic container resources merely to encode ordering.

`ResourceDependency` remains a structural graph fact. Domain-facing explanations and rule identities belong to separate provenance supplied by Integration semantics and carried by Engine as needed for diagnostics; they do not affect dependency equality or graph behavior.

## Domain Abstractions and Backend lowering

Domain Abstractions are the stable typed semantic input consumed by Backends.

```text
Domain Abstractions
        |
        v
      Backend
        |
        v
Target Abstractions
```

The Backend does not define domain resource semantics; those belong to the Integration and its Domain Abstractions. The Backend does not define Target serialization; that belongs to the Target. Its responsibility is translation between the two published contracts.

## Resource Graph ownership

The Adapter does not construct the canonical Resource Graph. It owns source translation only:

```text
YAML / JSON / API / other Source
        |
        v
      Adapter
        |
        v
   Parsed Intent
```

Engine constructs the canonical graph through the multi-phase Semantic Analysis lifecycle defined in ADR-010. Source declaration order SHALL NOT determine graph identity, relationship, or dependency semantics.

This permits different Source representations and declaration orders to resolve to equivalent infrastructure meaning and deterministic Resource Graphs.

## Backend consumption

A Backend is Integration-owned and understands the Integration's Domain Abstractions.

For example:

```text
SddcFlex.Backend.Terraform
    -> Engine.Abstractions
    -> SddcFlex.Abstractions
    -> Terraform.Target.Abstractions
```

A second Backend can consume the same resolved SDDC Flex graph types while mapping them into a different Target.

## Domain contract versioning

Domain Abstractions SHALL follow the same contract-generation principles defined by ADR-007.

A published Domain Abstractions generation is immutable in its required public shape and semantics. Package revisions within a generation MAY fix defects, documentation, tooling, metadata, or other concerns that do not invalidate already-conformant consumers.

Breaking changes require a new independently addressable contract generation. Backends choose which Domain Abstractions generation they consume. The existence of a newer generation does not by itself require a Backend to migrate.

Domain compatibility SHALL be explicit and testable.

## Physical extension model

The principal independently loadable extension types remain:

```text
Adapters
Integrations
Targets
```

A Semantic Model is an Integration responsibility and need not be a separate plugin assembly. A Backend is Integration-owned and may be packaged with the Integration or separately according to ownership and release needs. An Emitter is a Target responsibility and need not be an independently loaded plugin merely because it is an architectural concept.

## Guardrails

- Engine SHALL NOT reference Integration-specific resource types at compile time.
- Domain Abstractions SHALL depend only on stable Engine contracts and domain-neutral dependencies appropriate to the public contract.
- Backends MAY depend on Domain Abstractions and the Target Abstractions they support.
- Domain resource types SHALL NOT contain Target-specific representation concerns.
- Semantic Models define/apply resource semantics; Resource Graphs contain resolved resource instances and edges.
- Integrations create typed references; Engine resolves them.
- Integrations define relationship and prerequisite semantics; Engine constructs and analyzes resulting graph edges.
- Integration implementations SHALL NOT own an independent Resource Graph implementation.
- The Resource Graph SHALL preserve concrete Integration-owned typed resource instances rather than flattening them into generic property bags solely for Engine convenience.
- The Infrastructure IR SHALL preserve resolved information required by supported Backends; Backends SHALL NOT depend on raw Intent for information recovery.
- Source syntax and declaration order SHALL NOT leak into canonical graph semantics.
- Target concepts SHALL NOT leak into Domain Abstractions.
- Backends SHALL NOT redefine domain semantics owned by the Integration.
- Backends SHALL NOT own Target serialization or emission behavior owned by the Target.
- A contract abstraction SHOULD exist only when it protects an independently owned or independently evolving concern.

## Consequences

### Positive

- Strong typing is preserved for Integration and Backend developers.
- Engine remains independent of infrastructure-domain implementations.
- Multiple Backends can reuse the same resolved domain model.
- Domain contracts can evolve using explicit compatibility discipline.
- The Resource Graph can remain semantically rich without becoming a universal lowest-common-denominator cloud model.
- Typed identity-based resource references avoid embedding a cyclic object graph while preserving domain type safety.
- Source-order independence and graph ownership are explicit.

### Negative / risks

- Each Integration may introduce an additional public contract package and versioning responsibility.
- Engine plugin loading must preserve CLR type identity between Domain Abstractions consumed by Integrations and Backends.
- Poorly designed Domain Abstractions could become overly broad or change too frequently.
- Typed domain resources complicate serialization and persistence of the Resource Graph compared with a generic property bag.
- Supporting multiple Domain Abstractions generations in one process may require deliberate assembly isolation or compatibility adapters.
- Resource reference and graph-resolution APIs must be designed carefully so Backends can navigate relationships without coupling resources directly to one another.

## Open questions

- What is the exact common Engine resource contract beyond `Identity` and `Type`, if anything?
- What is the exact `IResourceGraph` contract and lookup/resolution API?
- What is the concrete `ResourceRelationship` contract and relationship-type representation?
- What common typed value system, if any, belongs in Engine Abstractions?
- What is the minimum public Semantic Model API that supports ADR-010's lifecycle without exposing unnecessary graph internals?
- How are typed Resource Graphs serialized into diagnostics, provenance, or Artifact Bundles?
- What constitutes the minimum conformance suite for a Domain Abstractions generation?
- How are multiple Domain Abstractions generations isolated and resolved by the in-process plugin loader?
- Should Backends normally be packaged with their Integration or independently when they have different release cadences?

## Related decisions

- ADR-007 defines contract evolution and compatibility.
- ADR-009 defines Resource Identity, typed references, relationships, dependencies, and dependency provenance boundaries.
- ADR-010 defines the multi-phase Semantic Model and Semantic Analysis lifecycle.