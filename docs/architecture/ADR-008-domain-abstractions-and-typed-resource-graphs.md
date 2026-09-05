# ADR-008: Domain Abstractions and Typed Resource Graphs

- **Status:** Proposed
- **Date:** 2026-09-05
- **Decision Type:** Architecture exploration

## Context

Engine must construct a deterministic Infrastructure IR from parsed Intent and an Integration-owned Semantic Model. Integration Backends must then consume that IR with enough domain information and type safety to map infrastructure semantics into a Target contract.

A purely generic property bag such as `Dictionary<string, object>` would keep Engine independent from individual infrastructure domains, but would move many structural errors to runtime and make Backend development unnecessarily weakly typed. At the opposite extreme, making Engine compile directly against every Integration's concrete CLR resource model would couple Engine releases to infrastructure-domain releases and defeat independent extension ownership.

The architecture needs a boundary that permits domain-owned strongly typed resources while allowing Engine to operate through stable common contracts.

## Proposed decision

Each Infrastructure Integration SHOULD publish a separately consumable **Domain Abstractions** contract containing the public resource types and domain contracts shared between the Integration and its Backends.

For example:

```text
SddcFlex.Abstractions
SddcFlex.Integration
SddcFlex.Backend.Terraform
```

Domain Abstractions SHALL implement or compose the common resource and graph contracts published by Engine Abstractions.

Engine SHALL construct and operate on the Resource Graph through Engine-level abstractions. Engine SHALL NOT take a compile-time dependency on a specific Integration's Domain Abstractions.

Integration Backends MAY reference their Integration's Domain Abstractions directly so they can consume resolved domain resources with strong typing.

The Resource Graph SHALL preserve the concrete Integration-owned typed resource instances produced by semantic analysis. Engine's minimal resource contract defines only the surface Engine itself requires; it does not flatten, discard, or replace domain-specific properties.

Conceptually:

```text
                    Engine.Abstractions
                     ^            ^
                     |            |
          SddcFlex.Abstractions   |
              ^          ^        |
              |          |        |
SddcFlex.Integration     SddcFlex.Backend.Terraform
                                  |
                                  v
                       Terraform.Target.Abstractions
```

The dependency arrows represent compile-time contract dependencies. `Intent.Engine` discovers and interacts with Integration resources through Engine abstractions rather than referencing `SddcFlex.Abstractions` directly.

## Ownership

The responsibilities are divided as follows:

| Component | Responsibility |
| --- | --- |
| Adapter | Translate an external source representation into parsed Intent. |
| Domain Abstractions | Define the Integration's public resource types and stable domain contracts shared with Backends. |
| Integration / Semantic Model | Define and apply domain meaning: resource definitions, constraints, defaults, identity rules, relationships, and semantic rules. |
| Engine | Orchestrate semantic analysis, resolve identities and references, construct relationships and dependency edges, and produce the deterministic Resource Graph. |
| Integration Backend | Lower resolved domain resources from the Resource Graph into one Target contract. |
| Target Abstractions | Define the public Target model and compatibility contract. |
| Target | Validate and emit the Target representation. |

The governing distinction is:

> The Integration defines domain meaning; Engine resolves that meaning into a Resource Graph; the Backend lowers those resolved domain resources into a Target contract.

Engine owns the graph structure and resolution semantics. Integrations own the typed resource payloads represented within that graph. Backends own the translation from those typed resources into Target-specific models.

## Semantic Model and Resource Graph

The Semantic Model and Resource Graph are related but distinct.

A Semantic Model defines resource semantics. The Resource Graph contains resolved resource instances.

For example, an SDDC Flex Semantic Model may define a virtual machine resource type, its CPU and memory properties, its identity rules, and a required relationship to a network.

Parsed Intent may contain an unresolved value such as:

```text
virtual-machine: web01
network: app
```

During semantic analysis, Engine uses the Semantic Model to determine that `network` represents a relationship, resolve `app` to a valid network resource, validate the relationship, and derive any dependency edge.

The resulting Resource Graph contains the resolved semantic relationship rather than merely preserving the source string.

```text
VirtualMachine:web01
    |
    +-- uses-network --> Network:app
```

## Typed Resource Graph

The Resource Graph SHALL have a common Engine-owned structural contract while containing Integration-owned strongly typed resource instances.

The minimum Engine resource contract SHOULD remain deliberately small. An illustrative shape is:

```csharp
public interface IResource
{
    ResourceIdentity Identity { get; }
    ResourceType Type { get; }
}
```

An Integration's Domain Abstractions may expose a complete strongly typed resource such as:

```csharp
public sealed record VirtualMachineResource : IResource
{
    public required ResourceIdentity Identity { get; init; }
    public ResourceType Type => SddcFlexResourceTypes.VirtualMachine;

    public required string Name { get; init; }
    public required int CpuCount { get; init; }
    public required MemorySize Memory { get; init; }
    public required ResourceReference<NetworkResource> Network { get; init; }
    public required StorageProfile StorageProfile { get; init; }
}
```

These interfaces and types are illustrative rather than accepted API contracts.

Engine may enumerate and reason about the object through `IResource`, while an SDDC Flex Backend can consume the same graph node as `VirtualMachineResource` because it references `SddcFlex.Abstractions`.

The narrow Engine interface does not imply a narrow or lossy graph. The full concrete resource instance remains available to the Backend with all domain properties produced during semantic analysis.

## Lossless Infrastructure IR

The Infrastructure IR SHALL preserve all resolved information required by a conformant Backend to lower the resource into a supported Target.

A Backend SHALL NOT need to return to raw or parsed Intent merely to recover a property that was available before semantic analysis.

For example, if parsed Intent provides CPU, memory, storage profile, and a network reference, and the Semantic Model accepts and resolves those values, the resulting typed resource SHALL preserve the corresponding resolved domain state for Backend consumption.

Conceptually:

```text
Parsed Intent
    cpu = 4
    memory = 8192
    network = application
        |
        v
Semantic Analysis
        |
        v
VirtualMachineResource
    CpuCount = 4
    Memory = 8192 MiB
    Network = typed reference
        |
        v
Resource Graph
        |
        v
SddcFlex.Backend.Terraform
```

The Resource Graph is therefore target-independent, but not information-poor.

## Resource references and graph connectivity

Domain resource objects MAY contain typed resource references, but graph connectivity and resolved edges remain Engine-owned concerns.

For example:

```csharp
public required ResourceReference<NetworkResource> Network { get; init; }
```

is preferable to embedding a live `NetworkResource` object directly inside `VirtualMachineResource`.

Engine resolves that reference into a graph edge:

```text
VirtualMachine:web01
    |
    +-- uses-network --> Network:application
```

This avoids turning domain objects themselves into a cyclic object graph while preserving strong typing for Integration and Backend developers.

A Backend may inspect the typed reference and/or ask the Resource Graph to resolve it to the target resource through Engine-owned graph APIs.

## Graph-owned relationships and dependencies

Semantic relationships and dependency edges SHOULD be represented by the Resource Graph rather than being required members of each resource object.

An illustrative graph contract may resemble:

```csharp
public interface IResourceGraph
{
    IReadOnlyCollection<IResource> Resources { get; }
    IReadOnlyCollection<ResourceRelationship> Relationships { get; }
    IReadOnlyCollection<ResourceDependency> Dependencies { get; }
}
```

This keeps graph state, cycle detection, dependency ordering, traversal, and reference resolution under Engine ownership while allowing resource classes to focus on domain state.

Relationships and dependencies are distinct concepts:

- a **relationship** expresses domain meaning, such as a virtual machine using a network;
- a **dependency** expresses ordering or graph dependency derived from resolved semantics.

## Domain Abstractions and Backend lowering

Domain Abstractions are the stable typed semantic input consumed by Backends.

A Backend is therefore the explicit lowering boundary between a domain model and a Target model:

```text
Domain Abstractions
        |
        v
      Backend
        |
        v
Target Abstractions
```

For example:

```text
SddcFlex.Abstractions
        |
        v
SddcFlex.Backend.Terraform
        |
        v
Terraform.Target.Abstractions
```

The Backend does not define SDDC Flex resource semantics; those belong to the Integration and its Domain Abstractions. The Backend does not define Terraform serialization; that belongs to the Terraform Target. Its responsibility is the translation between the two published contracts.

This makes Backend the formal domain-to-Target lowering stage. It is intentionally analogous to a compiler backend: it consumes a resolved typed domain representation and produces a Target-specific representation.

## Resource Graph ownership

The Adapter does not construct the canonical Resource Graph.

The Adapter owns source translation only:

```text
YAML / JSON / API / other source
        |
        v
      Adapter
        |
        v
   Parsed Intent
```

Engine constructs the canonical graph by combining parsed Intent with Integration semantics:

```text
Parsed Intent --------+
                      |
Semantic Model -------+--> Semantic Analysis --> Resource Graph
```

This permits different source representations to resolve to the same infrastructure meaning and therefore the same deterministic Resource Graph.

## Backend consumption

A Backend is Integration-owned and understands the Integration's Domain Abstractions.

For example:

```text
SddcFlex.Backend.Terraform
    -> Engine.Abstractions
    -> SddcFlex.Abstractions
    -> Terraform.Target.Abstractions
```

A second Backend can consume the same resolved SDDC Flex graph types while mapping them into a different Target:

```text
SddcFlex.Backend.OpenTofu
    -> Engine.Abstractions
    -> SddcFlex.Abstractions
    -> OpenTofu.Target.Abstractions
```

The Domain Abstractions therefore form the stable language shared between an Integration and its Backends.

## Domain contract versioning

Domain Abstractions SHALL follow the same contract-generation principles defined by ADR-007.

A published Domain Abstractions generation is immutable in its required public shape and semantics. Package revisions within a generation MAY fix defects, documentation, tooling, metadata, or other concerns that do not invalidate already-conformant consumers.

Breaking changes require a new independently addressable contract generation.

For example:

```text
SddcFlex.Abstractions.V1
SddcFlex.Abstractions.V2
```

A new resource type does not automatically require a new generation. New domain capability MAY be introduced without changing existing required resource semantics when it can be added without invalidating conformant Backends.

Changing the required shape or meaning of an existing resource contract requires a new generation.

Backends choose which Domain Abstractions generation they consume. The existence of a newer generation does not by itself require a Backend to migrate.

Domain compatibility SHALL be explicit and testable. An Integration implementation claiming support for a Domain Abstractions generation SHALL pass that generation's applicable conformance tests before advertising support, consistent with ADR-007.

## Physical extension model

The architecture distinguishes responsibilities without requiring every responsibility to become an independently loaded plugin.

The principal independently loadable extension types remain:

```text
Adapters
Integrations
Targets
```

A Semantic Model is an Integration responsibility and need not be a separate plugin assembly.

A Backend is Integration-owned and may be packaged with the Integration or separately according to ownership and release needs.

An Emitter is a Target responsibility and need not be an independently loaded plugin merely because it is an architectural concept.

This keeps deployable extension complexity smaller than the conceptual architecture.

## Rationale

The additional contract boundary exists because Integration implementation and Backend implementation are distinct ownership and change concerns that need a stable shared language.

Without Domain Abstractions, Engine would either need to expose resources as weakly typed generic data or depend on Integration implementation types. Neither provides the desired combination of extension independence and compile-time safety.

Domain Abstractions allow:

- Engine to remain infrastructure-domain independent;
- Integration authors to own rich resource contracts;
- Backend authors to work with strongly typed resolved resources;
- multiple Backends to reuse one resolved domain model;
- domain evolution to occur independently from Engine and Target implementations.

The Backend boundary also prevents domain semantics and Target representation from becoming entangled. Domain Abstractions remain Target-independent, while Target Abstractions remain domain-independent.

The intentionally small Engine resource contract provides a stable graph participation contract without reducing the actual resource payload available to Backends.

## Guardrails

- Engine SHALL NOT reference Integration-specific resource types at compile time.
- Domain Abstractions SHALL depend only on stable Engine contracts and domain-neutral dependencies appropriate to the public contract.
- Backends MAY depend on Domain Abstractions and the Target Abstractions they support.
- Domain resource types SHALL NOT contain Target-specific representation concerns.
- Semantic Models define resource semantics; Resource Graphs contain resolved resource instances.
- The Resource Graph SHALL preserve concrete Integration-owned typed resource instances rather than flattening them into generic property bags solely for Engine convenience.
- The Infrastructure IR SHALL preserve resolved information required by supported Backends; Backends SHALL NOT depend on raw Intent for information recovery.
- Graph connectivity and resolved dependency edges SHALL remain Engine-owned concerns.
- Source syntax SHALL NOT leak into the canonical Resource Graph merely because an Adapter supplied it.
- Target concepts SHALL NOT leak into Domain Abstractions.
- Backends SHALL NOT redefine domain semantics owned by the Integration.
- Backends SHALL NOT own Target serialization or emission behavior owned by the Target.
- A contract abstraction SHOULD exist only when it protects an independently owned or independently evolving concern.

## Consequences

### Positive

- Strong typing is preserved for Integration and Backend developers.
- Engine remains independent of infrastructure-domain implementations.
- Multiple Backends can reuse the same resolved domain model.
- Changes have clearer ownership and smaller expected release blast radii.
- Domain contracts can evolve using the same explicit compatibility discipline as Engine and Target contracts.
- The Resource Graph can remain semantically rich without becoming a universal lowest-common-denominator cloud model.
- The Backend has a precise responsibility: lowering from Domain Abstractions to Target Abstractions.
- The Engine resource contract can remain small and stable without sacrificing Backend access to domain properties.
- Typed resource references avoid embedding a cyclic object graph while preserving domain type safety.

### Negative / risks

- Each Integration may introduce an additional public contract package and versioning responsibility.
- Engine plugin loading must preserve CLR type identity between Domain Abstractions consumed by Integrations and Backends.
- Poorly designed Domain Abstractions could become overly broad or change too frequently, recreating dependency cascades at the Integration boundary.
- Typed domain resources complicate serialization and persistence of the Resource Graph compared with a generic property bag.
- Supporting multiple Domain Abstractions generations in one process may require deliberate assembly isolation or compatibility adapters.
- Resource reference and graph-resolution APIs must be designed carefully so Backends can navigate relationships without coupling resources directly to one another.

## Open questions

- What is the exact common Engine resource contract beyond `Identity` and `Type`, if anything?
- What is the exact `IResourceGraph` contract and lookup/resolution API?
- What is the shape of a typed `ResourceReference<TResource>`?
- What common typed value system, if any, belongs in Engine Abstractions?
- How does Engine invoke Integration semantic behavior without depending on Integration implementation types?
- How are typed Resource Graphs serialized into diagnostics, provenance, or Artifact Bundles?
- What constitutes the minimum conformance suite for a Domain Abstractions generation?
- How are multiple Domain Abstractions generations isolated and resolved by the in-process plugin loader?
- Should Backends normally be packaged with their Integration or independently when they have different release cadences?