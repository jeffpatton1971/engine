# ADR-009: Resource Identity, References, and Graph Edge Semantics

- **Status:** Proposed
- **Date:** 2026-09-05
- **Decision Type:** Architecture exploration

## Context

ADR-008 establishes a Resource Graph containing Integration-owned typed semantic nodes while Engine owns graph structure and resolution mechanics.

The graph needs stable identity, typed references, semantic relationships, prerequisite dependencies, mixed managed/existing lifecycle participation, deterministic traversal, and provisioning ordering without forcing Engine to understand infrastructure-domain scoping rules.

## Decision

### Resource identity

The minimum graph identity semantics are:

```csharp
public readonly record struct ResourceIdentity(
    IntegrationId Integration,
    ResourceType Type,
    ResourceKey Key);
```

The Integration owns construction and canonicalization of `ResourceKey` according to its domain identity/scoping semantics.

Engine SHALL NOT understand or manufacture cloud-specific scopes such as Azure resource groups/VNets/subscriptions, Kubernetes namespaces, GCP projects/zones, or other platform containers.

If scope is required for uniqueness, the Integration encodes it into the canonical key.

For example, two Azure subnets both named `app` may legitimately have identities conceptually equivalent to:

```text
azure | azure.subnet | network-east-rg/east-vnet/app
azure | azure.subnet | network-west-rg/west-vnet/app
```

The exact syntax belongs to Azure Domain Abstractions, not Engine.

Engine treats:

```text
Integration + ResourceType + ResourceKey
```

as authoritative identity and SHALL enforce uniqueness deterministically.

Existing and managed versions of the same infrastructure concept SHALL use the same identity namespace. Lifecycle is not part of `ResourceIdentity`.

### Ambiguous source references

A source value such as `subnet: app` may be insufficient for an Integration to construct a canonical identity when multiple domain scopes are possible.

That ambiguity is an Integration semantic/materialization diagnostic because only the Integration understands the domain's scoping rules.

Once the Integration produces a canonical `ResourceIdentity`, failure to find that identity in the registered graph is an Engine unresolved-reference diagnostic.

Therefore:

> Integration resolves domain naming/scoping into canonical identity; Engine resolves canonical identity into graph nodes.

## Typed references

Domain Abstractions MAY use strongly typed identity-based references.

A typed reference SHALL target a domain semantic contract rather than a lifecycle-specific implementation.

Conceptually:

```csharp
ResourceReference<ISubnet>
```

where `ISubnet` is an Integration-owned domain contract implemented by both managed and existing subnet shapes.

The reference contains canonical identity and SHALL NOT hold a live target object pointer.

It does not by itself define relationship meaning or imply a dependency.

The Integration constructs the reference because it understands the semantic property. Engine resolves it against the complete identity registry and verifies structural/domain-contract consistency through the Integration's published type metadata.

The exact generic constraint and domain-contract-to-`ResourceType` association API remain open; earlier examples constrained `TResource` directly to a managed `IResource` implementation and are superseded by ADR-008's domain-contract/lifecycle split.

## Relationships versus dependencies

A **relationship** expresses domain meaning:

```text
VM --attached-to--> NIC
NIC --attached-to--> Subnet
VM --uses-disk--> ManagedDisk
VM --contained-in--> ResourceGroup
```

A **dependency** expresses a prerequisite or ordering fact:

```text
Subnet -> NIC
NIC -> VM
ManagedDisk -> VM
ResourceGroup -> VM
```

Relationships and dependencies SHALL remain distinct.

A relationship may imply a dependency. A genuine platform prerequisite may also create a dependency without a useful enduring semantic relationship. Integrations SHALL NOT invent relationships merely to encode ordering.

### Dependency direction belongs to domain semantics

Lifecycle type does not determine whether a dependency exists or its direction.

For example:

```text
Existing Subnet -> Managed NIC
```

is valid in the Azure pressure test because the NIC requires the subnet.

The reverse edge is not rejected because it is `Managed -> Existing`; it is absent because Azure subnet semantics do not require the NIC.

The governing rule is:

> Integration determines whether a dependency exists and its direction. Engine validates and operates on the resulting graph; lifecycle affects provisioning behavior, not semantic edge validity.

Engine SHALL NOT reverse, invent, or reject dependency direction solely from lifecycle markers.

## Mixed-lifecycle dependency views

Managed and existing nodes both participate in semantic dependency traversal.

Provisioning order is different: existing prerequisites are already satisfied and are not scheduled for creation.

Conceptually:

```text
Semantic dependency graph
    Existing Subnet -> Managed NIC -> Managed VM

Managed provisioning projection
    Managed NIC -> Managed VM
```

Engine therefore needs to distinguish semantic dependency traversal from the managed provisioning projection without creating separate competing dependency edge types.

Managed-to-managed cycles are ordinary provisioning cycles and SHALL fail graph validation.

Existing-only dependency edges SHOULD NOT be generated merely to reconstruct historical provisioning order. If an Integration emits such an edge because it represents a current semantic prerequisite, Engine may retain it for semantic traversal.

A full mixed-lifecycle semantic cycle may be diagnostically relevant, but Engine SHALL NOT infer domain invalidity from lifecycle direction alone. Any domain-specific semantic-cycle prohibition belongs to Integration semantics; managed provisioning cycles remain Engine graph failures.

## Dependency provenance

`ResourceDependency` remains a structural fact:

```csharp
public readonly record struct ResourceDependency(
    ResourceIdentity Prerequisite,
    ResourceIdentity Dependent);
```

Human-readable reason, rule identity, and source context SHALL NOT participate in structural dependency equality, ordering, or cycle behavior.

Integration semantics own why a dependency exists. Engine MAY retain separate provenance for diagnostics/explainability. Multiple rules may therefore explain one deduplicated structural dependency.

## Avoid synthetic semantics

The graph SHALL model real infrastructure meaning rather than synthetic containers or artificial relationships created merely for ordering.

Platform-native containers remain valid when they are real domain concepts.

> Model real infrastructure meaning as resources and relationships; model prerequisite ordering as dependencies. Do not create fake domain structure to compensate for graph mechanics.

## Graph ownership

Engine owns:

- authoritative identity uniqueness;
- canonical-identity lookup;
- typed-reference resolution mechanics;
- resolved relationship/dependency storage;
- deterministic traversal;
- graph-integrity diagnostics;
- managed provisioning cycle detection;
- deterministic managed-resource ordering;
- provenance transport where required.

Integrations own:

- domain resource contracts;
- canonical ResourceKey construction;
- domain scope interpretation;
- typed reference creation;
- relationship semantics;
- dependency existence/direction;
- domain-specific semantic-cycle rules where applicable;
- domain-facing provenance/explanations.

Backends consume resolved graph semantics and SHALL NOT recreate source-level identity/reference resolution.

## Structural model

Conceptually:

```text
ResourceIdentity
    Integration
    ResourceType
    ResourceKey

ResourceRelationship
    SourceIdentity
    RelationshipType
    TargetIdentity

ResourceDependency
    PrerequisiteIdentity
    DependentIdentity
```

Exact APIs remain open.

## Guardrails

- Adapter identity SHALL NOT participate in ResourceIdentity.
- Target-specific addresses SHALL NOT participate in ResourceIdentity.
- Lifecycle SHALL NOT participate in ResourceIdentity.
- Integration authors SHALL canonicalize whatever domain scope is required for uniqueness.
- Engine SHALL enforce uniqueness of the complete ResourceIdentity tuple.
- Ambiguous domain scope/name resolution belongs to Integration diagnostics.
- Missing canonical identities belong to Engine reference-resolution diagnostics.
- Typed references SHALL target domain contracts rather than lifecycle-specific implementation types.
- Resource references SHALL remain identity-based rather than object-pointer-based.
- A reference SHALL NOT automatically imply a dependency.
- Relationships and dependencies SHALL remain independently represented concepts.
- Integration semantics own dependency existence and direction.
- Engine SHALL NOT infer dependency validity solely from managed/existing lifecycle.
- Existing nodes SHALL participate in semantic dependency traversal but SHALL NOT be scheduled as managed provisioning units.
- Managed provisioning ordering SHALL operate over managed resources while honoring existing prerequisites as satisfied boundaries.
- Integrations SHALL NOT fabricate semantic relationships or containers merely for ordering.
- Dependency provenance SHALL remain separate from structural edge identity.

## Consequences

### Positive

- Scoped duplicate names remain unambiguous without making Engine cloud-aware.
- Existing and managed resources share one identity/reference model.
- Typed references express semantic type rather than lifecycle implementation.
- Brownfield prerequisites remain visible without being scheduled for creation.
- Engine retains deterministic graph mechanics while Integrations retain domain ownership.
- Dependency semantics do not become polluted by lifecycle-specific special cases.

### Risks

- Integration authors must design stable canonical keys carefully.
- The domain-contract-to-ResourceType association requires a precise published contract.
- Semantic dependency traversal and managed provisioning views require clear graph APIs.
- Existing-only semantic dependencies require discipline so historical provisioning data does not pollute current graph meaning.
- Provenance remains a separate future diagnostics contract.

## Open questions

- What is the exact Resource Graph lookup/traversal API?
- How is a domain contract such as `ISubnet` associated with canonical ResourceType metadata?
- Should Engine expose separate APIs for semantic dependency traversal and managed provisioning order, or views over one graph?
- What is the concrete ResourceRelationship type representation?
- Should ResourceKey equality be strictly ordinal after Integration canonicalization?
- What diagnostics are required for duplicates, unresolved references, inconsistent typed references, and managed provisioning cycles?
- What is the separate provenance contract for diagnostics and Artifact Bundles?