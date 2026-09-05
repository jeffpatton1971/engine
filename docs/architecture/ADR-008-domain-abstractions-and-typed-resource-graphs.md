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
| Integration Backend | Map resolved domain resources from the Resource Graph into one Target contract. |
| Target Abstractions | Define the public Target model and compatibility contract. |
| Target | Validate and emit the Target representation. |

The governing distinction is:

> The Integration defines domain meaning; Engine resolves that meaning into a Resource Graph; the Backend understands the resolved domain resources and maps them into a Target.

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

The Resource Graph SHALL have a common Engine-owned structural contract while permitting Integration-owned strongly typed resource representations.

An illustrative Engine contract might resemble:

```csharp
public interface IResource
{
    ResourceIdentity Identity { get; }
    ResourceType Type { get; }
    IReadOnlyCollection<IResourceRelationship> Relationships { get; }
}
```

An Integration's Domain Abstractions may then expose a resource such as:

```csharp
public sealed record VirtualMachineResource : IResource
{
    public required ResourceIdentity Identity { get; init; }
    public required int CpuCount { get; init; }
    public required MemorySize Memory { get; init; }
    public required NetworkReference Network { get; init; }
}
```

These interfaces and types are illustrative rather than accepted API contracts.

Engine can reason about the resource as an `IResource` while an SDDC Flex Backend can consume the same graph resource as a `VirtualMachineResource` through its dependency on `SddcFlex.Abstractions`.

This preserves strong domain typing without teaching Engine what an SDDC Flex virtual machine is.

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

## Guardrails

- Engine SHALL NOT reference Integration-specific resource types at compile time.
- Domain Abstractions SHALL depend only on stable Engine contracts and domain-neutral dependencies appropriate to the public contract.
- Backends MAY depend on Domain Abstractions and the Target Abstractions they support.
- Domain resource types SHALL NOT contain Target-specific representation concerns.
- Semantic Models define resource semantics; Resource Graphs contain resolved resource instances.
- Source syntax SHALL NOT leak into the canonical Resource Graph merely because an Adapter supplied it.
- Target concepts SHALL NOT leak into Domain Abstractions.
- A contract abstraction SHOULD exist only when it protects an independently owned or independently evolving concern.

## Consequences

### Positive

- Strong typing is preserved for Integration and Backend developers.
- Engine remains independent of infrastructure-domain implementations.
- Multiple Backends can reuse the same resolved domain model.
- Changes have clearer ownership and smaller expected release blast radii.
- Domain contracts can evolve using the same explicit compatibility discipline as Engine and Target contracts.
- The Resource Graph can remain semantically rich without becoming a universal lowest-common-denominator cloud model.

### Negative / risks

- Each Integration may introduce an additional public contract package and versioning responsibility.
- Engine plugin loading must preserve CLR type identity between Domain Abstractions consumed by Integrations and Backends.
- Poorly designed Domain Abstractions could become overly broad or change too frequently, recreating dependency cascades at the Integration boundary.
- Typed domain resources complicate serialization and persistence of the Resource Graph compared with a generic property bag.
- Supporting multiple Domain Abstractions generations in one process may require deliberate assembly isolation or compatibility adapters.

## Open questions

- What is the exact common Engine resource contract?
- Does the Resource Graph store concrete Integration-owned resource instances directly, or a normalized Engine-owned representation with typed projections?
- What common typed value system, if any, belongs in Engine Abstractions?
- How does Engine invoke Integration semantic behavior without depending on Integration implementation types?
- How are typed Resource Graphs serialized into diagnostics, provenance, or Artifact Bundles?
- What constitutes the minimum conformance suite for a Domain Abstractions generation?
- How are multiple Domain Abstractions generations isolated and resolved by the in-process plugin loader?
- Should Backends normally be packaged with their Integration or independently when they have different release cadences?