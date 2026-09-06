# ADR-010: Semantic Model and Semantic Analysis Lifecycle

- **Status:** Proposed
- **Date:** 2026-09-05
- **Decision Type:** Architecture exploration

## Context

Engine transforms source-neutral Parsed Intent into a deterministic Infrastructure IR / Resource Graph without understanding Azure, AWS, GCP, VCFA, Kubernetes, or future domain semantics.

Integrations own those semantics and publish Domain Abstractions. Engine defines semantic-analysis operations and graph mechanics rather than a universal infrastructure meta-schema.

The Azure pressure tests validated the multi-phase lifecycle and added important requirements around mixed managed/existing resources, validation aggregation, scoped identity, semantically lossless IR, per-Intent compilation context, and Backend representability validation.

## Decision

Engine defines the **semantic operations and lifecycle** an Integration must provide, not the Integration's internal modeling technique.

An Integration may use hand-written code, generated definitions, reflection, source generators, upstream schemas, provider metadata, internal libraries, or combinations of them.

Initial semantic responsibilities are:

1. recognize an infrastructure resource type;
2. materialize a typed domain participant;
3. construct canonical `ResourceIdentity`;
4. construct typed domain references;
5. validate resource-local/domain-local semantics;
6. describe semantic relationships after references can be resolved;
7. derive prerequisite dependencies and provenance.

The exact public C# API remains deferred until behavior is sufficiently pressure-tested.

## Multi-phase semantic analysis

```text
Parsed Intent
     |
     v
1. Materialization
   Integration / Semantic Model
     |
     +-- recognize resource type
     +-- construct managed or existing typed domain participant
     +-- construct canonical ResourceIdentity
     +-- construct ResourceReference<TDomain>
     +-- validate resource-local/domain-local semantics
     |
     v
2. Identity Registration
   Engine
     |
     +-- collect materialized nodes
     +-- enforce ResourceIdentity uniqueness
     |
     v
3. Reference Resolution
   Engine
     |
     +-- resolve canonical identities
     +-- verify target exists
     +-- verify typed domain-contract consistency
     |
     v
4. Semantic Analysis
   Integration / Semantic Model
     |
     +-- interpret resolved resources/references
     +-- declare semantic relationships
     +-- declare prerequisite dependencies
     +-- provide domain provenance/explanation
     |
     v
5. Graph Construction and Validation
   Engine
     |
     +-- construct relationship edges
     +-- construct/deduplicate dependency edges
     +-- validate graph integrity
     +-- detect managed provisioning cycles
     +-- establish deterministic managed ordering
     |
     v
Infrastructure IR / Resource Graph
```

The phases express ownership and observable behavior, not necessarily one public method per phase.

## Materialization

Integration owns interpretation of domain-specific Intent and construction of concrete Domain Abstraction instances.

A participant may be managed by the current compilation or represent existing infrastructure. Domain semantic type and lifecycle are orthogonal as defined by ADR-008.

The Integration creates typed references because it understands what a field means. It does not resolve those references against the graph during materialization.

Source order SHALL NOT affect materialization identity or eventual graph semantics.

## Identity and reference resolution

Integration owns domain naming and scoping rules and therefore constructs canonical ResourceKeys.

If a source-level name is ambiguous because required domain scope cannot be inferred, Integration reports that semantic/materialization diagnostic.

Engine registers the complete identity set before reference resolution. Once a canonical identity exists, Engine owns lookup and unresolved-reference diagnostics.

Therefore:

> Integration turns domain naming into canonical identity; Engine turns canonical identity into resolved graph connectivity.

Typed references target domain contracts rather than managed/existing implementation classes. A reference to `ISubnet` may resolve to either a managed or existing subnet participant.

## Relationship and dependency derivation

After reference resolution, Integration semantics determine:

- what relationships mean;
- which resolved references imply relationships;
- whether relationships imply dependencies;
- dependency-only prerequisites where justified;
- dependency direction;
- domain provenance/explanation;
- domain-specific semantic-cycle restrictions where applicable.

Engine constructs and deduplicates the structural graph facts and performs deterministic graph analysis.

Lifecycle does not determine dependency direction. Engine SHALL NOT invent or reject an edge solely because its endpoints are managed or existing.

Managed and existing nodes both participate in semantic dependency traversal. Provisioning order is a managed-resource projection in which existing prerequisites are treated as already satisfied.

## Validation ownership

Validation has four distinct ownership layers.

### Integration validation

Answers:

> Is this valid infrastructure-domain intent?

Includes resource-local/domain-local concerns such as required fields, domain types, ranges, enums, cross-field rules, naming/scope ambiguity, and domain invariants that do not require Engine graph mechanics.

### Engine validation

Answers:

> Is this a valid resolved Resource Graph?

Includes identity uniqueness, canonical-reference resolution, typed-reference consistency, relationship/dependency endpoint integrity, managed provisioning cycles, and deterministic graph behavior.

### Backend representability validation

Answers:

> Can this valid domain graph be represented by this Target contract generation?

Backend is the component that knows both Domain Abstractions and Target Abstractions. If a valid domain feature cannot be expressed by the selected Target contract generation, Backend owns the unsupported-capability/representability diagnostic.

This is not an Integration error because the infrastructure semantics are valid, and it is not a Target-model validation error because no valid representation may exist.

### Target validation

Answers:

> Is the produced Target model valid for this deployment technology?

Includes Target expression/model constraints and emission prerequisites. Target validation does not redefine domain semantics.

The governing model is:

```text
Integration validation
    domain correctness

Engine validation
    graph correctness

Backend validation
    Target representability

Target validation
    Target-model correctness
```

## Validation aggregation and fail-fast behavior

Each layer SHOULD report the maximum useful set of independent user-correctable failures that can be discovered safely.

Ordinary validation failures SHOULD use structured diagnostic results rather than exceptions as control flow.

Processing MAY stop for an affected resource/subgraph/phase when continuing would rely on invalid state, produce cascades, or become nondeterministic.

Examples of legitimate early termination include unreadable source structure, inability to determine resource type/identity, ambiguity that prevents canonical identity construction, structural graph invariants that prevent meaningful continuation, or fatal implementation-contract violations.

> Fail fast on unrecoverable structural conditions; aggregate independent user-correctable validation failures whenever safe.

This rule applies to Integration, Engine, Backend representability, and Target-model validation.

## Semantically lossless handoff

Once Semantic Analysis succeeds, Parsed Intent is finished as a source of domain meaning.

The Resource Graph and defined compilation context SHALL preserve every accepted semantic fact required by conformant Backends.

A Backend SHALL NOT reopen raw or Parsed Intent to recover domain properties, relationships, lifecycle, scope, or dependencies.

Infrastructure IR is semantically lossless, not source-representation-lossless. Source formatting, comments, aliases, and ordering need not survive except for diagnostics/provenance.

Backend conformance SHOULD be testable without Adapter or Parsed Intent assemblies available.

## Compilation Context

Some values belong to one compilation rather than one resource. A defined `CompilationContext` may carry such information to downstream stages.

Possible examples include subscription/account selection, location defaults, credential/profile selection, naming context, workflow metadata, or Target selection.

`CompilationContext` SHALL be scoped to one Intent/compilation. Engine SHALL NOT treat compilation-specific context as global state or leak it between concurrent compilations.

Context SHALL NOT expose raw Parsed Intent as an escape hatch around the semantic IR boundary.

## Source-order independence

All participants are materialized and identities registered before graph-level resolution and semantic relationship/dependency analysis are considered complete.

Equivalent Intent expressed through different Adapters or source declaration orders SHOULD produce equivalent semantic resources and graph structure.

Adapters may preserve source locations/order solely for diagnostics/provenance where useful.

## Semantic Model versus universal schema

Engine SHALL NOT require Integrations to encode their complete domain into an Engine-owned universal taxonomy or declarative meta-schema.

Adding a resource type to an Integration SHOULD NOT require Engine Core changes when existing generic semantic and graph contracts remain sufficient.

## Guardrails

- Engine SHALL NOT contain infrastructure-platform-specific materialization logic.
- Integrations SHALL NOT own independent Resource Graph implementations.
- Domain type and managed/existing lifecycle SHALL remain orthogonal.
- Integrations create typed domain references; Engine resolves them.
- Typed references SHALL NOT be lifecycle-specific.
- Integrations own domain scope interpretation and canonical ResourceKey creation.
- Integrations define relationship/dependency semantics and dependency direction.
- Engine SHALL NOT infer dependency validity from lifecycle alone.
- Managed and existing nodes participate in semantic graph analysis; managed provisioning order schedules only managed resources.
- Integration owns domain correctness; Engine owns graph correctness; Backend owns Target representability; Target owns Target-model correctness.
- Independent validation failures SHOULD be aggregated when safe.
- Expected validation failures SHOULD NOT use exceptions as ordinary control flow.
- Parsed Intent SHALL NOT be consulted downstream to recover accepted domain meaning.
- `CompilationContext` SHALL be per Intent/compilation and SHALL NOT become global Engine state.
- Engine SHALL define required semantic behavior before freezing detailed APIs.
- Engine SHALL NOT introduce a universal infrastructure taxonomy merely to simplify Integration implementation.

## Consequences

### Positive

- Domain semantics remain Integration-owned and independently evolvable.
- Engine retains one deterministic graph implementation.
- Forward references and arbitrary source order are naturally supported.
- Brownfield and greenfield resources coexist in one lifecycle.
- Validation failures are attributed to the component that actually owns the rule.
- Valid domain semantics can be distinguished from Target representability limitations.
- Backends cannot legitimately depend on source syntax after Semantic Analysis.
- Per-compilation context supports concurrent compilations without global-state coupling.

### Risks

- Multi-phase orchestration is more complex than a single opaque Analyze call.
- Integration authors must distinguish domain validation from graph validation and Target representability.
- Aggregation requires suppression of cascading diagnostics.
- The eventual Semantic Model API must expose enough resolved context without leaking graph ownership.
- Common diagnostic/provenance contracts must normalize failures across four validation layers.
- CompilationContext needs a deliberately narrow contract to avoid becoming an unstructured property bag.

## Open questions

- What is the minimum public Semantic Model API?
- How does Engine present resolved context to Integration semantic rules during Phase 4?
- Are semantic rules resource-scoped, model-scoped, or both?
- How are relationship kinds represented while remaining Integration-owned?
- What is the common diagnostic contract across Integration, Engine, Backend, and Target?
- How does provenance integrate with Artifact Bundles?
- How should partially failed materialization affect later independent diagnostics?
- What exact contract defines per-compilation `CompilationContext`?
- How does a Backend declare/query representability for a Target generation before or during lowering?
- How should managed provisioning order and semantic dependency traversal be exposed?