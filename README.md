# Engine

> **Status: Architecture Exploration**

This repository is a temporary design space for exploring a ground-up infrastructure intent engine.

The working premise is that Engine compiles declarative infrastructure intent into deterministic deployment artifacts without making Terraform, Bicep, CloudFormation, or another target representation part of the core semantic model.

Nothing documented here should be considered final unless explicitly marked as accepted. Early ADRs are intentionally **Proposed** so competing ideas can be recorded and challenged before implementation hardens the architecture.

## Current architecture sketch

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
     Target Validation
       and Emission
              |
              v
       Artifact Bundle
```

Semantic Analysis is intentionally multi-phase. Integrations materialize and locally validate concrete typed domain resources; Engine registers identities, resolves typed references, constructs and validates graph edges, detects cycles, and produces the deterministic Resource Graph. Targets perform target-model validation after Backend lowering.

## Architecture documents

- [Architecture overview](docs/architecture/README.md)
- [Working glossary](docs/architecture/glossary.md)
- [ADR-001: Infrastructure Intent Compiler](docs/architecture/ADR-001-infrastructure-intent-compiler.md)
- [ADR-002: Infrastructure IR and Resource Graph](docs/architecture/ADR-002-infrastructure-ir-resource-graph.md)
- [ADR-003: Pluggable Backends and Emitters](docs/architecture/ADR-003-pluggable-backends-and-emitters.md)
- [ADR-004: Cloud-Native Operating Principles](docs/architecture/ADR-004-cloud-native-operating-principles.md)
- [ADR-005: Infrastructure Integrations and Extension Ownership](docs/architecture/ADR-005-infrastructure-integrations-and-extension-ownership.md)
- [ADR-006: Target Contracts and Backend Dependency Model](docs/architecture/ADR-006-target-contracts-and-backend-dependencies.md)
- [ADR-007: Contract Evolution and Compatibility](docs/architecture/ADR-007-contract-evolution-and-compatibility.md)
- [ADR-008: Domain Abstractions and Typed Resource Graphs](docs/architecture/ADR-008-domain-abstractions-and-typed-resource-graphs.md)
- [ADR-009: Resource Identity, References, and Graph Edge Semantics](docs/architecture/ADR-009-resource-identity-references-and-graph-edges.md)
- [ADR-010: Semantic Model and Semantic Analysis Lifecycle](docs/architecture/ADR-010-semantic-model-and-semantic-analysis-lifecycle.md)

## Current design principles

1. Begin with infrastructure intent, not a deployment language.
2. Preserve semantic meaning independently of Terraform, Bicep, CloudFormation, or other Targets.
3. Treat the Resource Graph as the canonical resolved infrastructure representation.
4. Keep the principal independently loadable extension types to Adapters, Infrastructure Integrations, and Targets unless demonstrated requirements justify more.
5. Let Integrations own domain semantics, Domain Abstractions, resource materialization, local validation, and domain-to-Target Backends.
6. Let Engine own semantic-analysis orchestration, identity enforcement, reference resolution, graph construction, graph validation, and deterministic graph behavior.
7. Let Targets own Target contracts, target-model validation, conformance, and emission.
8. Allow independently developed Integrations and Targets without modification to Engine Core when existing generic contracts are sufficient.
9. Produce deterministic, versioned Artifact Bundles.
10. Adopt cloud-native operating principles without requiring Kubernetes or a distributed architecture.
11. Keep architecture boundaries distinct from repository/package boundaries until real ownership and lifecycle requirements emerge.

## Working posture

This repository is intentionally a scratch space for architecture discovery. Proposed decisions should be challenged with concrete vertical slices before they are accepted.

The immediate validation step is to pressure-test the semantic-analysis lifecycle with a small real infrastructure model containing multiple resource types, forward references, relationships, dependencies, validation failures, and deterministic graph ordering before freezing the public Semantic Model API.