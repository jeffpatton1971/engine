# ADR-002: Infrastructure IR and Resource Graph

- **Status:** Proposed
- **Date:** 2026-09-04
- **Decision Type:** Architecture exploration

## Context

An infrastructure intent compiler requires a canonical representation after source parsing and semantic analysis but before target-specific lowering.

Using the source document itself as that representation would couple semantic behavior to external syntax. Using Terraform, Bicep, CloudFormation, or another target representation would couple the Engine to a deployment technology.

The domain naturally contains resources, stable identities, properties, references, relationships, and dependencies. These concepts form a graph rather than merely a hierarchical document.

Intent alone is a declaration of desired infrastructure. It does not, by itself, define the complete meaning of the infrastructure concepts it references. Domain meaning is supplied separately by a Semantic Model and applied during Semantic Analysis.

## Proposed decision

Introduce an **Infrastructure Intermediate Representation (Infrastructure IR)** as the canonical resolved representation of infrastructure intent.

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
Infrastructure IR
    |
    +-- Resources
    +-- Stable identities
    +-- Typed properties
    +-- Resolved references
    +-- Semantic relationships
    +-- Dependency edges
    +-- Metadata
    +-- Diagnostics/provenance as appropriate
```

A Resource is a typed unit of infrastructure meaning. Its fundamental identity SHALL NOT depend on a target-specific construct such as a Terraform resource address, Bicep symbolic name, CloudFormation logical ID, or output file path.

Dependencies and relationships SHALL be explicit semantic information in the graph and SHALL NOT depend on source document nesting, collection order, output file structure, or emitter behavior.

## Resource shape

The exact implementation is intentionally undecided, but the conceptual shape is approximately:

```csharp
public interface IResource
{
    ResourceIdentity Identity { get; }
    ResourceType Type { get; }
    IReadOnlyDictionary<string, Value> Properties { get; }
    IReadOnlyCollection<ResourceRelationship> Relationships { get; }
}
```

This example is illustrative rather than an accepted API contract.

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

Semantic Models SHALL remain independent of deployment targets. A Semantic Model may know that a virtual machine requires or references a network, for example, but it SHALL NOT define the Terraform resource address, Bicep symbolic expression, CloudFormation reference, file placement, or other target-specific representation of that relationship.

Those decisions belong to a Backend after Semantic Analysis has produced Infrastructure IR.

Whether Semantic Models are primarily declarative definitions, executable code, or a combination remains an open decision. A likely direction is declarative definitions for data and structure with executable rules available where domain semantics require behavior, but this is not yet accepted.

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
"How is that meaning represented by this deployment target?"
        |
        v
Target IR
```

This separation allows the same resolved infrastructure semantics to be lowered into different deployment targets without changing the domain definition.

## Rationale

A graph is a natural representation because infrastructure resources reference and depend on one another independently of their textual arrangement.

Treating that graph as an IR establishes a compiler boundary: semantic analysis produces resolved infrastructure meaning, and target backends consume that meaning without needing to reinterpret the source document.

Separating the Semantic Model from both Intent and target lowering also prevents resource definitions from becoming accidental deployment-language models.

This should make dependency analysis, diagnostics, visualization, explanation, testing, and future target lowering substantially clearer.

## Consequences

### Positive

- The Engine has a target-independent canonical representation.
- Intent remains a declaration rather than carrying all domain behavior itself.
- Semantic Models provide an explicit home for infrastructure-domain meaning.
- Dependency analysis becomes an explicit semantic phase.
- Backends consume resolved resources rather than raw source input.
- The graph can potentially support explainability and visualization independent of artifact generation.
- Stable identities can be defined once and mapped into multiple target representations.
- A single Semantic Model can potentially support multiple deployment backends.

### Negative / risks

- A generic property bag could become weakly typed and push complexity downstream.
- An overly rigid type hierarchy could make third-party Semantic Models difficult to author.
- The IR could accidentally become a lowest-common-denominator infrastructure model if resource semantics are over-normalized across domains.
- A Semantic Model could become an overly broad abstraction if its responsibilities are not kept distinct from parsing and target lowering.

## Guardrails

The Infrastructure IR should be **common in structure, not necessarily common in resource semantics**.

For example, Azure and VCFA resources do not need to be forced into an artificial universal `VirtualMachine` definition merely because both systems expose virtual machines.

The Engine should provide common primitives for identity, typing, properties, relationships, diagnostics, and graph behavior while allowing Semantic Models to express domain-specific resource meaning.

Semantic Models SHALL NOT contain target-specific lowering or emission behavior.

This distinction should be tested carefully before designing concrete resource APIs.

## Alternatives considered

### Source document as canonical model

Use the parsed source representation as the primary semantic representation.

Not preferred because source syntax and infrastructure meaning have different responsibilities and because multiple source representations should be able to express equivalent Intent.

### Target representation as canonical model

Use Terraform or another deployment representation as the canonical IR.

Rejected because it moves target mechanics into the semantic center of the Engine.

### Universal infrastructure resource model

Define one normalized VM, network, storage, and related model across every infrastructure domain.

Not currently proposed. This risks erasing meaningful domain semantics and producing a lowest-common-denominator abstraction.

## Open questions

- Is `IResource` the right extensibility shape?
- How strongly typed should resource properties be?
- Can Semantic Models introduce their own CLR types while preserving graph interoperability?
- Which Semantic Model capabilities should be declarative definitions versus executable rules?
- Does the IR contain only successfully resolved resources, or can it represent partially resolved Intent for diagnostics?
- How are cross-Semantic-Model relationships represented?
- Which metadata belongs in the graph versus the compilation request or artifact manifest?