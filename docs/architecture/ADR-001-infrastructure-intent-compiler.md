# ADR-001: Infrastructure Intent Compiler

- **Status:** Proposed
- **Date:** 2026-09-04
- **Decision Type:** Architecture exploration

## Context

The existing BAT architecture evolved around input providers, provider plugins, projection, and output rendering. Those boundaries contain useful ideas, but they also make the architecture easy to describe in terms of implementation stages and deployment technologies rather than the infrastructure intent being expressed.

A ground-up design should preserve flexibility without making a specific source format, cloud platform, or deployment representation the center of the system.

Terraform is an important target, but Bicep, CloudFormation, and other target-native representations may be preferable in some infrastructure domains. The architecture should permit these targets without requiring the core resource model to understand them.

The system also needs a coherent mental model for parsing, validation, identity, relationships, dependency analysis, diagnostics, target translation, and artifact production.

## Proposed decision

Model Engine as an **infrastructure intent compiler**.

The conceptual compilation pipeline is:

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

The Engine's primary responsibility is deterministic compilation of declarative infrastructure intent into deployment artifacts.

The Engine Core SHALL NOT require Terraform, Bicep, CloudFormation, or another deployment representation as part of its canonical infrastructure semantics.

## Rationale

Compiler terminology aligns naturally with the problem:

- Adapters handle external syntax and representation.
- Semantic analysis gives parsed intent infrastructure meaning.
- An intermediate representation provides a stable internal boundary.
- Backends perform target-specific lowering.
- Emitters perform physical serialization and artifact emission.

This allows the design to remain infrastructure-oriented while providing explicit extension points for different input forms and deployment technologies.

The compiler model also encourages deterministic and testable phase boundaries rather than a chain of plugins that can each reinterpret the request.

## Consequences

### Positive

- Infrastructure intent becomes the architectural center of the system.
- Target technologies do not leak into the canonical resource model.
- Terraform can be implemented deeply without becoming an architectural constraint.
- Cloud-native targets can coexist with cross-cloud targets.
- Compiler concepts provide established terminology for semantic analysis, IR, lowering, diagnostics, and emission.
- Individual phases can be tested deterministically.

### Negative / risks

- Compiler terminology may be unfamiliar to some infrastructure engineers and must be documented clearly.
- Excessive compiler abstraction could make a relatively straightforward infrastructure transformation problem unnecessarily academic.
- Target independence can become speculative abstraction if interfaces are created without concrete use cases.

## Guardrails

- Introduce abstractions only where a real phase boundary exists.
- Do not implement multiple targets merely to prove theoretical extensibility.
- Do not generalize infrastructure intent into a universal workflow or configuration language.
- Keep the domain centered on declarative infrastructure intent.

## Alternatives considered

### Terraform-first architecture

Model Terraform as the Engine's canonical output representation.

Rejected as the core architecture because it would make target-specific concepts difficult to separate later and would privilege Terraform where a native target may be more appropriate.

Terraform remains a strong candidate for the first production backend.

### Generic transformation pipeline

Retain an input -> provider -> projection -> output pipeline.

Not preferred because the stages describe implementation responsibilities but provide a weaker semantic model for parsing, analysis, intermediate representation, lowering, and emission.

### General orchestration engine

Expand the architecture beyond infrastructure into arbitrary workflows and business processes.

Rejected because it makes the domain boundary too broad. Engine is intended to compile declarative infrastructure intent, not become a general workflow platform.

## Validation

This proposal should be tested by implementing at least one complete vertical slice from source intent through artifact generation and evaluating whether a second materially different target can be modeled without changing Engine Core.