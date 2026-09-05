# ADR-006: Target Contracts and Backend Dependency Model

- **Status:** Proposed
- **Date:** 2026-09-05
- **Decision Type:** Architecture exploration

## Context

Infrastructure Integrations own Backends that lower resolved domain semantics into deployment Targets. Backend authors therefore need a stable, strongly typed way to consume their Integration's domain resources and construct Target-specific models without depending on concrete Integration or Target implementations.

A Terraform Backend, for example, may need to construct Terraform resources, expressions, variables, outputs, references, and other Terraform concepts. Moving those concepts into Engine Core would make Engine Target-aware. Hiding them behind untyped dictionaries, reflection, or dynamic invocation would trade explicit compile-time contracts for weaker runtime coupling.

The architecture therefore needs clear dependency boundaries between Engine, Domain Abstractions, Integration Backends, Target contracts, Target implementations, and conformance testing.

## Proposed decision

Every Target SHALL publish a separate, versioned **Target Abstractions** contract containing the public Target model required by Integration Backends.

An Integration Backend SHALL compile against:

- Engine Abstractions required for Backend participation;
- its Integration's Domain Abstractions generation; and
- the Target Abstractions generation of the specific Target it supports.

An Integration Backend SHALL NOT reference the concrete Target implementation assembly.

Conceptually:

```text
Engine.Abstractions
      ^
      |
Domain Abstractions --------+
                            |
Target Abstractions --------+--> Integration Backend
      ^
      |
Target Implementation
```

The concrete Target implementation independently references its Target Abstractions. Engine composes the Backend and Target implementation at runtime after validating Target identity and explicit contract-generation compatibility.

## Example

A Terraform-capable SDDC Flex Backend may resemble:

```csharp
using Engine.Abstractions;
using SddcFlex.Abstractions;
using Terraform.Target.Abstractions;

public sealed class SddcFlexTerraformBackend
    : IBackend<TerraformDocument>
{
    // Lowers resolved typed SDDC Flex resources into the Terraform Target model.
}
```

The Backend assembly may depend on:

```text
Engine.Abstractions.dll
SddcFlex.Abstractions.V1.dll
Terraform.Target.Abstractions.V1.dll
```

It SHALL NOT depend on:

```text
SddcFlex.Integration.dll
Terraform.Target.dll
```

At runtime:

```text
Engine
  |
  +-- loads SDDC Flex Integration / Backend
  +-- loads Terraform Target
  +-- verifies Target identity and supported contract generation
  +-- invokes Backend lowering over the typed Resource Graph
  +-- passes the resulting Target model to the Target
```

Names are illustrative; the dependency boundaries are the decision.

## Target package shape

A Target is expected to publish at least three independently consumable concerns:

```text
<Target>.Target.Abstractions.<Generation>
    public Target contract and Target model

<Target>.Target
    concrete Target implementation, validation, and emission

<Target>.Target.Conformance.<Generation>
    reusable Backend conformance suite
```

For Terraform this may resemble:

```text
Terraform.Target.Abstractions.V1
    TerraformDocument
    TerraformResource
    TerraformExpression
    TerraformVariable
    TerraformOutput
    TerraformReference
    Target identity / contract-generation metadata

Terraform.Target
    runtime Target implementation
    Target-model validation
    HCL emission
    Terraform JSON emission

Terraform.Target.Conformance.V1
    required reusable Backend conformance tests
```

Exact package and assembly naming remains open.

## Target contract contents

The Target Abstractions assembly SHALL contain only the public surface needed for a Backend to produce a valid Target model and declare compatibility.

The contract may include:

- Target identity;
- Target contract-generation identity;
- Target model / Target IR types;
- Target-specific expression or reference types;
- Target-specific diagnostics or result contracts where required;
- Backend compatibility metadata;
- abstractions required to submit the Target model to the Target runtime.

The Target contract SHALL NOT contain infrastructure-domain semantics, Integration-specific mappings, concrete emitter implementations, deployment credentials, live infrastructure state, or private Target implementation details.

## Backend contract

Engine SHOULD provide a Backend contract that preserves compile-time Target-model typing for Backend authors while also permitting Engine to invoke Backends without becoming aware of Target-specific CLR types.

An illustrative author-facing shape is:

```csharp
public interface IBackend<TTargetModel>
{
    TargetRequirement Target { get; }

    CompilationResult<TTargetModel> Lower(
        IResourceGraph input,
        BackendContext context);
}
```

This remains illustrative rather than an accepted API contract.

The important properties are:

- Backend input is the resolved, lossless Resource Graph;
- the Backend may consume its Integration's concrete typed resources through Domain Abstractions;
- Backend output types come from Target Abstractions;
- Engine Core does not need compile-time knowledge of either domain-specific or Target-specific implementation types.

The exact non-generic runtime invocation bridge remains open.

## Compile-time, test-time, and runtime compatibility

### Compile time

The Backend author references the applicable Engine, Domain, and Target Abstractions packages. This provides strong typing and explicit dependencies on public contracts rather than implementation details.

### Test time

The Backend author references the applicable Target Conformance package as a test dependency. Domain conformance may also be used to prove compatibility with the Domain Abstractions generation.

### Runtime

Engine discovers the Backend and Target independently and verifies:

- Target IDs match;
- the Target explicitly supports the contract generation required by the Backend;
- the Integration/Backend graph types are compatible with the loaded Domain Abstractions generation;
- incompatible combinations fail before lowering begins.

## Contract generations

Target implementation versions and Target contract generations are separate concerns.

For example:

```text
Terraform Target implementation: 4.7.3
Supported contracts: V1, V2
```

A Backend compiled against Terraform Contract V1 does not require Terraform Target implementation 1.x or a semantic-version range. It requires an installed Terraform Target that explicitly supports V1.

A newer Target implementation may advertise support for V1 only while it passes V1 conformance.

A Backend may remain on V1 while V2 or V3 exists if the installed Target continues to support V1. The Backend author decides when to migrate; existence of a newer generation does not force migration.

Breaking contract generations SHOULD be independently addressable, potentially through distinct packages and assemblies, as defined by ADR-007.

## Conformance dependency

The Target Conformance package is normally a development/test dependency, not a runtime Backend dependency.

```text
SddcFlex.Backend.Terraform
    -> Engine.Abstractions
    -> SddcFlex.Abstractions.V1
    -> Terraform.Target.Abstractions.V1

SddcFlex.Backend.Terraform.Tests
    -> SddcFlex.Backend.Terraform
    -> Terraform.Target.Conformance.V1
```

The Backend must pass the applicable conformance suite before claiming Target compatibility.

## Dependency rules

Permitted dependencies include:

```text
Domain Abstractions -> Engine.Abstractions
Integration -> Engine.Abstractions
Integration -> Domain Abstractions
Integration Backend -> Engine.Abstractions
Integration Backend -> Domain Abstractions
Integration Backend -> Target Abstractions
Target Implementation -> Target Abstractions
Target Conformance -> Target Abstractions
Engine runtime -> Engine.Abstractions
```

Prohibited dependencies include:

```text
Integration Backend -X-> Target Implementation
Integration Backend -X-> Integration Implementation
Target Implementation -X-> Integration
Engine Core -X-> Integration-specific resource types
Engine Core -X-> Target-specific model types
Target Abstractions -X-> Integration-specific semantics
Domain Abstractions -X-> Target-specific semantics
```

## Rationale

The dependency itself is not the architectural problem. Hidden, unstable, or implementation-level dependencies are.

Small separately versioned Domain and Target contracts give Backend developers compile-time safety while allowing Integration, Engine, and Target implementations to evolve independently.

Passing Target-specific models through weakly typed dictionaries would move structural failures to runtime. Directly referencing Target implementations would couple Integration releases to implementation details.

Contract generations plus conformance evidence allow implementation releases to evolve without automatically cascading rebuilds through dependent components.

## Consequences

### Positive

- Backend developers receive strongly typed domain and Target SDK-like surfaces.
- Engine Core remains both infrastructure-domain and Target independent.
- Target implementations can evolve independently behind supported contract generations.
- Backends do not reference concrete Integration or Target implementations.
- Compatibility failures can be detected at compile, test, and runtime boundaries.
- Conformance remains primarily a development/test concern.
- Third-party Backend development requires only published contracts and conformance tooling.

### Negative / risks

- Each Domain and Target introduces public contract assemblies and associated versioning responsibilities.
- Contract authors must maintain compatibility intentionally.
- Poorly designed abstractions can become difficult to evolve.
- In-process plugin loading must handle assembly generations and CLR type identity deliberately.
- Engine still needs a runtime invocation mechanism that can compose strongly typed generic Backends without compile-time Target-model knowledge.

## Guardrails

- Target Abstractions SHALL be smaller and more stable than Target implementations.
- Domain Abstractions SHALL remain Target-independent.
- Engine Core SHALL NOT absorb Target-specific or Integration-specific primitives for convenience.
- Backend authors SHALL NOT reference concrete Integration or Target implementations.
- Public Target abstractions SHALL represent Target concepts, not implementation mechanics.
- Conformance suites SHALL test public contracts rather than private implementation behavior.
- Contract generations SHALL be independent from implementation versions.
- Engine SHALL validate contract compatibility before invoking a Backend.

## Alternatives considered

### Engine brokers all Target-specific interfaces

Rejected because Engine would need to understand or relay every Target-specific concept, making Core increasingly Target-aware.

### Backend references concrete implementations

Rejected because it couples Integration Backends to Integration or Target implementation releases and private details.

### Untyped Target documents

Rejected as the primary model because dictionaries, generic object graphs, serialized JSON, or dynamic values move structural failures from compile time to runtime.

### Reflection or dynamic invocation as the authoring contract

Rejected as the primary author-facing contract because it hides dependencies rather than removing them. Reflection may still be considered internally at a runtime composition boundary if necessary, but it is not the public Backend programming model.

## Open questions

- What is the exact generic Backend interface?
- What non-generic runtime bridge allows Engine to invoke `IBackend<TTargetModel>` without knowing `TTargetModel`?
- What is the minimum common metadata every Target Abstractions generation must expose?
- How does Engine discover Target contract metadata without eagerly activating a plugin?
- How are incompatible Domain and Target contract generations isolated in the initial in-process plugin model?
- Which validation belongs in Target conformance versus the concrete Target runtime?