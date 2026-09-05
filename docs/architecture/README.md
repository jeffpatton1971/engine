# Architecture Exploration

> **Decision Status: Exploration**

This directory records the current architectural exploration for Engine. It is intentionally a design workspace: terminology and proposed decisions should be expected to evolve as the model is tested against real infrastructure domains and deployment targets.

## Working premise

Engine is an infrastructure intent compiler.

It accepts declarative infrastructure intent, resolves that intent against infrastructure semantic models, constructs a deterministic intermediate representation, lowers that representation through a target backend, and emits a versioned artifact bundle.

```text
Source Intent
    |
    v
Adapter
    |
    v
Parsed Intent
    |
    v
Semantic Analysis
    |
    v
Infrastructure IR
(Resource Graph)
    |
    v
Backend
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

## Why start from intent

The architecture should describe infrastructure independently of the mechanism eventually used to deploy it.

Terraform is an important and likely early target, but it should not define the Engine's resource model. A cloud-native target such as Bicep or CloudFormation may be preferable for a particular domain, and future users should be able to introduce their own backends and emitters without modifying Engine Core.

This gives the architecture a deliberate separation:

- **Infrastructure semantics** describe what resources mean.
- **Infrastructure IR** represents resolved infrastructure meaning.
- **Backends** map that meaning into a deployment target.
- **Emitters** turn target models into physical artifacts.

## Extension model

The current exploration identifies four principal extension boundaries:

1. **Adapters** - external source representations to parsed intent.
2. **Semantic Models** - infrastructure resource types, constraints, defaults, identities, and relationships.
3. **Backends** - Infrastructure IR to target-specific IR.
4. **Emitters** - target-specific IR to physical artifacts.

These boundaries are intended to permit independently developed integrations without changes to Engine Core.

The exact packaging, repository, and runtime discovery mechanisms are intentionally undecided.

## Resource graph

The Engine's canonical resolved representation is currently envisioned as an Infrastructure IR represented by a Resource Graph.

The graph contains typed resources with stable identities, properties, explicit relationships, resolved references, and deterministic dependency edges.

The graph is semantic. It should not contain Terraform resource blocks, Bicep syntax, CloudFormation templates, file paths, or other target-specific representation concerns.

## Target model

A Backend lowers Infrastructure IR into a Target IR. Target IRs are allowed to be strongly shaped by their target because they exist on the target side of the architecture boundary.

For example:

```text
Infrastructure IR
    |
    +--> Terraform Backend --> Terraform IR --> HCL Emitter
    |
    +--> Azure Bicep Backend --> Bicep IR --> Bicep Emitter
    |
    +--> AWS CloudFormation Backend --> CloudFormation IR --> JSON/YAML Emitter
```

Only one backend needs to exist initially. The abstraction exists because target lowering is a real architectural boundary, not because the project promises to implement every possible deployment technology.

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

Adapters, semantic models, backends, and emitters may be developed outside the Engine repository. Repository topology should follow real ownership and release boundaries rather than mirroring every architecture interface one-for-one.

## Open questions

The following remain deliberately unresolved:

- Is **Semantic Model** the best term for the infrastructure-domain definition?
- What is the precise boundary between parsed Intent and Infrastructure IR?
- Is the Resource Graph the entire Infrastructure IR or one representation within it?
- Which semantic-model capabilities should be declarative data versus executable code?
- How are extensions discovered, versioned, packaged, trusted, and loaded?
- What belongs in the Artifact Bundle contract?
- What is the first vertical slice that proves the architecture?
- Should the first target be Terraform, a cloud-native target, or both as a deliberate architecture test?

## Proposed ADRs

- [ADR-001 - Infrastructure Intent Compiler](ADR-001-infrastructure-intent-compiler.md)
- [ADR-002 - Infrastructure IR and Resource Graph](ADR-002-infrastructure-ir-resource-graph.md)
- [ADR-003 - Pluggable Backends and Emitters](ADR-003-pluggable-backends-and-emitters.md)
- [ADR-004 - Cloud-Native Operating Principles](ADR-004-cloud-native-operating-principles.md)

See also the [working glossary](glossary.md).