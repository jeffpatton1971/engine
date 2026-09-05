# ADR-003: Pluggable Backends and Emitters

- **Status:** Proposed
- **Date:** 2026-09-04
- **Decision Type:** Architecture exploration

## Context

Infrastructure intent may be deployable through different target technologies.

Terraform provides broad cross-platform coverage, while cloud-native representations such as Azure Bicep or AWS CloudFormation may be preferable for some domains. Other infrastructure technologies may introduce additional targets in the future.

The architecture should allow target integrations to be developed independently without making Engine Core aware of every target. At the same time, it should avoid speculative abstraction and should not require multiple target implementations merely to justify an interface.

Target translation and physical serialization are also different responsibilities. Mapping a resolved VCFA resource into Terraform semantics is not the same concern as correctly serializing a Terraform model into HCL files.

## Proposed decision

Separate target generation into **Backend**, **Target IR**, and **Emitter** responsibilities.

```text
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
Artifacts
```

### Backend

A Backend lowers resolved Infrastructure IR into a target-specific IR.

The Backend understands both the source infrastructure semantics relevant to its integration and the target deployment model.

Examples:

```text
VCFA semantics -> Terraform Backend -> Terraform IR
Azure semantics -> Terraform Backend -> Terraform IR
Azure semantics -> Bicep Backend -> Bicep IR
AWS semantics -> CloudFormation Backend -> CloudFormation IR
```

### Target IR

A Target IR represents the concepts of a deployment target in a form suitable for deterministic emission.

Unlike Infrastructure IR, a Target IR is expected to be target-shaped.

A Terraform IR may therefore contain Terraform resources, data sources, providers, variables, outputs, expressions, references, and dependency constructs.

### Emitter

An Emitter serializes a Target IR into physical artifacts.

An Emitter SHALL NOT infer infrastructure semantics or reinterpret unresolved source intent.

Examples:

```text
Terraform IR -> HCL Emitter -> .tf files
Terraform IR -> Terraform JSON Emitter -> .tf.json files
CloudFormation IR -> YAML Emitter -> template.yaml
CloudFormation IR -> JSON Emitter -> template.json
```

## Extensibility goal

The Engine SHALL permit independently developed backends and emitters without modification to Engine Core.

This is a design goal, not a commitment to a specific plugin loading mechanism.

A third party should ultimately be able to implement a new target integration using published Engine contracts and package it independently.

## Rationale

Separating backend lowering from emission prevents target serialization concerns from accumulating infrastructure-domain behavior.

It also creates reuse. Multiple infrastructure domains can lower into the same Target IR and share an emitter. Likewise, one Target IR can support multiple physical representations where the target permits it.

This architecture allows Terraform to be a first-class and deeply supported target without making Terraform the Engine's canonical model.

## Consequences

### Positive

- Target integrations can evolve independently of Engine Core.
- Common emitters can be reused across infrastructure domains.
- Target-specific concepts remain on the target side of the architecture boundary.
- Native cloud targets and cross-cloud targets can coexist.
- Testing can distinguish semantic lowering failures from serialization failures.

### Negative / risks

- Backend versus Emitter may be confusing if their responsibilities are not rigorously documented.
- Some targets may not justify a rich Target IR and separate emitter.
- A generic plugin system can introduce versioning, trust, dependency loading, and compatibility complexity.
- Independently developed extensions create a long-term contract-compatibility obligation.

## Guardrails

- Engine Core must not contain target-specific semantic branches.
- Emitters must not resolve infrastructure semantics.
- Backends must not depend on source document layout when equivalent information is available in Infrastructure IR.
- Do not require a separate repository for every interface implementation.
- Do not introduce remote/distributed plugin execution unless a concrete requirement justifies it.

## Packaging and repositories

Architecture boundaries do not automatically dictate repository boundaries.

The Engine itself is currently expected to be developed as a monorepo because its compiler phases and contracts will evolve together.

An infrastructure integration may reasonably keep its Semantic Model and one or more Backends in a single repository if they share ownership and release cadence.

Shared Target IRs and Emitters may live with Engine or in independent repositories depending on their lifecycle. This remains deliberately undecided.

## Alternatives considered

### Backend emits files directly

Allow each Backend to generate physical files itself.

Simpler initially, but it duplicates serialization behavior and encourages infrastructure semantics, target semantics, and file generation to become entangled.

### Terraform as the only target contract

Make Terraform the sole target representation and extension surface.

Not proposed because cloud-native deployment targets are legitimate alternatives and because third-party target extensibility is an explicit goal.

### Universal emitter interface over strings/files

Have all target integrations return arbitrary file content.

Rejected as the primary model because it removes a useful target-model boundary and makes deterministic validation of target output more difficult.

## Open questions

- What is the minimum viable Backend contract?
- Does each infrastructure-domain/target pair require its own Backend, or can mappings be composed?
- Which Target IRs should Engine maintain as first-party components?
- How are backend and emitter capabilities discovered?
- How are extension compatibility and API versions negotiated?
- Should extensions execute in-process initially?
- What trust/signing model is eventually required for third-party extensions?