# ADR-008: Domain Abstractions and Typed Resource Graphs

- **Status:** Proposed
- **Date:** 2026-09-05
- **Decision Type:** Architecture exploration

## Context

Engine must construct a deterministic Infrastructure IR from Parsed Intent and an Integration-owned Semantic Model. Integration Backends must consume that IR with enough domain information and type safety to map infrastructure semantics into a Target contract.

A generic property bag would weaken Backend type safety; making Engine compile directly against every Integration's concrete models would couple Engine releases to domain releases. The architecture therefore needs stable Integration-owned Domain Abstractions while Engine operates only through common contracts.

The Azure pressure tests further established that the graph must support mixed greenfield and brownfield compilations without duplicating domain semantics or requiring existing infrastructure to be fully redescribed.

## Decision

Each Infrastructure Integration SHOULD publish separately consumable **Domain Abstractions** containing its stable semantic contracts and typed resource models.

Engine SHALL NOT take compile-time dependencies on Integration-specific Domain Abstractions. Backends MAY reference their Integration's Domain Abstractions directly.

The Resource Graph SHALL preserve Integration-owned typed semantic state rather than flattening it into a generic property bag.

### Domain type and lifecycle are orthogonal

A domain contract expresses **what the infrastructure thing is**. Lifecycle contracts express whether that thing is managed by the current compilation or already exists.

Conceptually:

```csharp
public interface IResourceNode
{
    ResourceIdentity Identity { get; }
    ResourceType Type { get; }
}

public interface IManagedResource : IResourceNode
{
}

public interface IExistingResource : IResourceNode
{
}
```

Integration Domain Abstractions define semantic contracts such as:

```csharp
public interface ISubnet : IResourceNode
{
}

public interface IVirtualMachine : IResourceNode
{
}
```

A managed and an existing subnet may therefore both satisfy `ISubnet` while differing in lifecycle:

```text
SubnetResource
    ISubnet + IManagedResource

ExistingSubnet
    ISubnet + IExistingResource
```

These APIs are illustrative; the semantic split is the decision.

The working architectural term **resource participant** refers generally to any managed or existing node that participates in graph semantics. It need not become a separate public CLR type if `IResourceNode` plus lifecycle markers provides the required contract.

## Managed and existing participants

Both managed and existing participants SHALL participate in:

```text
identity
references
relationships
dependency resolution
semantic validation
Backend lowering
```

A managed resource is infrastructure the current compilation intends to manage. An existing resource is infrastructure asserted to already exist and SHALL NOT be scheduled for creation merely because it participates in the graph.

Existing resources SHOULD carry only the minimum domain information required for identity, semantic validation, and supported Backend lowering. The architecture SHALL NOT require a complete managed-resource description merely to reference brownfield infrastructure.

A mixed graph is therefore normal:

```text
Existing subnet
    -> Managed NIC
    -> Managed VM
```

No compilation-wide greenfield/brownfield switch is required for graph semantics.

## Typed references target domain contracts

Typed references SHALL target a domain semantic contract rather than a lifecycle-specific implementation.

Conceptually:

```csharp
ResourceReference<ISubnet>
```

may resolve to either:

```text
ISubnet + IManagedResource
```

or:

```text
ISubnet + IExistingResource
```

The lifecycle of the target does not change the semantic meaning of the reference.

This supersedes earlier illustrative forms such as `ResourceReference<SubnetResource>` that accidentally coupled reference type safety to managed-resource implementation classes.

The Integration creates typed references because it understands domain meaning. Engine resolves the identity and validates that the resolved node is compatible with the expected domain contract and canonical `ResourceType` without becoming domain-aware.

The CLR/domain-type mapping may follow a registry pattern in which Integration-owned type metadata associates stable `IntegrationId + ResourceType` identities with domain contracts. The exact API remains open.

## Ownership

| Component | Responsibility |
| --- | --- |
| Adapter | Translate Source into Parsed Intent. |
| Domain Abstractions | Define stable domain contracts, managed resource types, existing-resource shapes, and typed values shared with Backends. |
| Integration / Semantic Model | Materialize semantic nodes, construct canonical identities/references, perform domain-local validation, and define relationship/prerequisite semantics. |
| Engine | Register identities, resolve references, construct graph edges, validate graph integrity, expose graph views, and produce deterministic ordering. |
| Backend | Lower resolved domain semantics into one Target contract and validate whether those semantics are representable by that Target generation. |
| Target | Validate the resulting Target model and emit Target-native artifacts. |

The governing distinction is:

> Integration supplies domain meaning; Engine supplies graph mechanics; Backend determines Target representability; Target validates and emits the Target representation.

## Semantic Model and Resource Graph

The Semantic Model and Resource Graph are related but distinct.

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

Source declaration order SHALL NOT determine identity, reference resolution, relationship semantics, or dependencies.

## Resource Graph structure

The Resource Graph SHALL expose a common Engine-owned structural model while retaining concrete Integration-owned domain state.

Conceptually:

```csharp
public interface IResourceGraph
{
    IReadOnlyCollection<IResourceNode> Nodes { get; }
    IReadOnlyCollection<ResourceRelationship> Relationships { get; }
    IReadOnlyCollection<ResourceDependency> Dependencies { get; }
}
```

The exact API is not frozen.

Engine may reason through `IResourceNode`, while a Backend can consume the resolved node through its Integration's domain contract such as `IVirtualMachine`, `ISubnet`, or `INetworkInterface`.

## Semantic-lossless Infrastructure IR

The Infrastructure IR SHALL be **semantically lossless** for conformant Backends.

Every accepted semantic fact required by a supported Backend must survive into:

- Domain Abstraction state;
- resolved identities/references;
- relationships/dependencies; or
- defined per-compilation context.

A Backend SHALL NOT return to raw or Parsed Intent to recover accepted semantic information.

Semantic-lossless does not mean source-representation-lossless. YAML ordering, comments, aliases, formatting, and Adapter-specific syntax need not survive except where required for diagnostics/provenance.

If downstream lowering reveals that an accepted domain property was lost, the architectural correction is to preserve that property in Domain Abstractions or defined compilation context—not to expose raw Intent to the Backend.

## Compilation Context

A Backend MAY receive defined `CompilationContext` in addition to the Resource Graph for information that belongs to the current compilation rather than an individual resource.

Examples may include account/subscription selection, region defaults, naming context, credential/profile selection, workflow metadata, or Target selection.

`CompilationContext` SHALL be scoped to one Intent/compilation and SHALL NOT be treated as global Engine state.

It SHALL NOT contain raw Parsed Intent as an escape hatch around the semantic IR boundary.

## Relationships, dependencies, and lifecycle

Relationships and dependencies remain graph-owned resolved facts as defined by ADR-009.

Lifecycle does not determine whether a dependency exists or its direction; the Integration's domain semantics do.

Existing and managed nodes may both appear in semantic dependency traversal. Provisioning order, however, is a managed-resource projection of the graph: existing prerequisites are treated as already satisfied rather than scheduled for creation.

This preserves one semantic dependency model without inventing special brownfield edge types.

## Backend lowering

A Backend bridges:

```text
Domain Abstractions
        |
        v
      Backend
        |
        v
Target Abstractions
```

The same resolved domain graph may be consumed by multiple Backends, such as Azure-to-Terraform and Azure-to-Bicep.

Managed and existing implementations of the same domain contract remain semantically compatible while allowing the Backend to choose different Target representations.

Target-specific mechanics such as Terraform data/reference constructs, Bicep `existing` declarations, ARM resource IDs, or equivalent concepts SHALL NOT leak into Domain Abstractions or Engine graph semantics.

## Backend conformance

Backend conformance SHOULD be executable using only:

```text
Domain Abstractions
Resource Graph
defined Compilation Context
Target Abstractions
```

Adapter and Parsed Intent dependencies should not be available to the conformance fixture. This makes the semantic IR boundary structurally testable.

## Domain contract versioning

Domain Abstractions SHALL follow ADR-007 contract-generation principles. Breaking required shape/semantic changes require a new independently addressable generation; non-breaking package revisions may occur within a generation.

Backend compatibility is demonstrated through conformance rather than inferred from package major versions alone.

## Guardrails

- Engine SHALL NOT reference Integration-specific resource types at compile time.
- Domain Abstractions SHALL NOT contain Target-specific representation concerns.
- Domain semantic contracts and lifecycle contracts SHALL remain orthogonal.
- Typed references SHALL target domain contracts rather than managed-resource implementation classes.
- Managed and existing nodes SHALL use the same identity and relationship model.
- Existing infrastructure SHALL NOT require a full managed-resource description unless semantic requirements genuinely demand those properties.
- Integrations create typed references; Engine resolves them.
- Integrations define relationship/dependency semantics; Engine constructs and analyzes resulting graph facts.
- Engine SHALL NOT infer dependency direction from managed/existing lifecycle type.
- Provisioning order SHALL be derived from managed resources while existing prerequisites are treated as already satisfied.
- The Resource Graph SHALL preserve Integration-owned semantic state rather than flattening it for Engine convenience.
- Backends SHALL NOT depend on raw or Parsed Intent for information recovery.
- `CompilationContext` SHALL be per Intent/compilation, defined, and non-global.
- Backends validate Target representability; Targets validate Target-model correctness.

## Consequences

### Positive

- Strong typing is preserved without making Engine domain-aware.
- Greenfield and brownfield resources coexist in one semantic graph.
- Existing resources can remain lightweight.
- Typed references do not leak lifecycle assumptions.
- Multiple Backends can reuse one semantically complete domain graph.
- Per-compilation context avoids singleton/global compilation state.
- The IR boundary can be enforced through Backend conformance tests.

### Risks

- Domain contracts may require separate existing implementations where identity/type alone is insufficient.
- The CLR domain-contract to `ResourceType` registry requires deliberate design.
- Multiple Domain Abstractions generations still create assembly/type-identity concerns.
- Existing-resource enrichment may eventually be needed for semantic rules or Target lowering; that mechanism remains separate from the initial graph contract.

## Open questions

- What exact common base contract should managed and existing nodes share beyond `Identity` and `Type`?
- What is the exact `IResourceGraph` lookup/traversal API?
- How is a domain interface such as `ISubnet` deterministically associated with canonical `ResourceType` metadata?
- What is the minimum existing-resource shape when identity/type alone is insufficient?
- How are graph nodes and typed domain state serialized into diagnostics/provenance/Artifact Bundles?
- What is the minimum Domain Abstractions conformance suite?
- How are multiple Domain Abstractions generations isolated in the plugin loader?

## Related decisions

- ADR-007 defines contract evolution and compatibility.
- ADR-009 defines Resource Identity, typed references, relationships, dependencies, and graph semantics.
- ADR-010 defines the Semantic Analysis lifecycle and validation ownership.