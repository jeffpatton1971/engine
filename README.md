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

## Architecture documents

- [Architecture overview](docs/architecture/README.md)
- [Working glossary](docs/architecture/glossary.md)
- [ADR-001: Infrastructure Intent Compiler](docs/architecture/ADR-001-infrastructure-intent-compiler.md)
- [ADR-002: Infrastructure IR and Resource Graph](docs/architecture/ADR-002-infrastructure-ir-resource-graph.md)
- [ADR-003: Pluggable Backends and Emitters](docs/architecture/ADR-003-pluggable-backends-and-emitters.md)
- [ADR-004: Cloud-Native Operating Principles](docs/architecture/ADR-004-cloud-native-operating-principles.md)

## Current design principles

1. Begin with infrastructure intent, not a deployment language.
2. Preserve semantic meaning independently of Terraform, Bicep, CloudFormation, or other targets.
3. Treat the Resource Graph as the canonical resolved infrastructure representation.
4. Make adapters, semantic models, backends, and emitters explicit extensibility boundaries.
5. Allow independently developed integrations without modification to Engine Core.
6. Produce deterministic, versioned Artifact Bundles.
7. Adopt cloud-native operating principles without requiring Kubernetes or a distributed architecture.
8. Keep architecture boundaries distinct from repository/package boundaries until real ownership and lifecycle requirements emerge.

## Working posture

This repository is intentionally a scratch space for architecture discovery. Proposed decisions should be challenged with concrete vertical slices before they are accepted. In particular, the design should be tested against both a broadly portable target such as Terraform and a cloud-native target such as Bicep or CloudFormation before target extensibility is considered proven.