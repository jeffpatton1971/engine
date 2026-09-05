# ADR-003: Pluggable Backends and Emitters

- **Status:** Proposed
- **Date:** 2026-09-04
- **Decision Type:** Architecture exploration

## Context

Infrastructure Intent may be deployable through different deployment technologies.

Terraform provides broad cross-platform coverage, while technologies such as Azure Bicep or AWS CloudFormation may be preferable for some domains. OpenTofu, Ansible, and future technologies may introduce additional Targets.

The architecture should allow Integrations and Targets to be developed independently without making Engine Core aware of every infrastructure domain or deployment technology.

Domain-to-Target translation and Target emission are different responsibilities. Mapping a resolved SDDC Flex resource into Terraform semantics is not the same concern as validating and serializing a Terraform model into HCL files.

## Proposed decision

Separate Target generation into **Integration Backend**, **Target model / Target IR**, and **Target emission** responsibilities.

```text
Infrastructure IR / Resource Graph
        |
        v
Integration Backend
        |
        v
Target Abstractions
(Target model / Target IR)
        |
        v
Target
(validation + emission)
        |
        v
Artifacts
```

### Integration Backend

A Backend is owned by an Infrastructure Integration and lowers that Integration's resolved typed domain resources into one specific published Target contract.

The Backend therefore bridges two strongly typed contracts:

```text
Domain Abstractions
        |
        v
      Backend
        |
        v
Target Abstractions
```

Examples:

```text
SddcFlex.Abstractions -> SddcFlex.Backend.Terraform -> Terraform.Target.Abstractions
SddcFlex.Abstractions -> SddcFlex.Backend.OpenTofu  -> OpenTofu.Target.Abstractions
Azure.Abstractions    -> Azure.Backend.Bicep        -> Bicep.Target.Abstractions
AWS.Abstractions      -> AWS.Backend.CloudFormation -> CloudFormation.Target.Abstractions
```

A Backend SHALL NOT reinterpret raw Intent to recover domain information that should have survived semantic analysis. It consumes the lossless resolved Resource Graph.

### Target model / Target IR

A Target Abstractions contract defines the public model a Backend must construct for that deployment technology.

Unlike Infrastructure IR, a Target model is expected to be Target-shaped.

A Terraform Target contract may therefore contain Terraform resources, data sources, providers, variables, outputs, expressions, references, and dependency constructs.

The exact internal architecture of a Target is not required to use a rich IR when that does not fit the technology. The public requirement is a stable contract that Backends can produce and the Target can validate and consume.

### Target and Emitter

A Target owns validation and physical emission of its Target model. It may contain one or more Emitters when multiple physical representations are useful.

An Emitter SHALL NOT infer infrastructure-domain semantics or reinterpret unresolved source Intent.

Examples:

```text
Terraform Target model -> HCL Emitter            -> .tf files
Terraform Target model -> Terraform JSON Emitter -> .tf.json files
CloudFormation model   -> YAML Emitter           -> template.yaml
CloudFormation model   -> JSON Emitter           -> template.json
```

Emitter is a Target responsibility and is not required to be an independently loaded Engine plugin.

## Extensibility model

The principal independently loaded extension types are Adapters, Integrations, and Targets.

Backends are Integration-owned and may be packaged with an Integration or separately when ownership and release cadence justify it. Emitters are Target-owned and may be internal components of a Target.

This keeps the runtime extension model smaller than the conceptual architecture.

A third party should be able to implement an Integration and its Backend using published Engine, Domain, and Target Abstractions without referencing Engine or Target implementation internals.

## Backend and Target dependencies

A Backend may depend on:

```text
Engine.Abstractions
<Integration>.Abstractions
<Target>.Target.Abstractions
```

It SHALL NOT depend on the concrete Target implementation.

For example:

```text
SddcFlex.Backend.Terraform
    -> Engine.Abstractions
    -> SddcFlex.Abstractions
    -> Terraform.Target.Abstractions

Terraform.Target
    -> Terraform.Target.Abstractions
```

Engine composes the Backend and Target at runtime only after verifying explicit Target identity and supported contract generation.

## Distinct Targets

Distinct deployment technologies are modeled as distinct Targets even when they share syntax, ancestry, or implementation behavior.

Terraform and OpenTofu are therefore separate Targets. An Integration that supports both provides explicit Backend support for both.

Engine SHALL NOT introduce a Target-family or compatibility-profile abstraction solely to deduplicate similar technologies. Implementations may share internal libraries where useful.

## Contract evolution and conformance

Backend/Target compatibility is based on explicit contract generations rather than Target implementation-version ranges.

Every published Target SHALL provide a versioned Backend conformance suite. Every Backend claiming compatibility with a Target contract generation SHALL pass that generation's suite.

Published contract generations are immutable in their required public shape and semantics as defined by ADR-007. A newer Target implementation may continue supporting older contract generations only while it explicitly advertises and demonstrates that support through conformance.

The Backend author decides when to migrate to a newer Target contract generation. The existence of a newer generation does not itself invalidate an older supported Backend.

## Rationale

Separating Backend lowering from Target validation and emission prevents Target serialization concerns from accumulating infrastructure-domain behavior and prevents Integrations from duplicating common Target behavior.

Strongly typed Domain Abstractions on the input side and Target Abstractions on the output side give the Backend a precise responsibility: translate one published semantic contract into another.

This architecture allows Terraform to be a first-class and deeply supported Target without making Terraform Engine's canonical model and allows similar but independently evolving technologies such as OpenTofu to remain explicit.

## Consequences

### Positive

- Integrations and Targets can evolve independently of Engine Core.
- Backends have a precise domain-to-Target responsibility.
- Strong typing exists on both sides of Backend lowering.
- Target validation and emission can be reused across multiple infrastructure Integrations.
- Target-specific concepts remain outside Engine Core.
- Testing can distinguish semantic-analysis, Backend-lowering, Target-validation, and emission failures.
- Implementation releases need not force downstream rebuilds while contract generations remain supported.

### Negative / risks

- Backend versus Target responsibility must remain rigorously documented.
- Each Integration/Target combination requires explicit Backend support.
- Public Domain and Target contracts create long-term compatibility obligations.
- In-process plugin loading introduces type-identity and dependency-isolation concerns.
- Some Targets may not naturally use an IR-plus-Emitter architecture and must not be forced into one.

## Guardrails

- Engine Core SHALL NOT contain Target-specific semantic branches.
- Backends SHALL NOT depend on raw source layout or raw Intent when equivalent resolved information belongs in Infrastructure IR.
- Backends SHALL NOT reference concrete Target implementations.
- Backends SHALL NOT redefine domain semantics owned by their Integration.
- Targets and Emitters SHALL NOT resolve infrastructure-domain semantics.
- Emitters are Target responsibilities, not mandatory independent plugins.
- Distinct deployment technologies SHALL remain distinct Targets unless repeated evidence demonstrates a genuine shared public abstraction.
- Do not require a separate repository for every architectural interface.
- Do not introduce remote/distributed plugin execution without a concrete requirement.

## Packaging and repositories

Architecture boundaries do not automatically dictate repository boundaries.

Engine itself is currently expected to be developed as a monorepo because its compiler phases and contracts will evolve together.

An Infrastructure Integration may keep its Semantic Model, Domain Abstractions, and one or more Backends in a single repository when they share ownership. A Backend may be separately packaged when it has an independent release cadence.

A Target owns its Target Abstractions, implementation, Emitters, and conformance suite. These may be one repository with multiple packages or separate repositories according to real ownership and lifecycle needs.

## Alternatives considered

### Backend emits files directly

Allow each Backend to generate physical files itself.

Rejected as the primary architecture because it duplicates Target serialization behavior and encourages infrastructure semantics, Target semantics, and file generation to become entangled.

### Terraform as the only Target contract

Make Terraform the sole Target representation and extension surface.

Rejected because other deployment technologies are legitimate Targets and third-party Target extensibility is an explicit goal.

### Universal emitter interface over strings/files

Have all Target integrations return arbitrary file content.

Rejected as the primary model because it removes a useful Target-model boundary and makes deterministic validation of Target output more difficult.

### Target families or profiles

Represent similar technologies such as Terraform and OpenTofu through a shared family/profile abstraction.

Rejected for now. Similarity does not establish durable compatibility, and the additional profile-version dimension would complicate Backend ownership and compatibility without demonstrated need.

## Open questions

- What is the exact generic Backend interface and runtime invocation bridge?
- What is the minimum common metadata every Backend and Target must expose?
- How are Backend and Target capabilities discovered without unnecessary plugin activation?
- How are incompatible contract generations isolated in the initial in-process plugin model?
- Which validation belongs in Target conformance versus the concrete Target runtime?
- How should a Target such as Ansible fit when a traditional Target IR plus Emitter is not its natural internal architecture?