# ADR-002: Infrastructure IR and Resource Graph

- **Status:** Proposed
- **Date:** 2026-09-04
- **Decision Type:** Architecture exploration

## Context

An infrastructure intent compiler requires a canonical representation after source parsing and semantic analysis but before Target-specific lowering.

Using the source document itself as that representation would couple semantic behavior to external syntax. Using Terraform, Bicep, CloudFormation, or another Target representation would couple Engine to a deployment technology.

The domain naturally contains resources, stable identities, properties, references, relationships, and dependencies. These concepts form a graph rather than merely a hierarchical document.

Intent alone is a declaration of desired infrastructure. It does not, by itself, define the complete meaning of the infrastructure concepts it references. Domain meaning is supplied separately by an Integration-owned Semantic Model and applied during Semantic Analysis.

## Proposed decision

Introduce an **Infrastructure Intermediate Representation (Infrastructure IR)** as the canonical resolved representation of infrastructure Intent.

The Infrastructure IR is represented as a deterministic **Resource Graph**.

Conceptually:

```text
Parsed Intent
    |
    +---- Semantic Model
    |       - domain concepts
    |       - resource types
    |       - identities
    |       - properties
    |       - constraints
    |       - defaults
    |       - relationships
    |       - semantic rules
    |
    v
Semantic Analysis
    |
    v
Infrastructure IR / Resource Graph
    |
    +-- concrete Integration-owned typed resources
    +-- stable identities
    +-- resolved typed references
    +-- semantic relationship edges
    +-- dependency edges
    +-- deterministic ordering
    +-- diagnostics/provenance as appropriate
```

A Resource is a typed unit of infrastructure meaning. Its fundamental identity SHALL NOT depend on a Target-specific construct such as a Terraform resource address, Bicep symbolic name, CloudFormation logical ID, or output file path.

Dependencies and relationships SHALL be explicit semantic information in the graph and SHALL NOT depend on source document nesting, collection order, output file structure, or emitter behavior.

## Resource shape

The Resource Graph SHALL contain the concrete Integration-owned typed resource instances produced by semantic analysis.

Engine SHALL interact with those resources through a deliberately small common contract. The current minimum candidate is:

```csharp
public interface IResource
{
    ResourceIdentity Identity { get; }
    ResourceType Type { get; }
}
```

This example remains illustrative rather than an accepted API contract.

The narrow Engine contract does not flatten or discard domain state. A concrete resource may contain all strongly typed domain properties required by its Integration and Backends:

```csharp
public sealed record VirtualMachineResource : IResource
{
    public required ResourceIdentity Identity { get; init; }
    public ResourceType Type => SddcFlexResourceTypes.VirtualMachine;

    public required int CpuCount { get; init; }
    public required MemorySize Memory { get; init; }
    public required ResourceReference<NetworkResource> Network { get; init; }
}
```

Engine may see this object as `IResource`; an SDDC Flex Backend that references `SddcFlex.Abstractions` may consume it as `VirtualMachineResource`.

Domain properties therefore belong to Integration-owned Domain Abstractions rather than a generic Engine property bag.

See ADR-008 for the Domain Abstractions and typed-resource decision.

## Lossless IR

Infrastructure IR SHALL preserve all resolved information required by a conformant Backend to lower supported domain resources into a Target contract.

A Backend SHALL NOT need to return to raw or parsed Intent merely to recover information that was accepted before or during semantic analysis.

The Resource Graph is therefore Target-independent, but it is not information-poor.

## Graph ownership

Engine owns graph structure and resolution semantics. Integrations own the concrete typed resource payloads that participate in the graph.

Semantic relationships and dependency edges belong to the Resource Graph rather than being required state maintained independently by every resource object.

An illustrative graph shape is:

```csharp
public interface IResourceGraph
{
    IReadOnlyCollection<IResource> Resources { get; }
    IReadOnlyCollection<ResourceRelationship> Relationships { get; }
    IReadOnlyCollection<ResourceDependency> Dependencies { get; }
}
```

This is illustrative rather than an accepted API contract.

A **relationship** expresses domain meaning, for example:

```text
VirtualMachine:web01 --uses-network--> Network:application
```

A **dependency** expresses graph or ordering information derived from resolved semantics.

Engine owns reference resolution, relationship construction, dependency analysis, cycle detection, deterministic traversal, and deterministic topological ordering.

Domain resource objects MAY contain typed references such as `ResourceReference<NetworkResource>`, but SHOULD NOT embed live referenced resource objects merely to create graph connectivity. Engine resolves typed references into graph edges and provides graph APIs for Backends to navigate those resolved relationships.

## Semantic Models

A Semantic Model is the authoritative definition of an infrastructure domain's concepts, resource types, identities, properties, constraints, defaults, relationships, and semantic rules used to interpret and validate infrastructure Intent.

A Semantic Model is deliberately broader than a collection of resource schemas. Resource shape is one part of the model, but the model also provides the domain knowledge necessary to determine what an expression of Intent means.

For a resource type, that may include:

- which properties exist and what they mean;
- property types and constraints;
- required and optional properties;
- identity rules;
- defaulting rules;
- valid references and relationships;
- semantic validation rules;
- other domain-specific behavior required to resolve Intent.

Semantic Models SHALL remain independent of deployment Targets. A Semantic Model may know that a virtual machine requires or references a network, for example, but it SHALL NOT define the Terraform resource address, Bicep symbolic expression, CloudFormation reference, file placement, or other Target-specific representation of that relationship.

Those decisions belong to a Backend after Semantic Analysis has produced Infrastructure IR.

An Integration may construct its Semantic Model using declarative definitions, executable code, generated definitions, upstream metadata, or a combination. Engine defines the contract the model exposes rather than prescribing its internal implementation technique.

## Intent, semantics, and lowering

The architecture deliberately separates three questions:

```text
Intent
"What infrastructure is desired?"
        |
        v
Semantic Model + Semantic Analysis
"What does that infrastructure mean in this domain?"
        |
        v
Infrastructure IR / Resource Graph
"What has been resolved?"
        |
        v
Backend
"How is that meaning represented by this deployment Target?"
        |
        v
Target model / Target IR
```

This separation allows the same resolved infrastructure semantics to be lowered into different deployment Targets without changing the domain definition.

## Rationale

A graph is a natural representation because infrastructure resources reference and depend on one another independently of their textual arrangement.

Treating that graph as an IR establishes a compiler boundary: semantic analysis produces resolved infrastructure meaning, and Backends consume that meaning without needing to reinterpret the source document.

Separating the Semantic Model from both Intent and Target lowering also prevents resource definitions from becoming accidental deployment-language models.

Keeping Engine's resource participation contract small while retaining concrete typed resources in the graph avoids both a weakly typed generic property bag and Engine dependencies on Integration-specific resource types.

## Consequences

### Positive

- Engine has a Target-independent canonical representation.
- Intent remains a declaration rather than carrying all domain behavior itself.
- Semantic Models provide an explicit home for infrastructure-domain meaning.
- Dependency analysis becomes an explicit Engine-owned semantic phase.
- Backends consume resolved strongly typed resources rather than raw source input.
- The graph can support explainability and visualization independent of artifact generation.
- Stable identities can be defined once and mapped into multiple Target representations.
- A single Semantic Model and Domain Abstractions generation can support multiple Backends.
- Engine can keep a small, stable resource contract without discarding domain properties.

### Negative / risks

- Typed domain resources complicate Resource Graph serialization and persistence compared with a generic property bag.
- Plugin loading must preserve CLR type identity between Integrations and Backends sharing Domain Abstractions.
- Resource-reference and graph-navigation APIs require careful design.
- Multiple Domain Abstractions generations may require deliberate assembly isolation.

## Guardrails

- Infrastructure IR is **common in structure, not necessarily common in resource semantics**.
- Engine SHALL NOT define a universal cloud resource taxonomy merely to normalize unrelated infrastructure domains.
- Engine SHALL NOT depend at compile time on Integration-specific resource types.
- The Resource Graph SHALL preserve concrete Integration-owned typed resources rather than flattening them into generic property bags solely for Engine convenience.
- The Infrastructure IR SHALL be lossless for supported Backend needs.
- Semantic relationships and dependency edges SHALL be explicit Engine-owned graph information.
- Semantic Models and Domain Abstractions SHALL NOT contain Target-specific lowering or emission behavior.

## Relationship to earlier dependency-graph experience

A deterministic dependency graph is not a new requirement. Existing compiler work has already demonstrated useful behavior around explicit resource dependencies, missing-dependency diagnostics, cycle detection, and deterministic topological ordering.

Engine should retain those proven graph behaviors while improving the ownership model: unresolved dependency declarations are interpreted during semantic analysis, and the canonical Resource Graph contains resolved identity-based dependency edges rather than requiring each resource to expose string dependency names.

The Resource Graph is broader than a dependency sorter. It also contains the concrete typed resources and semantic relationship edges that Backends need for lowering.

## Alternatives considered

### Source document as canonical model

Use the parsed source representation as the primary semantic representation.

Not preferred because source syntax and infrastructure meaning have different responsibilities and because multiple source representations should be able to express equivalent Intent.

### Target representation as canonical model

Use Terraform or another deployment representation as the canonical IR.

Rejected because it moves Target mechanics into the semantic center of Engine.

### Universal infrastructure resource model

Define one normalized VM, network, storage, and related model across every infrastructure domain.

Rejected as the default architecture because it risks erasing meaningful domain semantics and producing a lowest-common-denominator abstraction.

### Generic property bag Resource Graph

Represent every resource as Engine-owned properties such as `Dictionary<string, object>`.

Rejected as the primary model because it discards compile-time domain typing and pushes structural failures into Backend runtime behavior.

## Open questions

- What is the exact `IResource` contract beyond `Identity` and `Type`, if anything?
- What is the exact `IResourceGraph` lookup, resolution, traversal, and ordering API?
- What is the exact shape of `ResourceIdentity` and `ResourceType`?
- What is the shape of a typed `ResourceReference<TResource>`?
- Does the canonical graph contain only successfully resolved resources, or can compilation expose a partial graph for diagnostics?
- How are relationships spanning multiple Integrations represented, if such relationships are supported?
- Which provenance and metadata belong in the graph versus the compilation request or Artifact Bundle?
- How is the typed Resource Graph serialized without weakening the runtime type model?