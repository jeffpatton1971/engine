# ADR-006: Target Contracts and Backend Dependency Model

- **Status:** Proposed
- **Date:** 2026-09-05
- **Decision Type:** Architecture exploration

## Context

Integration Backends lower resolved domain semantics into deployment Targets. Backend authors need strongly typed Domain and Target contracts without depending on concrete Integration or Target implementations.

The Azure pressure tests further established that Backend owns the boundary between a valid domain graph and what a particular Target contract generation can represent, and that compilation-specific context may be required without reopening Parsed Intent.

## Decision

Every Target SHALL publish separately consumable versioned **Target Abstractions** containing the public Target model required by Backends.

A Backend SHALL compile against:

- Engine Abstractions required for Backend participation;
- its Integration's Domain Abstractions generation; and
- the Target Abstractions generation it supports.

A Backend SHALL NOT reference the concrete Integration implementation or concrete Target implementation.

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

## Example package shape

```text
Azure.Abstractions.V1
Azure.Integration
Azure.Backend.Terraform

Terraform.Target.Abstractions.V1
Terraform.Target
Terraform.Target.Conformance.V1
```

`Azure.Backend.Terraform` may reference:

```text
Engine.Abstractions
Azure.Abstractions.V1
Terraform.Target.Abstractions.V1
```

but not:

```text
Azure.Integration
Terraform.Target
```

## Target contract contents

Target Abstractions may contain:

- Target identity;
- contract-generation identity;
- Target model/IR types;
- Target expressions/references;
- public result/diagnostic contracts where required;
- compatibility metadata;
- abstractions required to submit the model to the Target runtime.

They SHALL NOT contain infrastructure-domain semantics, Integration mappings, concrete emitter implementations, credentials, live infrastructure state, or private implementation details.

## Backend contract

Engine SHOULD preserve compile-time Target-model typing for Backend authors while permitting runtime invocation without Engine knowing Target-specific CLR model types.

Conceptually:

```csharp
public interface IBackend<TTargetModel>
{
    TargetRequirement Target { get; }

    CompilationResult<TTargetModel> Lower(
        IResourceGraph input,
        CompilationContext context);
}
```

The exact API and runtime bridge remain open.

The important requirements are:

- input is the resolved semantically-lossless Resource Graph;
- Backend can consume Integration Domain Abstractions strongly typed;
- Backend may receive defined context for the **current Intent/compilation**;
- Backend output comes from Target Abstractions;
- Backend SHALL NOT use raw or Parsed Intent to recover domain meaning;
- Engine need not know domain-specific or Target-specific implementation types.

`CompilationContext` SHALL be per compilation and SHALL NOT be global Engine state. It SHALL NOT become an arbitrary source-data property bag.

## Backend representability validation

Backend knows both sides of the mapping:

```text
Domain Abstractions
        +
Target Abstractions
```

It therefore owns the question:

> Can this valid domain graph be represented by this Target contract generation?

A valid domain feature that cannot be expressed by the selected Target contract is a Backend representability/capability diagnostic, not an Integration domain-validation error and not an Engine graph error.

Target validation occurs after lowering and answers a different question:

> Is the Target model actually produced valid for this deployment technology?

Therefore:

```text
Backend validates representability
Target validates representation
```

Backends SHOULD aggregate independent unsupported-capability diagnostics when safe rather than failing on the first representability issue.

## Existing resources

Managed and existing domain nodes use the same semantic graph/reference model. Backend uses lifecycle plus Target capability to choose Target-native representation.

Target-specific existing-resource mechanisms SHALL remain downstream and SHALL NOT leak into Engine or Domain Abstractions.

If an existing domain node lacks semantic information required by a Target, Backend reports an insufficient-information/unsupported-capability diagnostic unless that information has been supplied before lowering through the defined domain/context contracts.

It SHALL NOT reopen Parsed Intent to recover it.

## Conformance

Target Conformance is normally a development/test dependency.

A Backend claiming compatibility SHALL pass the applicable Target generation's conformance suite.

Backend conformance SHOULD additionally prove the semantic handoff boundary by lowering prepared Resource Graphs using only:

```text
Engine Abstractions
Domain Abstractions
Resource Graph
Compilation Context
Target Abstractions
```

without Adapter or Parsed Intent dependencies available.

Conformance should cover:

- supported domain resources/features lower successfully;
- unsupported features produce stable deterministic diagnostics;
- managed/existing lifecycle is handled according to Target capability;
- required semantic state survives lowering;
- no source recovery dependency exists;
- independent representability failures aggregate when safe.

## Runtime compatibility

Engine discovers Backend and Target independently and verifies:

- Target identity matches;
- Target explicitly supports the contract generation required by Backend;
- loaded Domain Abstractions are compatible with the Backend;
- incompatible combinations fail before lowering.

Implementation versions and contract generations remain separate. Compatibility is explicit and conformance-backed as defined by ADR-007.

## Dependency rules

Permitted:

```text
Domain Abstractions -> Engine.Abstractions
Integration -> Engine.Abstractions
Integration -> Domain Abstractions
Backend -> Engine.Abstractions
Backend -> Domain Abstractions
Backend -> Target Abstractions
Target Implementation -> Target Abstractions
Target Conformance -> Target Abstractions
Engine runtime -> Engine.Abstractions
```

Prohibited:

```text
Backend -X-> Target Implementation
Backend -X-> Integration Implementation
Target Implementation -X-> Integration
Engine Core -X-> Integration-specific resource types
Engine Core -X-> Target-specific model types
Target Abstractions -X-> Integration-specific semantics
Domain Abstractions -X-> Target-specific semantics
Backend -X-> Adapter/Parsed Intent for semantic recovery
```

## Guardrails

- Target Abstractions SHALL be smaller/more stable than Target implementations.
- Domain Abstractions SHALL remain Target-independent.
- Backend authors SHALL NOT reference concrete Integration or Target implementations.
- Backend owns Target representability; Target owns Target-model validation/emission.
- CompilationContext SHALL be scoped to one Intent/compilation.
- Parsed Intent SHALL NOT be available as a Backend semantic recovery mechanism.
- Conformance suites SHALL test public contracts rather than private behavior.
- Contract generations SHALL be independent from implementation versions.
- Engine SHALL validate compatibility before Backend invocation.

## Consequences

### Positive

- Backend developers receive strongly typed SDK-like domain and Target surfaces.
- Engine remains domain/Target independent.
- Target implementation releases need not cascade through Backends while supported contract generations remain conformant.
- Valid domain semantics are cleanly separated from Target capability limitations.
- Backend/source coupling can be prevented structurally through conformance fixtures.
- Per-compilation context supports distinct simultaneous Intent compilations.

### Risks

- Domain/Target contracts create public versioning responsibilities.
- Representability diagnostics need a common diagnostic contract.
- CompilationContext needs discipline to remain narrow and typed.
- Engine still needs a runtime bridge for generic strongly typed Backends.
- Plugin loading must deliberately handle contract generations and CLR type identity.

## Open questions

- What exact generic Backend interface should be published?
- What non-generic runtime bridge invokes it?
- What is the minimum CompilationContext contract?
- How does Backend declare/query representability before or during lowering?
- What common metadata must every Target contract generation expose?
- How does Engine discover compatibility metadata without eager activation?
- Which validation belongs in Target conformance versus Target runtime?