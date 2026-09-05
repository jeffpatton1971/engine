# ADR-006: Target Contracts and Backend Dependency Model

- **Status:** Proposed
- **Date:** 2026-09-05
- **Decision Type:** Architecture exploration

## Context

Infrastructure Integrations own Backends that map domain semantics into deployment Targets. Backend authors therefore need a stable, strongly typed way to construct Target-specific models without depending on the concrete Target implementation.

A Backend must be able to express concepts that are genuinely Target-specific. A Terraform Backend, for example, may need to construct Terraform resources, expressions, variables, outputs, references, and other Terraform concepts. Moving those concepts into Engine Core would make Engine Target-aware. Hiding them behind untyped dictionaries, reflection, or dynamic invocation would trade an explicit compile-time dependency for weaker runtime coupling.

The architecture therefore needs a clear dependency boundary between Engine, Integration Backends, Target contracts, Target implementations, and conformance testing.

## Proposed decision

Every Target SHALL publish a separate, versioned **Target Abstractions** assembly containing the public contract and Target model required by Integration Backends.

An Integration Backend SHALL compile against:

- Engine abstractions required for Backend participation; and
- the Abstractions assembly of each Target it explicitly supports.

An Integration Backend SHALL NOT reference the concrete Target implementation assembly.

Conceptually:

```text
Engine.Abstractions
      ^
      |
Target.Abstractions
      ^
      |
Integration Backend
```

The concrete Target implementation independently references the same Target Abstractions:

```text
                 Target.Abstractions
                  ^              ^
                  |              |
       Integration Backend    Target Implementation
```

Engine composes the Backend and Target implementation at runtime after validating their Target identity and contract compatibility.

## Example

A Terraform-capable VCFA Integration may contain:

```csharp
using InfrastructureIntent.Abstractions;
using Terraform.Target.Abstractions;

public sealed class VcfaTerraformBackend
    : IBackend<TerraformDocument>
{
    // Maps resolved VCFA Infrastructure IR into Terraform Target IR.
}
```

The Backend assembly may depend on:

```text
InfrastructureIntent.Abstractions.dll
Terraform.Target.Abstractions.dll
```

It SHALL NOT depend on:

```text
Terraform.Target.dll
```

At runtime:

```text
Engine
  |
  +-- loads VCFA Integration / Terraform Backend
  +-- loads Terraform Target
  +-- verifies Target identity and contract compatibility
  +-- invokes Backend lowering
  +-- passes resulting Target model to the Target
```

## Target package shape

A Target is expected to publish at least three independently consumable concerns:

```text
<Target>.Target.Abstractions
    public Target contract and Target model

<Target>.Target
    concrete Target implementation, validation, and emission

<Target>.Target.Conformance
    reusable Backend conformance suite
```

Package and assembly names are illustrative. The separation of responsibilities is the decision.

For Terraform this may resemble:

```text
Terraform.Target.Abstractions
    TerraformDocument
    TerraformResource
    TerraformExpression
    TerraformVariable
    TerraformOutput
    TerraformReference
    Target identity / contract metadata

Terraform.Target
    runtime Target implementation
    Target-model validation
    HCL emission
    Terraform JSON emission

Terraform.Target.Conformance
    required reusable Backend conformance tests
```

## Target contract contents

The Target Abstractions assembly SHALL contain only the public surface needed for a Backend to produce a valid Target model and declare compatibility.

The contract may include:

- Target identity;
- Target contract version;
- Target model / Target IR types;
- Target-specific expression or reference types;
- Target-specific diagnostics or result contracts where required;
- Backend compatibility metadata;
- abstractions required to submit the Target model to the Target runtime.

The Target contract SHALL NOT contain:

- infrastructure-domain semantics;
- Integration-specific mappings;
- concrete emitter implementations;
- deployment credentials;
- live infrastructure state;
- private Target implementation details;
- runtime execution or apply behavior unless a future ADR explicitly expands Target responsibility.

## Backend contract

Engine SHOULD provide a generic Backend contract that preserves compile-time Target-model typing.

An illustrative shape is:

```csharp
public interface IBackend<TTargetModel>
{
    TargetRequirement Target { get; }

    CompilationResult<TTargetModel> Lower(
        InfrastructureIr input,
        BackendContext context);
}
```

This example is illustrative rather than an accepted API contract.

The important property is that the Backend's output type is defined by the Target Abstractions assembly rather than by Engine Core or the concrete Target implementation.

## Compile-time and runtime compatibility

Compatibility has two distinct layers.

### Compile time

The Backend author references the Target Abstractions package.

This provides:

- strongly typed Target primitives;
- compile-time API compatibility;
- explicit dependency on a Target contract rather than implementation details.

### Test time

The Backend author references the Target Conformance package as a test dependency.

This provides executable proof that the Backend satisfies the behavioral requirements of the Target contract.

### Runtime

Engine discovers the Backend and Target implementation independently and verifies that:

- Target IDs match;
- the installed Target contract version satisfies the Backend's requirement;
- incompatible combinations fail before compilation proceeds.

## Versioning

Target implementation versions and Target contract versions are separate concerns.

For example:

```text
Terraform Target implementation: 4.7.3
Terraform Target contract:       1.2
```

A Backend depends on the Target **contract version**, not the Target implementation version.

A Target implementation may release fixes and internal improvements without forcing Integration authors to rebuild so long as it continues to honor a compatible Target contract.

Breaking changes to the public Target contract require an incompatible contract version.

The precise range syntax and version-negotiation algorithm remain open, but Target contract compatibility SHALL be explicit.

## Conformance dependency

The Target Conformance package is normally a development/test dependency, not a runtime Backend dependency.

For example:

```text
Vcfa.Integration.Terraform
    -> InfrastructureIntent.Abstractions
    -> Terraform.Target.Abstractions

Vcfa.Integration.Terraform.Tests
    -> Vcfa.Integration.Terraform
    -> Terraform.Target.Conformance
```

The Backend must pass the applicable conformance suite before claiming Target compatibility, as required by ADR-005.

## Dependency rules

The following dependencies are permitted:

```text
Integration Backend -> Engine.Abstractions
Integration Backend -> Target.Abstractions
Target Implementation -> Target.Abstractions
Target Conformance -> Target.Abstractions
Engine runtime -> Engine.Abstractions
```

The following dependencies are prohibited:

```text
Integration Backend -X-> Target Implementation
Target Implementation -X-> Integration
Engine Core -X-> Target-specific model types
Target.Abstractions -X-> Integration-specific semantics
```

## Rationale

The dependency itself is not the architectural problem. Hidden or unstable dependencies are.

A small, separately versioned Target contract gives Backend developers compile-time safety while allowing the Target implementation to evolve independently.

Passing Target-specific models indirectly through Engine would force Engine to become aware of Target semantics or require weakly typed runtime mechanisms. Directly referencing the concrete Target implementation would couple Integration releases to Target implementation details.

A dedicated Target Abstractions assembly provides the cleanest boundary between these concerns.

## Consequences

### Positive

- Backend developers receive a strongly typed Target SDK-like surface.
- Engine Core remains Target-independent.
- Target implementations can evolve independently behind stable contracts.
- Backends do not reference concrete Target implementations.
- Compatibility failures can be detected at compile, test, and runtime boundaries.
- Conformance remains a test concern rather than a production dependency.
- Third-party Backend development requires only published contracts and conformance tooling.

### Negative / risks

- Each Target introduces at least one public contract assembly and associated versioning responsibility.
- Target authors must maintain binary/source compatibility intentionally.
- Poorly designed Target abstractions could expose too much implementation detail and become difficult to evolve.
- In-process plugin loading may still create assembly-version and dependency-resolution challenges.

## Guardrails

- Target Abstractions SHALL be kept smaller and more stable than the Target implementation.
- Engine Core SHALL NOT absorb Target-specific primitives for convenience.
- Backend authors SHALL NOT reference the concrete Target implementation.
- Public Target abstractions SHALL represent Target concepts, not implementation mechanics.
- Conformance suites SHALL test the public Target contract rather than private implementation behavior.
- Contract versioning SHALL be independent from Target implementation versioning.
- Engine SHALL validate Target contract compatibility before invoking a Backend.

## Alternatives considered

### Engine brokers all Target-specific interfaces

Rejected because Engine would need to understand or relay every Target-specific concept, making Core increasingly Target-aware.

### Backend references the concrete Target implementation

Rejected because it couples Integration Backends to Target implementation releases and private implementation details.

### Untyped Target documents

Return dictionaries, generic object graphs, serialized JSON, or dynamic values from Backends.

Rejected as the primary model because it moves structural failures from compile time to runtime and weakens third-party developer experience.

### Reflection or dynamic invocation

Rejected as the primary contract mechanism because it hides dependencies rather than removing them and makes compatibility failures harder to diagnose.

## Open questions

- What is the exact generic Backend interface?
- What is the minimum common metadata every Target Abstractions package must expose?
- Should Target contract packages use semantic versioning directly or expose a distinct contract-version identifier?
- What version-range model should Backends declare?
- How does Engine discover Target contract metadata without eagerly activating a plugin?
- How are incompatible assembly versions isolated in the initial in-process plugin model?
- Which validation belongs in the Target contract/conformance suite versus the concrete Target runtime?