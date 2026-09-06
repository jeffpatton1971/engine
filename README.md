# Engine

> **Status: Architecture Exploration**

This repository is a temporary design space for exploring a ground-up infrastructure intent engine.

Engine compiles declarative infrastructure Intent into deterministic deployment artifacts without making Terraform, Bicep, CloudFormation, or another Target representation part of the core semantic model.

Nothing here is final unless explicitly marked accepted. ADRs remain **Proposed** while concrete pressure tests shape the contracts.

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

Semantic Analysis is multi-phase. Integrations materialize managed/existing typed domain nodes, canonical identities/references, and domain diagnostics. Engine resolves those semantics into a deterministic Resource Graph. Backends determine whether a valid domain graph can be represented by their Target contract generation; Targets validate and emit the resulting Target model.

Compilation-specific context belongs to one Intent/compilation and is never global Engine state.

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
- [Azure pressure-test scenarios](docs/architecture/scenarios/)

## Current design principles

1. Begin with infrastructure Intent, not a deployment language.
2. Keep Semantic Models and Domain Abstractions Target-independent.
3. Treat the deterministic Resource Graph as canonical resolved infrastructure semantics.
4. Keep principal loadable extension types to Adapters, Integrations, and Targets unless evidence justifies more.
5. Let Integrations own domain semantics, canonical identity/scoping, managed/existing domain types, local validation, and Backends.
6. Let Engine own identity enforcement, reference resolution, graph construction/integrity, traversal, and deterministic managed ordering.
7. Model domain semantic type independently from managed/existing lifecycle.
8. Make typed references target domain contracts rather than lifecycle-specific implementations.
9. Preserve semantic information through IR so Backends never need raw/Parsed Intent to recover accepted domain meaning.
10. Scope Compilation Context to one Intent/compilation; never treat it as global Engine state.
11. Let Backends own Target representability and Targets own Target-model validation/emission.
12. Make contract compatibility explicit and conformance-tested.
13. Produce deterministic versioned Artifact Bundles.
14. Adopt cloud-native operating principles without requiring Kubernetes or distributed architecture.
15. Add abstractions only when a concrete independently owned/evolving concern requires them.

## Working posture

The Azure VM pressure-test series has validated the major semantic-analysis boundaries and exposed concrete requirements around scoped identity, brownfield resources, typed references, mixed-lifecycle dependencies, semantic-lossless IR, per-compilation context, and Target representability.

The next design work should use those findings to shape minimal public contracts rather than adding abstractions without a concrete pressure case.