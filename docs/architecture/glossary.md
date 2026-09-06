# Engine Glossary

> Status: Working Draft

This glossary defines the current canonical terminology for the Engine architecture exploration.

## Intent

A declarative description of desired infrastructure state or capability, independent of a specific deployment representation.

## Source

An external representation containing infrastructure Intent, such as YAML, JSON, API payloads, spreadsheets, or another system's data model.

## Adapter

A component that converts a Source representation into Parsed Intent. Adapters parse and normalize input; they do not apply infrastructure-domain semantics, construct the canonical Resource Graph, or generate deployment artifacts.

## Infrastructure Integration

The independently owned extension boundary for one infrastructure domain.

An Integration owns its Semantic Model, Domain Abstractions, materialization/domain validation, canonical identity/scoping rules, relationship/dependency semantics, and Backends.

## Domain Abstractions

Stable, separately consumable strongly typed semantic contracts shared between an Integration and its Backends.

They define domain contracts such as `ISubnet` or `IVirtualMachine`, managed/existing implementations, typed values, and other public domain state. Engine does not compile against Integration-specific Domain Abstractions.

## Semantic Model

The versioned semantic contract an Integration exposes to Engine to give Parsed Intent domain meaning.

It owns resource concepts, identity/scoping rules, properties, constraints, defaults, relationships, prerequisite semantics, and domain validation. It remains independent of deployment Targets.

Engine defines required semantic operations and lifecycle, not a universal infrastructure meta-schema.

## Semantic Analysis

The multi-phase process that combines Parsed Intent with Integration semantics to produce deterministic Infrastructure IR.

Current lifecycle:

1. **Materialization - Integration:** create managed/existing typed domain nodes, canonical identities/references, and domain-local diagnostics.
2. **Identity Registration - Engine:** collect nodes and enforce identity uniqueness.
3. **Reference Resolution - Engine:** resolve canonical typed references.
4. **Semantic Analysis - Integration:** derive relationships, prerequisites, and provenance.
5. **Graph Construction/Validation - Engine:** construct/deduplicate edges, validate integrity, detect managed provisioning cycles, and establish deterministic managed ordering.

Source declaration order has no semantic significance.

## Resource Node

Working common Engine-level graph participant contract containing at least canonical `ResourceIdentity` and `ResourceType`.

The exact public API and final name remain open. Domain semantic type and lifecycle are modeled orthogonally to this common graph participation contract.

## Managed Resource

A domain node implementing the managed lifecycle contract (`IManagedResource` in the current illustrative model).

It represents infrastructure the current compilation intends to manage and may therefore participate in managed provisioning order and Target-managed output.

## Existing Resource

A domain node implementing the existing lifecycle contract (`IExistingResource` in the current illustrative model).

It represents infrastructure asserted to already exist. It participates in identity, references, relationships, semantic dependencies, validation, and Backend lowering but is not scheduled for creation by the current compilation.

Existing implementations should contain only the minimum semantic information required by supported operations.

## Resource Participant

An architectural description for any managed or existing domain node participating in the Resource Graph.

It is not currently required to be a distinct public CLR type; `IResourceNode` plus managed/existing lifecycle contracts may provide the actual API surface.

## ResourceIdentity

The authoritative graph identity:

```text
IntegrationId + ResourceType + ResourceKey
```

Integration owns ResourceKey canonicalization and domain scope interpretation. Engine enforces uniqueness. Lifecycle and Adapter/Target identity do not participate.

## ResourceKey

The Integration-owned canonical key portion of ResourceIdentity. It may encode whatever domain scope is necessary for uniqueness.

## ResourceReference<TDomain>

A strongly typed identity-based reference to a domain semantic contract, such as `ResourceReference<ISubnet>`.

The reference is independent of managed/existing lifecycle and does not contain a live object pointer, define relationship meaning, or automatically imply dependency. Integration creates it; Engine resolves it.

## Relationship

A resolved graph edge expressing domain meaning, such as `attached-to`, `uses-disk`, or `contained-in`.

Integration owns relationship semantics; Engine owns resolved graph representation.

## Dependency

A resolved structural prerequisite edge.

A dependency may derive from a relationship or exist independently when the domain has a genuine prerequisite. Integration owns whether the dependency exists and its direction. Engine does not infer dependency direction from managed/existing lifecycle.

Dependency reason/provenance remains separate from structural edge identity.

## Semantic Dependency Graph

The dependency view containing prerequisite facts across managed and existing nodes. It is useful for traversal, diagnostics, explainability, and Backend context.

## Managed Provisioning Projection

The provisioning-order view derived from the semantic dependency graph in which managed resources are scheduled and existing prerequisites are treated as already satisfied.

Managed-to-managed cycles are provisioning failures. Existing nodes remain semantically visible without becoming scheduled creation units.

## Resource Graph

Engine's deterministic graph of Integration-owned typed managed/existing nodes plus Engine-owned resolved relationship and dependency edges.

Engine owns identity enforcement, reference resolution, graph construction, traversal, graph integrity, managed provisioning cycle detection, and deterministic ordering. Integrations own the semantics that produce graph facts.

## Infrastructure IR

Engine's canonical Target-independent representation of resolved infrastructure Intent: the Resource Graph plus defined compilation-scoped semantic context required by downstream stages.

Infrastructure IR is **semantically lossless** for supported Backend needs. It need not preserve source formatting, ordering, comments, aliases, or other representation-only details.

## Compilation Context

Defined state belonging to one Intent/compilation rather than one resource.

It may include values such as account/subscription selection, location defaults, credential/profile selection, naming context, workflow metadata, or Target selection.

Compilation Context is per compilation, never global Engine state, and must not expose raw Parsed Intent as a downstream escape hatch.

## Backend

An Integration-owned component that lowers resolved domain semantics into one specific Target contract.

```text
Domain Abstractions -> Backend -> Target Abstractions
```

Backend also owns **Target representability**: determining whether a valid domain graph can be expressed by the Target contract generation it supports.

A Backend does not reinterpret raw Intent or own Target serialization.

## Target Representability

The Backend-owned question:

> Can this valid domain graph be represented by this Target contract generation?

Unsupported but valid domain features produce Backend capability/representability diagnostics rather than Integration or Engine graph errors.

## Target

A distinct deployment technology represented through stable published Target contracts. Examples include Terraform, OpenTofu, Bicep, ARM, CloudFormation, and potentially Ansible.

A Target owns Target Abstractions, Target-model validation, emission, supported contract generations, and Backend conformance suites.

## Target Abstractions / Target Contract

The stable public model and compatibility contract produced by a Backend for a specific Target generation.

## Target Model / Target IR

The Target-shaped representation produced by a Backend and consumed by a Target.

## Target Validation

The Target-owned question:

> Is the produced Target model valid for this deployment technology?

This is distinct from Backend representability validation.

## Emitter

A Target-owned responsibility that serializes a valid Target model into physical deployment artifacts. It need not be an independently loadable plugin.

## Contract Generation

An independently addressable generation of a published Engine, Domain, or Target contract. Required public shape and semantics are immutable within a generation; breaking changes require a new generation.

## Conformance Suite

A versioned executable test suite demonstrating compatibility with a contract generation.

Backend conformance should be executable without Adapter or Parsed Intent dependencies so the semantic IR boundary is testable.

## Artifact

A deterministic physical output produced by Target emission.

## Artifact Bundle

The complete versioned result of a compilation, potentially including artifacts, diagnostics, versions/contract generations, source digest, and provenance/graph information.

## Compilation

The deterministic transformation of one Intent plus its per-compilation context into an Artifact Bundle.

Validation ownership is layered:

```text
Integration -> domain correctness
Engine      -> graph correctness
Backend     -> Target representability
Target      -> Target-model correctness
```

## Engine

The runtime/compiler coordinating Adapters, Integrations, Semantic Analysis, Resource Graph construction, Backends, Targets, diagnostics, and Artifact Bundle production while remaining independent of domain and Target-specific semantics.

## Cloud-Native

An operating/design posture favoring stateless execution, declarative contracts, immutable/versioned artifacts, portable runtime packaging, structured diagnostics, observability, API/CLI parity, and independently distributable extensions where useful.

## Provider

An intentionally avoided Engine architecture term because it is overloaded across cloud providers, Terraform providers, and service providers. Use Infrastructure Integration, Semantic Model, Backend, Target, or the specific external technology name as appropriate.