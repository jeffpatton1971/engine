# ADR-009: Resource Identity, References, and Graph Edge Semantics

- **Status:** Proposed
- **Date:** 2026-09-05
- **Decision Type:** Architecture exploration

## Context

ADR-008 establishes that the Infrastructure IR is a Resource Graph containing concrete Integration-owned typed resources while Engine owns graph structure and resolution semantics.

The graph therefore needs a stable way to identify resources, express typed references between domain resources, distinguish semantic relationships from execution dependencies, and preserve deterministic graph behavior without forcing Engine to understand cloud-specific scoping rules.

BAT vNext already proves several useful graph behaviors through its dependency graph: deterministic topological ordering, missing-dependency detection, cycle detection, and dependency traversal. The new architecture should evolve those proven behaviors while replacing string-name dependencies with resolved identities and explicit graph edges.

A particular design risk is allowing execution-order requirements to distort the domain model. Earlier designs can accumulate synthetic container resources or artificial relationships whose only purpose is to make ordering work. This ADR explicitly separates semantic meaning from prerequisite ordering to discourage that pattern.

## Proposed decision

The minimum graph identity model is based on three values:

```csharp
public readonly record struct ResourceIdentity(
    IntegrationId Integration,
    ResourceType Type,
    ResourceKey Key);
```

The exact API shape remains illustrative, but the semantics are accepted for this exploration.

### Integration ownership of ResourceKey

The Integration author owns construction of the canonical `ResourceKey` for resources in that infrastructure domain.

Engine SHALL NOT understand or manufacture cloud-specific scope such as Azure subscriptions/resource groups, Kubernetes namespaces, GCP projects/zones, or SDDC Flex organizational containers.

If domain-specific scope is required to make a key unique, the Integration incorporates that scope into the canonical `ResourceKey`.

Examples may conceptually resemble:

```text
azure.virtual-machine.production/web-01
sddcflex.virtual-machine.customer-a/vapp-01/web-01
kubernetes.deployment.production/web-api
```

The specific key syntax belongs to the Integration contract rather than Engine.

Engine treats the tuple:

```text
Integration + ResourceType + ResourceKey
```

as the authoritative graph identity and SHALL enforce uniqueness within the Resource Graph.

Engine SHALL reject duplicate identities deterministically rather than attempting to repair, rename, or infer a better key.

## Resource references

Domain Abstractions MAY use strongly typed identity-based references between resources.

The preferred minimum shape is conceptually:

```csharp
public readonly record struct ResourceReference<TResource>(
    ResourceIdentity Identity)
    where TResource : IResource;
```

A `ResourceReference<TResource>` identifies the expected target resource type and target identity. It SHALL NOT hold a live object pointer to the target resource.

For example:

```csharp
public sealed record VirtualMachineResource : IResource
{
    public required ResourceIdentity Identity { get; init; }
    public required ResourceReference<NetworkResource> Network { get; init; }
}
```

This preserves compile-time type safety without turning Integration-owned resource objects into a cyclic CLR object graph.

Engine resolves the reference against the Resource Graph. Resolution failure, missing resources, duplicate identities, or incompatible target types SHALL produce deterministic diagnostics.

A Backend may use graph APIs to resolve the typed reference into the concrete Integration-owned resource instance.

Conceptually:

```csharp
NetworkResource network = graph.Resolve(vm.Network);
```

The exact lookup API remains an open design question.

## Relationships versus dependencies

Relationships and dependencies are distinct graph concepts and SHALL NOT be treated as synonyms.

### Relationship

A **relationship** expresses domain meaning between resources.

Examples:

```text
VirtualMachine --uses-network--> Network
VirtualMachine --uses-disk-----> VirtualDisk
VirtualMachine --contained-in--> ResourceGroup
Service -------exposes---------> Endpoint
```

Relationships answer questions such as:

> What does this resource use, contain, connect to, expose, belong to, or otherwise relate to in the infrastructure domain?

Relationship semantics are defined by the Integration / Semantic Model.

A typed `ResourceReference<TResource>` identifies the referenced resource but does not by itself define the semantic meaning of the relationship. The Semantic Model supplies that meaning.

### Dependency

A **dependency** expresses ordering or prerequisite behavior.

Examples:

```text
VirtualDisk  --> VirtualMachine
Network      --> VirtualMachine
ResourceGroup --> VirtualMachine
```

These edges mean the prerequisite resource must be available before the dependent resource can be correctly lowered, emitted, or ultimately provisioned according to the domain semantics being represented.

A dependency may be derived from a semantic relationship, but the two are not equivalent.

For example:

```text
VirtualMachine --uses-disk--> VirtualDisk
```

may imply:

```text
VirtualDisk --> VirtualMachine
```

because the disk must exist before the VM can use it.

Similarly:

```text
VirtualMachine --contained-in--> ResourceGroup
```

may imply:

```text
ResourceGroup --> VirtualMachine
```

because the resource group must exist before the VM can be deployed into it.

However, some semantic relationships may not imply ordering. A peer relationship between two resources, for example, may be meaningful without establishing that one must precede the other.

Dependencies MAY also exist without a corresponding semantic relationship when the infrastructure platform imposes a genuine prerequisite or ordering rule that has no useful enduring domain relationship. Integrations SHALL NOT fabricate a semantic relationship solely to justify an ordering edge.

Therefore:

- Relationships express **meaning**.
- Dependencies express **ordering/prerequisite constraints**.
- Semantic Models determine whether and how relationships produce dependency edges.
- Dependencies may exist independently when the domain has a real prerequisite without a meaningful semantic relationship.
- Engine owns the resolved graph edges and deterministic dependency analysis once those semantics have been supplied.

## Avoid synthetic containers and artificial semantics

The Resource Graph SHALL model resources and relationships that are meaningful in the infrastructure domain. It SHOULD NOT introduce synthetic container resources merely to provide grouping, traversal, or deployment ordering when those concerns can be represented directly through graph metadata or dependency edges.

Likewise, an Integration SHALL NOT invent semantic relationships whose only purpose is to make topological ordering work.

A platform-native container is valid when it is a real resource or domain concept. For example, an Azure Resource Group is a meaningful Azure resource and `VirtualMachine --contained-in--> ResourceGroup` is a valid semantic relationship. A made-up `VirtualMachineContainer` that exists only so Engine can order VMs is not.

The design principle is:

> Model real infrastructure meaning as resources and relationships; model prerequisite ordering as dependencies. Do not create fake domain structure to compensate for graph mechanics.

This separation is intended to prevent the Resource Graph from accumulating generic container abstractions that obscure the underlying infrastructure model.

## Graph ownership

Engine owns:

- authoritative graph identity enforcement;
- reference resolution;
- resolved semantic relationship edges;
- resolved dependency edges;
- deterministic traversal;
- missing-reference diagnostics;
- cycle detection;
- deterministic topological ordering.

Integrations own:

- domain resource types;
- canonical `ResourceKey` construction;
- typed resource reference properties;
- semantic relationship definitions;
- rules determining whether a relationship implies dependency behavior;
- domain-specific prerequisite rules that create dependency-only edges where justified.

Backends consume the resolved graph. They SHALL NOT recreate identity resolution or reinterpret raw Intent merely to discover relationships or dependencies that Semantic Analysis should already have resolved.

## Evolution from BAT vNext dependency graph

BAT vNext currently models dependency identity primarily through resource names and string-returning dependency declarations.

Conceptually:

```text
IResource.Name
IResource.GetDependencies() -> string names
```

The new Resource Graph evolves that approach to explicit identity and resolved edges:

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

The exact edge APIs are not yet accepted, but the architectural direction is to preserve the proven deterministic graph algorithms while strengthening identity and semantic resolution.

## Guardrails

- Adapter identity SHALL NOT participate in `ResourceIdentity`; equivalent Intent from different Adapters must resolve to the same domain identity.
- Target-specific addresses SHALL NOT participate in `ResourceIdentity`.
- Engine SHALL NOT encode infrastructure-domain scope as dedicated fields solely for one cloud or platform.
- Integration authors SHALL provide canonical ResourceKeys suitable for uniqueness in their domain.
- Engine SHALL enforce uniqueness of the complete `ResourceIdentity` tuple.
- `ResourceReference<TResource>` SHALL remain identity-based rather than object-pointer-based.
- A resource reference SHALL NOT automatically imply an execution dependency.
- Relationships and dependencies SHALL remain independently represented graph concepts.
- Dependencies MAY exist without a semantic relationship when a genuine prerequisite exists.
- Integrations SHALL NOT invent semantic relationships solely to encode ordering.
- Synthetic container resources SHOULD NOT be introduced solely for grouping, traversal, or dependency ordering.
- Platform-native containers remain valid resources when they are genuine domain concepts.
- Backends SHALL consume resolved graph semantics rather than re-resolving source-level references.

## Consequences

### Positive

- Graph identity is stronger than name-only identity without making Engine cloud-aware.
- Integrations retain ownership of real domain scoping rules.
- Typed references provide Backend and Integration developers with compile-time safety.
- Engine can resolve and diagnose references deterministically.
- Semantic relationships remain expressive without forcing every relationship into an ordering constraint.
- Dependency-only edges can represent genuine platform prerequisites without polluting domain semantics.
- Synthetic container abstractions are discouraged unless they represent real infrastructure concepts.
- Proven BAT vNext topological-sort and cycle-detection behavior can be carried forward into the richer graph model.

### Negative / risks

- Integration authors must design canonical ResourceKeys carefully.
- Poor ResourceKey design can create collisions or unstable identity even when Engine behaves correctly.
- Relationship-to-dependency derivation needs a precise Semantic Model contract.
- Dependency-only edges require disciplined use so they do not become an escape hatch for poorly modeled semantics.
- Typed references and multiple Domain Abstractions generations increase assembly/type-identity considerations for the plugin loader.

## Open questions

- What is the exact `IResourceGraph` lookup and traversal API?
- What are the concrete shapes of `ResourceRelationship` and `ResourceDependency`?
- How does the Semantic Model declare relationship kinds and dependency behavior?
- What criteria distinguish a justified dependency-only edge from a missing semantic relationship?
- Should graph identity comparison be strictly ordinal and case-sensitive by Engine contract, or can an Integration define canonicalization before identity creation?
- What diagnostics are required for duplicate identities, unresolved references, wrong-type references, and dependency cycles?
- How are graph identities and edges serialized into diagnostics, provenance, and Artifact Bundles?