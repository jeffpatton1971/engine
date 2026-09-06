# Architecture Exploration

> **Decision Status: Exploration**

This directory records the current architectural exploration for Engine. Terminology and proposed decisions may evolve as the design is pressure-tested against real infrastructure domains and deployment Targets.

## Working premise

Engine is an infrastructure intent compiler.

```text
Source Intent
    |
    v
Adapter
    |
    v
Parsed Intent --------+
    |                  |
    |          Infrastructure Integration
    |                  |
    |          Semantic Model
    |                  |
    +---------+--------+
              |
              v
      Semantic Analysis
              |
              v
      Infrastructure IR
       (Resource Graph)
              |
              v
  Integration Backend
              |
              v
       Target Contract
              |
              v
         Target Model
              |
              v
 Target Validation + Emission
              |
              v
       Artifact Bundle
```

Semantic Analysis is multi-phase. Integrations materialize typed domain nodes, canonical identities/references, and domain diagnostics. Engine registers identities, resolves references, constructs graph facts, validates graph integrity, and establishes deterministic managed provisioning order. Integrations define the domain semantics that produce relationships and dependencies; they do not own an independent graph implementation.

## Architectural ownership

- **Intent** declares desired infrastructure.
- **Adapters** translate Source representations into Parsed Intent.
- **Integrations** own infrastructure-domain semantics, Domain Abstractions, materialization/domain validation, canonical identity/scoping, and domain-to-Target Backends.
- **Domain Abstractions** define stable typed semantic contracts shared by an Integration and its Backends.
- **Semantic Analysis** combines Integration semantics with Engine graph mechanics.
- **Infrastructure IR / Resource Graph** contains resolved Integration-owned managed/existing typed nodes plus Engine-owned graph facts.
- **Backends** lower valid domain graphs into Target contracts and determine whether domain semantics are representable by a supported Target generation.
- **Targets** own deployment-technology models, Target validation, conformance, and emission.

## Infrastructure Integrations

An Infrastructure Integration is the extension/ownership boundary for one infrastructure domain.

Engine defines Integration contracts; Integration authors own domain implementation and lifecycle. Semantic Models may be hand-written, generated, schema-driven, reflection-based, metadata-driven, or hybrid.

The initial extension model is an in-process .NET plugin assembly. Adding a new domain resource type should not require an Engine Core change when existing generic contracts remain sufficient.

## Semantic analysis lifecycle

```text
1. Materialization                 Integration
2. Identity Registration           Engine
3. Reference Resolution            Engine
4. Relationship / Dependency       Integration
   Semantic Analysis
5. Graph Construction / Validation Engine
```

Source declaration order has no semantic significance.

Validation ownership is layered:

```text
Integration validation -> domain correctness
Engine validation      -> identity/resolution/graph correctness
Backend validation     -> Target representability
Target validation      -> Target-model correctness
```

Independent user-correctable diagnostics should be aggregated when safe; unrecoverable structural failures may terminate affected processing early.

See [ADR-010](ADR-010-semantic-model-and-semantic-analysis-lifecycle.md).

## Resource Graph and lifecycle

The Resource Graph is common in structure, not in infrastructure semantics.

A working Engine-level participant contract contains canonical identity/type. Domain Abstractions provide semantic contracts such as `ISubnet` and `IVirtualMachine`. Lifecycle is orthogonal:

```text
ISubnet + IManagedResource
ISubnet + IExistingResource
```

Both managed and existing nodes participate in identity, references, relationships, semantic dependencies, validation, and Backend lowering.

Managed nodes are scheduled by the current compilation; existing nodes are asserted to already exist and are treated as satisfied prerequisites for provisioning.

Typed references target the domain contract:

```csharp
ResourceReference<ISubnet>
```

rather than a lifecycle-specific implementation.

See [ADR-008](ADR-008-domain-abstractions-and-typed-resource-graphs.md).

## Identity, relationships, and dependencies

Graph identity is:

```text
IntegrationId + ResourceType + ResourceKey
```

Integration owns canonical ResourceKey construction and infrastructure-domain scope interpretation. Engine enforces uniqueness. Lifecycle, Adapter identity, and Target addresses do not participate.

If two scoped resources share a logical name, Integration must encode enough scope in the canonical key to distinguish them. Engine does not learn Azure Resource Groups/VNets, GCP projects, Kubernetes namespaces, or equivalent domain scoping rules.

Relationships express domain meaning. Dependencies express prerequisites/order. A relationship may imply a dependency, but dependencies may also exist independently when justified by real domain semantics.

Integration determines whether a dependency exists and its direction. Engine does not infer dependency direction from managed/existing lifecycle.

Semantic dependency traversal may include managed and existing nodes. Managed provisioning order is a projection in which existing prerequisites are already satisfied.

Dependency reason/provenance remains separate from structural dependency identity.

See [ADR-009](ADR-009-resource-identity-references-and-graph-edges.md).

## Semantically lossless IR

Infrastructure IR must preserve every accepted semantic fact required by conformant Backends.

Once Semantic Analysis succeeds, Parsed Intent is finished as a source of domain meaning. Backends must not reopen YAML, JSON, or Parsed Intent to recover properties, relationships, lifecycle, scope, or dependencies.

This is **semantic losslessness**, not source-representation preservation. Source comments, formatting, aliases, and ordering need not survive except for diagnostics/provenance.

Backend conformance should be executable without Adapter or Parsed Intent dependencies available.

## Compilation Context

Some information belongs to one compilation rather than one resource. Defined `CompilationContext` may carry such values downstream.

Possible examples include subscription/account selection, location defaults, credential/profile selection, naming context, workflow metadata, or Target selection.

Context is scoped to **one Intent/compilation**. It is not global Engine state and must not contain raw Parsed Intent as an escape hatch around the IR boundary.

## Targets and Backend compatibility

A Target represents one distinct deployment technology through published versioned Target Abstractions. Terraform and OpenTofu remain distinct Targets even when implementation reuse is possible.

A Backend references Engine contracts, its Integration's Domain Abstractions, and the Target Abstractions generation it supports. It does not reference the concrete Target implementation.

```text
Azure.Backend.Terraform
    -> Engine.Abstractions
    -> Azure.Abstractions
    -> Terraform.Target.Abstractions
```

A valid domain graph does not imply every Target can represent it. Backend owns representability diagnostics because it knows both Domain and Target contracts. Target then validates the Target model actually produced.

Every published Target SHALL provide a versioned Backend conformance suite; compatibility is demonstrated rather than inferred from package versions.

See [ADR-005](ADR-005-infrastructure-integrations-and-extension-ownership.md), [ADR-006](ADR-006-target-contracts-and-backend-dependencies.md), and [ADR-007](ADR-007-contract-evolution-and-compatibility.md).

## Artifact contract

Compilation produces an Artifact Bundle rather than loose files. Its exact schema remains open but should eventually cover artifacts, diagnostics, component/contract versions, source digest, and useful provenance/graph information.

## Extension model

Principal independently loadable extension types remain:

1. Adapters
2. Infrastructure Integrations
3. Targets

Semantic Models, Backends, and Emitters are architectural responsibilities but need not become independently loaded plugins unless ownership/release needs justify it.

## Cloud-native posture

Cloud-native describes operating principles rather than mandatory technologies: stateless execution, declarative contracts, immutable/versioned outputs, portable execution, structured diagnostics, observability, API/CLI parity, and independently distributable extensions where useful.

## Repository direction

A monorepo remains the current preference for Engine itself. Integrations and Targets may live elsewhere according to ownership/release boundaries.

## Open questions

- What exact common `IResourceNode` contract is required beyond `Identity` and `Type`?
- What is the exact Resource Graph lookup/traversal API, including semantic dependency and managed provisioning views?
- How are domain contracts such as `ISubnet` associated with canonical ResourceType metadata?
- What is the concrete ResourceRelationship representation?
- What are the exact Integration/Semantic Model public APIs?
- What is the exact generic Backend runtime invocation bridge?
- What is the minimum defined per-compilation `CompilationContext` contract?
- How does a Backend declare/query Target representability before or during lowering?
- What is the common diagnostic/provenance contract across Integration, Engine, Backend, and Target?
- How are plugin assemblies discovered, loaded, isolated, and trusted across contract generations?
- How do Targets such as Ansible fit if IR-plus-Emitter is unnatural for them?
- How are typed graphs serialized for diagnostics/provenance/Artifact Bundles?
- What belongs in the Artifact Bundle contract?

The Azure VM pressure-test series under [`scenarios/`](scenarios/) has now validated the major semantic-analysis boundaries. The next design work should use those findings to shape minimal public contracts rather than adding abstraction without a concrete pressure case.

## Proposed ADRs

- [ADR-001 - Infrastructure Intent Compiler](ADR-001-infrastructure-intent-compiler.md)
- [ADR-002 - Infrastructure IR and Resource Graph](ADR-002-infrastructure-ir-resource-graph.md)
- [ADR-003 - Pluggable Backends and Emitters](ADR-003-pluggable-backends-and-emitters.md)
- [ADR-004 - Cloud-Native Operating Principles](ADR-004-cloud-native-operating-principles.md)
- [ADR-005 - Infrastructure Integrations and Extension Ownership](ADR-005-infrastructure-integrations-and-extension-ownership.md)
- [ADR-006 - Target Contracts and Backend Dependency Model](ADR-006-target-contracts-and-backend-dependencies.md)
- [ADR-007 - Contract Evolution and Compatibility](ADR-007-contract-evolution-and-compatibility.md)
- [ADR-008 - Domain Abstractions and Typed Resource Graphs](ADR-008-domain-abstractions-and-typed-resource-graphs.md)
- [ADR-009 - Resource Identity, References, and Graph Edge Semantics](ADR-009-resource-identity-references-and-graph-edges.md)
- [ADR-010 - Semantic Model and Semantic Analysis Lifecycle](ADR-010-semantic-model-and-semantic-analysis-lifecycle.md)

See also the [working glossary](glossary.md).