# ADR-002: Infrastructure IR and Resource Graph

- **Status:** Proposed
- **Date:** 2026-09-04
- **Decision Type:** Architecture exploration

## Context

An infrastructure intent compiler needs a canonical representation after source parsing/semantic analysis and before Target-specific lowering.

Using source documents as that representation would couple semantics to external syntax. Using Terraform, Bicep, CloudFormation, or another Target model would couple Engine to deployment technology.

Infrastructure naturally contains typed semantic participants, stable identities, references, relationships, and prerequisites. These form a graph rather than merely a hierarchical document.

## Decision

Introduce **Infrastructure IR** as the canonical resolved representation of infrastructure Intent, represented primarily as a deterministic **Resource Graph**.

```text
Parsed Intent + Integration Semantic Model
                |
                v
        Semantic Analysis
                |
                v
Infrastructure IR / Resource Graph
    Integration-owned typed domain nodes
    stable canonical identities
    resolved typed references
    semantic relationships
    prerequisite dependencies
    managed provisioning view
    diagnostics/provenance as appropriate
```

Infrastructure IR is Target-independent.

## Domain nodes and lifecycle

The graph SHALL retain concrete Integration-owned typed semantic state while Engine operates through a deliberately small common graph-participation contract.

The current illustrative shape is:

```csharp
public interface IResourceNode
{
    ResourceIdentity Identity { get; }
    ResourceType Type { get; }
}
```

Domain type and lifecycle are orthogonal:

```text
ISubnet + IManagedResource
ISubnet + IExistingResource
```

Both managed and existing nodes participate in identity, references, relationships, semantic dependencies, validation, and Backend lowering.

Managed nodes represent infrastructure the current compilation intends to manage. Existing nodes represent infrastructure asserted to already exist and are not scheduled for creation.

The exact API names remain illustrative.

## Typed domain state

Engine SHALL NOT flatten Integration resources into generic property bags solely for its own convenience.

A Backend referencing the Integration's Domain Abstractions must be able to consume the full semantic domain state retained in the graph.

Typed references target domain semantic contracts rather than lifecycle-specific implementation classes:

```text
ResourceReference<ISubnet>
```

may resolve to either a managed or existing subnet.

See ADR-008 and ADR-009.

## Semantically lossless IR

Infrastructure IR SHALL preserve every accepted semantic fact required by conformant Backends.

A Backend SHALL NOT return to raw or Parsed Intent to recover domain information that should have survived Semantic Analysis.

This requirement is **semantic losslessness**, not source-representation preservation. Source formatting, ordering, comments, aliases, and Adapter-specific syntax need not survive except where required for diagnostics/provenance.

Defined per-compilation context may carry values that belong to the compilation rather than a resource. That context is part of the downstream semantic handoff but SHALL NOT expose raw Parsed Intent as an escape hatch.

## Graph ownership

Engine owns graph structure/resolution mechanics. Integrations own domain semantics and typed payloads.

Conceptually:

```csharp
public interface IResourceGraph
{
    IReadOnlyCollection<IResourceNode> Nodes { get; }
    IReadOnlyCollection<ResourceRelationship> Relationships { get; }
    IReadOnlyCollection<ResourceDependency> Dependencies { get; }
}
```

The exact API is open.

Relationships express domain meaning. Dependencies express prerequisites/order. Integrations determine which facts exist and dependency direction; Engine constructs/deduplicates/resolves/analyzes those structural facts.

Engine SHALL NOT infer dependency validity from managed/existing lifecycle.

## Semantic and provisioning dependency views

Managed and existing nodes participate in semantic dependency traversal.

Provisioning order is a managed-resource projection of that graph. Existing prerequisites are treated as already satisfied and remain visible for semantic traversal, diagnostics, and Backend lowering without being scheduled.

Managed provisioning cycles are Engine graph failures. Additional domain-specific semantic-cycle restrictions belong to Integration semantics.

## Identity

Resource identity is independent of Target representation and lifecycle:

```text
IntegrationId + ResourceType + ResourceKey
```

Integration owns ResourceKey canonicalization/scoping. Engine enforces uniqueness.

Equivalent semantic infrastructure expressed through different Adapters should resolve to equivalent identities and graph semantics.

## Semantic Models

A Semantic Model is the Integration-owned definition/application of infrastructure-domain concepts, identities, properties, constraints, defaults, relationships, prerequisite rules, and validation semantics.

Semantic Models remain Target-independent and may be implemented through code, generated definitions, schemas, metadata, or hybrid techniques.

Engine defines required semantic operations/lifecycle rather than a universal infrastructure schema.

## Compiler boundary

```text
Intent
"What infrastructure is desired?"
        |
        v
Semantic Model + Semantic Analysis
"What does it mean in this domain?"
        |
        v
Infrastructure IR / Resource Graph
"What semantic state has been resolved?"
        |
        v
Backend
"Can and how should it be represented by this Target contract?"
        |
        v
Target Model
```

Backend owns Target representability; Target owns Target-model correctness and emission.

## Guardrails

- Infrastructure IR is common in structure, not necessarily common in resource semantics.
- Engine SHALL NOT define a universal cloud resource taxonomy.
- Engine SHALL NOT depend on Integration-specific resource types at compile time.
- The graph SHALL preserve Integration-owned typed semantic state.
- Managed and existing nodes SHALL share the same identity/reference/relationship model.
- Lifecycle SHALL NOT participate in ResourceIdentity.
- Infrastructure IR SHALL be semantically lossless for supported Backend needs.
- Parsed Intent SHALL NOT be a downstream recovery mechanism after successful Semantic Analysis.
- Semantic relationships and dependencies SHALL be explicit graph information.
- Semantic Models/Domain Abstractions SHALL NOT contain Target-specific lowering/emission behavior.
- Per-compilation context SHALL remain scoped to one Intent/compilation and SHALL NOT become global Engine state.

## Consequences

### Positive

- Engine has a Target-independent canonical representation.
- Brownfield and greenfield infrastructure coexist in one graph.
- Strong domain typing survives into Backend lowering.
- Source-order and source-format coupling are removed from semantic graph behavior.
- Multiple Backends can consume the same resolved domain semantics.
- Explainability/visualization can operate independently of artifact generation.

### Risks

- Typed graph serialization/persistence is more complex than generic property bags.
- Plugin loading must preserve CLR type identity across shared Domain Abstractions.
- Graph navigation and domain-contract metadata APIs require careful design.
- Existing-resource enrichment may eventually be required without weakening the offline semantic boundary.

## Alternatives considered

### Source document as canonical model
Rejected because source syntax and infrastructure meaning have different responsibilities.

### Target representation as canonical model
Rejected because it moves Target mechanics into Engine's semantic center.

### Universal infrastructure resource model
Rejected as the default because it risks erasing domain-specific semantics.

### Generic property bag graph
Rejected as the primary model because it discards compile-time domain typing and moves structural failures to Backend runtime.

## Open questions

- What exact common IResourceNode contract is required beyond Identity/Type?
- What is the Resource Graph lookup/traversal API?
- How are domain contracts associated with canonical ResourceType metadata?
- How are partial graphs exposed for diagnostics after failed compilation phases?
- How are cross-Integration relationships represented, if supported?
- What provenance belongs in graph versus Artifact Bundle?
- How is typed IR serialized without weakening runtime type safety?
- What exact contract defines per-compilation context?