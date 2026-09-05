# Architecture Exploration

> **Decision Status: Exploration**

This directory records the current architectural exploration for Engine. It is intentionally a design workspace: terminology and proposed decisions should be expected to evolve as the model is tested against real infrastructure domains and deployment targets.

## Working premise

Engine is an infrastructure intent compiler.

It accepts declarative infrastructure intent, interprets that intent using infrastructure Semantic Models supplied by independently owned Integrations, constructs a deterministic intermediate representation, lowers that representation through an Integration Backend into a Target contract, and emits a versioned Artifact Bundle.

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

Terraform is an important target, but it should not define the Engine's resource model. Targets such as ARM, Bicep, CloudFormation, Terraform, OpenTofu, Ansible, and future deployment technologies should be able to coexist without modifying Engine Core.

This gives the architecture a deliberate separation:

- **Intent** declares desired infrastructure.
- **Integrations** own infrastructure-domain knowledge and domain-to-target mappings.
- **Semantic Models** define the meaning and rules of infrastructure domains.
- **Semantic Analysis** applies domain meaning to Intent and resolves it.
- **Infrastructure IR** represents resolved infrastructure meaning.
- **Backends** map an Integration's resolved semantics into a published Target contract.
- **Targets** own deployment-technology models, validation, and emission.
- **Emitters** turn target models into physical artifacts.

## Infrastructure Integrations

An Infrastructure Integration is the extension and ownership boundary for an infrastructure domain.

Engine defines the Integration contract; the Integration author owns the implementation and lifecycle of the domain support. An Integration may build its Semantic Model from hand-written code, generated definitions, reflection, upstream schemas, source generators, metadata, or any other suitable mechanism. Engine does not prescribe that implementation.

The initial extension model is an in-process .NET plugin assembly implementing published Engine abstractions.

An Integration owns its Semantic Model and the Backends that map its domain into supported Targets. For example:

```text
VCFA Integration
    |
    +-- VCFA Semantic Model
    +-- VCFA -> Terraform Backend
    +-- VCFA -> another supported Target Backend
```

A change to the VCFA domain, such as adding a new resource type, should require a VCFA Integration change rather than an Engine Core change when the existing generic contracts are sufficient.

## Semantic Models

A Semantic Model is the authoritative definition of an infrastructure domain's concepts, resource types, identities, properties, constraints, defaults, relationships, and semantic rules used to interpret and validate Intent.

It is deliberately broader than a collection of resource schemas. Resource definitions are part of a Semantic Model, but the model also defines the domain behavior necessary to determine whether Intent is meaningful and valid.

A Semantic Model SHALL remain independent of deployment targets. For example, it may define that a virtual machine has a required relationship to a network, but it does not decide how that relationship becomes a Terraform address, Bicep expression, CloudFormation reference, or other target-specific construct.

## Extension model

The current exploration identifies these principal extension boundaries:

1. **Adapters** - external source representations to parsed Intent.
2. **Infrastructure Integrations** - independently owned infrastructure domains.
3. **Semantic Models** - domain concepts, resource types, identities, constraints, defaults, relationships, and semantic rules exposed by an Integration.
4. **Backends** - Integration-owned mappings from Infrastructure IR to a published Target contract.
5. **Targets and Emitters** - deployment-technology models and physical artifact generation.

These boundaries are intended to permit independently developed Integrations and Targets without changes to Engine Core.

## Resource Graph

The Engine's canonical resolved representation is currently envisioned as an Infrastructure IR represented by a Resource Graph.

The graph contains typed resources with stable identities, properties, explicit relationships, resolved references, and deterministic dependency edges.

The graph is semantic. It should not contain Terraform resource blocks, Bicep syntax, CloudFormation templates, file paths, or other target-specific representation concerns.

The Infrastructure IR should be common in structure without requiring unrelated infrastructure domains to share artificial universal resource semantics.

## Targets and Backend compatibility

A Target represents a deployment technology through a published, versioned contract.

A Backend is a mapping between an Integration's infrastructure semantics and that Target contract. This means a Backend should not merely assume that a target implementation with a matching name will work; compatibility must be explicit and testable.

For example:

```text
VCFA Integration
    |
    +-- VCFA -> Terraform Backend
                    |
                    +-- requires Terraform Target Contract X
                                      |
                                      v
                              Terraform Target
                              - Terraform IR
                              - validation
                              - HCL emitter
                              - JSON emitter
```

Multiple Integrations may reuse the same Target, and an Integration may support multiple Targets:

```text
VCFA ----+
GCP -----+----> Terraform Target ----> HCL
Azure ---+

Azure Integration
    |
    +--> Terraform Target
    +--> Bicep Target
    +--> ARM Target
```

Initial target families under consideration include ARM, Bicep, CloudFormation, Terraform, OpenTofu, and Ansible. This is an architectural horizon, not a commitment to implement every target immediately.

Published Target contracts should eventually include reusable conformance tests so Integration authors can verify their Backends independently.

## Artifact contract

Compilation produces an Artifact Bundle rather than an unstructured collection of files.

The bundle should eventually define a stable contract for generated artifacts, diagnostics, target information, version information, provenance, and potentially the resolved graph used to produce the output.

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

- What is the precise boundary between parsed Intent and Infrastructure IR?
- Is the Resource Graph the entire Infrastructure IR or one representation within it?
- How strongly typed should Resources and their properties be?
- What are the exact Integration and Semantic Model contracts?
- How are plugin assemblies discovered, versioned, trusted, and loaded?
- What is the Target contract/version compatibility model?
- Are Target contracts distributed separately from Target implementations?
- What belongs in a reusable Target conformance test kit?
- Can compatible technologies such as Terraform and OpenTofu share some Target contracts or IR components without conflating their independent lifecycles?
- How do Targets such as Ansible fit if IR-plus-Emitter is not their natural architecture?
- What belongs in the Artifact Bundle contract?
- What is the first vertical slice that proves the architecture?

## Proposed ADRs

- [ADR-001 - Infrastructure Intent Compiler](ADR-001-infrastructure-intent-compiler.md)
- [ADR-002 - Infrastructure IR and Resource Graph](ADR-002-infrastructure-ir-resource-graph.md)
- [ADR-003 - Pluggable Backends and Emitters](ADR-003-pluggable-backends-and-emitters.md)
- [ADR-004 - Cloud-Native Operating Principles](ADR-004-cloud-native-operating-principles.md)
- [ADR-005 - Infrastructure Integrations and Extension Ownership](ADR-005-infrastructure-integrations-and-extension-ownership.md)

See also the [working glossary](glossary.md).