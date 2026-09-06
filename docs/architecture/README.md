# Architecture Exploration

> **Decision Status: Exploration**

This directory records the current architectural exploration for Engine. It is intentionally a design workspace: terminology and proposed decisions should be expected to evolve as the model is tested against real infrastructure domains and deployment Targets.

## Working premise

Engine is an infrastructure intent compiler.

It accepts declarative infrastructure Intent, interprets that Intent using infrastructure Semantic Models supplied by independently owned Integrations, constructs a deterministic Resource Graph containing Integration-owned typed resources, lowers that graph through an Integration Backend into a Target contract, and emits a versioned Artifact Bundle.

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

Semantic Analysis is multi-phase. Integrations materialize concrete typed resources, canonical identities and typed references and perform resource-local/domain-local validation. Engine registers identities, resolves references, constructs and validates graph edges, detects cycles, and establishes deterministic ordering. Integrations supply the domain semantics from which relationships and prerequisites are derived; they do not own an independent Resource Graph implementation.

## Why start from Intent

The architecture should describe infrastructure independently of the mechanism eventually used to deploy it.

Intent declares what infrastructure is desired. It does not need to carry the complete definition of what every infrastructure concept means. That domain knowledge belongs to a Semantic Model supplied by an Infrastructure Integration and is applied during Semantic Analysis.

Terraform is an important Target, but it does not define Engine's resource model. Distinct deployment technologies such as ARM, Bicep, CloudFormation, Terraform, OpenTofu, Ansible, and future Targets may coexist without modifying Engine Core.

This gives the architecture a deliberate separation:

- **Intent** declares desired infrastructure.
- **Adapters** translate external source representations into Parsed Intent.
- **Integrations** own infrastructure-domain knowledge, Domain Abstractions, resource materialization/local validation, and domain-to-Target mappings.
- **Domain Abstractions** define stable, strongly typed resource contracts shared by an Integration and its Backends.
- **Semantic Models** define domain meaning and the semantic operations Engine invokes during analysis.
- **Semantic Analysis** is the multi-phase Integration/Engine lifecycle that materializes resources, resolves identities/references, derives semantics, and produces the graph.
- **Infrastructure IR / Resource Graph** contains resolved Integration-owned typed resource instances plus Engine-owned relationship and dependency edges.
- **Backends** lower resolved domain resources into one published Target contract.
- **Targets** own deployment-technology models, target validation, conformance, and emission.
- **Emitters**, where useful, are Target responsibilities that turn Target models into physical artifacts.

## Infrastructure Integrations

An Infrastructure Integration is the extension and ownership boundary for an infrastructure domain.

Engine defines the Integration contract; the Integration author owns the implementation and lifecycle of the domain support. An Integration may build its Semantic Model from hand-written code, generated definitions, reflection, upstream schemas, source generators, metadata, or any other suitable mechanism. Engine does not prescribe that implementation.

The initial extension model is an in-process .NET plugin assembly implementing published Engine abstractions.

An Integration owns its Semantic Model, Domain Abstractions, and the Backends that map its domain into supported Targets. A change to an infrastructure domain, such as adding a new resource type, should require an Integration/domain-contract change rather than an Engine Core change when existing generic contracts remain sufficient.

## Semantic Model and analysis lifecycle

A Semantic Model is the versioned semantic contract an Integration exposes to Engine. Engine defines the semantic operations and lifecycle required, not a universal declarative schema or the Integration's internal modeling technique.

The current lifecycle is:

```text
1. Materialization                 Integration
2. Identity Registration           Engine
3. Reference Resolution            Engine
4. Semantic Relationship /         Integration
   Prerequisite Analysis
5. Graph Construction / Validation Engine
```

During materialization the Integration recognizes domain resource types, constructs concrete typed resources, creates canonical `ResourceIdentity` values and typed `ResourceReference<TResource>` values, and validates resource-local/domain-local semantics.

Engine then enforces identity uniqueness and resolves references against the complete materialized resource set. The Integration interprets the resolved resources/references to declare relationship and prerequisite semantics. Engine constructs the resulting graph edges, validates graph integrity, detects cycles, and establishes deterministic ordering.

Source declaration order has no semantic significance. Equivalent Intent expressed through different Adapters should resolve to equivalent semantic resources and graph structure.

Validation ownership is intentionally layered:

```text
Integration validation -> domain correctness
Engine validation      -> identity, resolution, graph correctness
Target validation      -> target-model correctness
```

See [ADR-010 - Semantic Model and Semantic Analysis Lifecycle](ADR-010-semantic-model-and-semantic-analysis-lifecycle.md).

## Extension model

The principal independently loadable extension types are currently:

1. **Adapters** - external source representations to Parsed Intent.
2. **Infrastructure Integrations** - independently owned infrastructure domains, including Semantic Models, Domain Abstractions, and Backends.
3. **Targets** - deployment-technology contracts, models, validation, conformance, and emission.

Semantic Models, Backends, and Emitters remain important architectural responsibilities, but they are not required to become independently loaded plugin types merely because they are separate concepts.

## Resource Graph

Engine's canonical resolved representation is the Infrastructure IR represented by a deterministic Resource Graph.

The graph contains concrete Integration-owned strongly typed resources. Engine interacts with those resources through a deliberately small common contract, while Integration Backends can consume the full domain types through the Integration's Domain Abstractions.

Graph identity is conceptually:

```text
IntegrationId + ResourceType + ResourceKey
```

The Integration owns canonical `ResourceKey` construction according to its domain's identity and scoping semantics; Engine enforces uniqueness of the complete identity.

Domain resources may contain identity-based typed `ResourceReference<TResource>` values. Integrations create those references because they understand domain meaning; Engine resolves them against the complete resource set.

Relationships and dependencies are distinct:

- a **relationship** expresses domain meaning;
- a **dependency** expresses prerequisite/order.

A relationship may derive a dependency, but dependencies may also exist without a semantic relationship when a genuine prerequisite exists. Integrations must not fabricate semantic relationships or synthetic container resources merely to make ordering work.

Dependency edges remain structural. Domain-facing reasons/rule identities belong to separate provenance supplied by Integration semantics and carried by Engine as needed for diagnostics; they do not participate in dependency identity or graph behavior.

The graph is lossless for Backend needs: a Backend must not return to raw or Parsed Intent merely to recover accepted information that should have survived Semantic Analysis.

See [ADR-008 - Domain Abstractions and Typed Resource Graphs](ADR-008-domain-abstractions-and-typed-resource-graphs.md) and [ADR-009 - Resource Identity, References, and Graph Edge Semantics](ADR-009-resource-identity-references-and-graph-edges.md).

## Contract evolution

Engine, Domain, and Target abstraction contracts follow explicit contract generations rather than inferring compatibility from implementation versions.

Published contract generations are immutable in their required public shape and semantics. Package revisions within a generation may contain non-breaking fixes and tooling changes, but breaking changes require a new independently addressable generation.

A component may advertise support for a contract generation only when it passes that generation's conformance suite. Newer implementations may continue to support older generations, but support is demonstrated rather than assumed.

Consumers choose when to adopt newer contract generations. The existence of V2 or V3 does not force a Backend using V1 to migrate while the supplying implementation continues to support and conform to V1.

See [ADR-007 - Contract Evolution and Compatibility](ADR-007-contract-evolution-and-compatibility.md).

## Targets and Backend compatibility

A Target represents one distinct deployment technology through a published, versioned Target Abstractions contract.

Shared syntax, ancestry, or implementation details do not imply a shared Target identity. Terraform and OpenTofu, for example, are separate Targets. If an Integration supports both, it explicitly supplies Backend support for both.

A Backend references Engine Abstractions, its Integration's Domain Abstractions, and the Target Abstractions for the specific Target it supports. It SHALL NOT reference the concrete Target implementation.

```text
SddcFlex.Backend.Terraform
    -> Engine.Abstractions
    -> SddcFlex.Abstractions
    -> Terraform.Target.Abstractions
```

Every published Target SHALL provide a versioned Backend conformance suite. Every Backend SHALL pass the applicable conformance suite before claiming compatibility with that Target contract generation.

Target implementation versions and Target contract generations are separate. Compatibility is based on explicit supported contract generations and conformance evidence rather than package-version ranges.

See [ADR-005 - Infrastructure Integrations and Extension Ownership](ADR-005-infrastructure-integrations-and-extension-ownership.md) and [ADR-006 - Target Contracts and Backend Dependency Model](ADR-006-target-contracts-and-backend-dependencies.md).

## Artifact contract

Compilation produces an Artifact Bundle rather than an unstructured collection of files.

The bundle should eventually define a stable contract for generated artifacts, diagnostics, Target information, contract-generation information, provenance, and potentially a serializable representation of the resolved graph used to produce the output.

The exact bundle schema remains open.

## Cloud-native posture

Cloud-native is being used here to describe operating principles rather than mandatory technologies.

The design should favor stateless execution, declarative contracts, immutable/versioned outputs, portable execution, structured diagnostics, observability, API/CLI parity, and independently distributable extensions.

The design does not currently require Kubernetes, microservices, service meshes, controllers, operators, or CRDs.

## Repository direction

A monorepo is the current preference for Engine itself because the core compiler components are expected to evolve together.

Integrations and Targets may be developed outside the Engine repository. Repository topology should follow real ownership and release boundaries rather than mirroring every architecture interface one-for-one.

## Open questions

The following remain deliberately unresolved:

- What is the exact minimum `IResource` contract beyond `Identity` and `Type`, if anything?
- What is the exact `IResourceGraph` lookup, resolution, traversal, and ordering API?
- What is the concrete `ResourceRelationship` contract and relationship-type representation?
- What are the exact `IInfrastructureIntegration` and Semantic Model public APIs?
- How does Engine present resolved resource/reference context to Integration semantic rules without surrendering graph ownership?
- What is the exact generic Backend contract and runtime invocation bridge?
- What is the separate provenance contract used for graph diagnostics and explainability?
- What is the minimum mandatory content of Engine, Domain, and Target conformance suites?
- How are plugin assemblies discovered, loaded, isolated, and trusted, including multiple contract generations in one process?
- How does Engine discover plugin and contract metadata without eagerly activating plugins?
- How do Targets such as Ansible fit if IR-plus-Emitter is not their natural internal architecture?
- How are typed Resource Graphs serialized for diagnostics, provenance, or Artifact Bundles?
- What belongs in the Artifact Bundle contract?

The immediate validation step is a small real infrastructure model that exercises the five-phase semantic lifecycle before the public Semantic Model API is frozen.

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