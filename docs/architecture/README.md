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
    |          - domain concepts
    |          - resource types
    |          - identities
    |          - properties
    |          - constraints
    |          - defaults
    |          - relationships
    |          - semantic rules
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
          Target IR
              |
              v
           Emitter
              |
              v
       Artifact Bundle
```

## Why start from Intent

The architecture should describe infrastructure independently of the mechanism eventually used to deploy it.

Intent declares what infrastructure is desired. It does not need to carry the complete definition of what every infrastructure concept means. That domain knowledge belongs to a Semantic Model supplied by an Infrastructure Integration and is applied during Semantic Analysis.

Terraform is an important Target, but it does not define the Engine's resource model. Distinct deployment technologies such as ARM, Bicep, CloudFormation, Terraform, OpenTofu, Ansible, and future Targets may coexist without modifying Engine Core.

This gives the architecture a deliberate separation:

- **Intent** declares desired infrastructure.
- **Adapters** translate external source representations into parsed Intent.
- **Integrations** own infrastructure-domain knowledge and domain-to-Target mappings.
- **Domain Abstractions** define stable, strongly typed resource contracts shared by an Integration and its Backends.
- **Semantic Models** define the meaning and rules of infrastructure domains.
- **Semantic Analysis** applies domain meaning to Intent and resolves it.
- **Infrastructure IR / Resource Graph** contains resolved Integration-owned typed resource instances plus Engine-owned graph relationships and dependency edges.
- **Backends** lower resolved domain resources into one published Target contract.
- **Targets** own deployment-technology models, validation, conformance, and emission.
- **Emitters** are Target responsibilities that turn Target models into physical artifacts.

## Infrastructure Integrations

An Infrastructure Integration is the extension and ownership boundary for an infrastructure domain.

Engine defines the Integration contract; the Integration author owns the implementation and lifecycle of the domain support. An Integration may build its Semantic Model from hand-written code, generated definitions, reflection, upstream schemas, source generators, metadata, or any other suitable mechanism. Engine does not prescribe that implementation.

The initial extension model is an in-process .NET plugin assembly implementing published Engine abstractions.

An Integration owns its Semantic Model, Domain Abstractions, and the Backends that map its domain into supported Targets. For example:

```text
SddcFlex.Abstractions
        ^
        |
SddcFlex.Integration
        |
        +-- Semantic Model
        +-- SddcFlex -> Terraform Backend
        +-- SddcFlex -> OpenTofu Backend
```

A change to the SDDC Flex domain, such as adding a new resource type, should require an Integration/domain-contract change rather than an Engine Core change when the existing generic contracts are sufficient.

## Semantic Models

A Semantic Model is the authoritative definition of an infrastructure domain's concepts, resource types, identities, properties, constraints, defaults, relationships, and semantic rules used to interpret and validate Intent.

It is deliberately broader than a collection of resource schemas. Resource definitions are part of a Semantic Model, but the model also defines the domain behavior necessary to determine whether Intent is meaningful and valid.

A Semantic Model SHALL remain independent of deployment Targets. For example, it may define that a virtual machine has a required relationship to a network, but it does not decide how that relationship becomes a Terraform address, Bicep expression, CloudFormation reference, or other Target-specific construct.

## Extension model

The principal independently loadable extension types are currently:

1. **Adapters** - external source representations to parsed Intent.
2. **Infrastructure Integrations** - independently owned infrastructure domains, including Semantic Models, Domain Abstractions, and Backends.
3. **Targets** - deployment-technology contracts, models, validation, conformance, and emission.

Semantic Models, Backends, and Emitters remain important architectural responsibilities, but they are not required to become independently loaded plugin types merely because they are separate concepts.

## Resource Graph

The Engine's canonical resolved representation is the Infrastructure IR represented by a deterministic Resource Graph.

The graph contains concrete Integration-owned strongly typed resources. Engine itself interacts with those resources through a deliberately small common contract, while Integration Backends can consume the full domain types through the Integration's Domain Abstractions.

Engine owns graph structure and resolution semantics: stable graph identity, resolved relationships, dependency edges, traversal, cycle detection, deterministic ordering, and reference resolution. Integrations own the typed resource payloads inside the graph.

The graph is lossless for Backend needs: a Backend must not return to raw or parsed Intent merely to recover accepted input properties that should have survived semantic analysis.

The graph is Target-independent. It does not contain Terraform resource blocks, Bicep syntax, CloudFormation templates, file paths, or other Target representation concerns.

See [ADR-008 - Domain Abstractions and Typed Resource Graphs](ADR-008-domain-abstractions-and-typed-resource-graphs.md).

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

A monorepo is the current preference for the Engine itself because the core compiler components are expected to evolve together.

Integrations and Targets may be developed outside the Engine repository. Repository topology should follow real ownership and release boundaries rather than mirroring every architecture interface one-for-one.

## Open questions

The following remain deliberately unresolved:

- What is the exact minimum `IResource` contract beyond `Identity` and `Type`, if anything?
- What is the exact `IResourceGraph` lookup, resolution, traversal, and ordering API?
- What is the shape of a typed `ResourceReference<TResource>`?
- What are the exact `IInfrastructureIntegration` and `ISemanticModel` contracts?
- How does Engine invoke Integration semantic behavior without depending on Integration implementation types?
- What is the exact generic Backend contract?
- What is the minimum mandatory content of Engine, Domain, and Target conformance suites?
- How are plugin assemblies discovered, loaded, isolated, and trusted, including multiple contract generations in one process?
- How does Engine discover plugin and contract metadata without eagerly activating plugins?
- How do Targets such as Ansible fit if IR-plus-Emitter is not their natural internal architecture?
- How are typed Resource Graphs serialized for diagnostics, provenance, or Artifact Bundles?
- What belongs in the Artifact Bundle contract?
- What is the first vertical slice that proves the architecture?

## Proposed ADRs

- [ADR-001 - Infrastructure Intent Compiler](ADR-001-infrastructure-intent-compiler.md)
- [ADR-002 - Infrastructure IR and Resource Graph](ADR-002-infrastructure-ir-resource-graph.md)
- [ADR-003 - Pluggable Backends and Emitters](ADR-003-pluggable-backends-and-emitters.md)
- [ADR-004 - Cloud-Native Operating Principles](ADR-004-cloud-native-operating-principles.md)
- [ADR-005 - Infrastructure Integrations and Extension Ownership](ADR-005-infrastructure-integrations-and-extension-ownership.md)
- [ADR-006 - Target Contracts and Backend Dependency Model](ADR-006-target-contracts-and-backend-dependencies.md)
- [ADR-007 - Contract Evolution and Compatibility](ADR-007-contract-evolution-and-compatibility.md)
- [ADR-008 - Domain Abstractions and Typed Resource Graphs](ADR-008-domain-abstractions-and-typed-resource-graphs.md)

See also the [working glossary](glossary.md).