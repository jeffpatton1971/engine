# ADR-002: Infrastructure IR and Resource Graph

- **Status:** Proposed
- **Date:** 2026-09-04
- **Decision Type:** Architecture exploration

## Context

An infrastructure intent compiler requires a canonical representation after source parsing and semantic analysis but before target-specific lowering.

Using the source document itself as that representation would couple semantic behavior to external syntax. Using Terraform, Bicep, CloudFormation, or another target representation would couple the Engine to a deployment technology.

The domain naturally contains resources, stable identities, properties, references, relationships, and dependencies. These concepts form a graph rather than merely a hierarchical document.

## Proposed decision

Introduce an **Infrastructure Intermediate Representation (Infrastructure IR)** as the canonical resolved representation of infrastructure intent.

The Infrastructure IR is represented as a deterministic **Resource Graph**.

Conceptually:

```text
Parsed Intent
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

## Semantic models

Resource meaning is supplied by a Semantic Model.

A Semantic Model may define:

- resource types;
- property types and constraints;
- defaults;
- identity rules;
- valid relationships;
- validation rules;
- domain-specific semantic behavior.

The Semantic Model describes infrastructure meaning. It does not generate target artifacts.

Whether semantic models are primarily declarative schemas, executable code, or a combination remains an open decision.

## Rationale

A graph is a natural representation because infrastructure resources reference and depend on one another independently of their textual arrangement.

Treating that graph as an IR establishes a compiler boundary: semantic analysis produces resolved infrastructure meaning, and target backends consume that meaning without needing to reinterpret the source document.

This should make dependency analysis, diagnostics, visualization, explanation, testing, and future target lowering substantially clearer.

## Consequences

### Positive

- The Engine has a target-independent canonical representation.
- Dependency analysis becomes an explicit semantic phase.
- Backends consume resolved resources rather than raw source input.
- The graph can potentially support explainability and visualization independent of artifact generation.
- Stable identities can be defined once and mapped into multiple target representations.

### Negative / risks

- A generic property bag could become weakly typed and push complexity downstream.
- An overly rigid type hierarchy could make third-party semantic models difficult to author.
- The IR could accidentally become a lowest-common-denominator cloud model if resource semantics are over-normalized across domains.

## Guardrails

The Infrastructure IR should be **common in structure, not necessarily common in resource semantics**.

For example, Azure and VCFA resources do not need to be forced into an artificial universal `VirtualMachine` definition merely because both systems expose virtual machines.

The Engine should provide common primitives for identity, typing, properties, relationships, diagnostics, and graph behavior while allowing semantic models to express domain-specific resource meaning.

This distinction should be tested carefully before designing concrete resource APIs.

## Alternatives considered

### BuildSpec as canonical model

Retain a BuildSpec-style document object as the primary representation.

Not preferred because a request/document envelope and a resolved semantic graph have different responsibilities.

### Target representation as canonical model

Use Terraform or another deployment representation as the canonical IR.

Rejected because it moves target mechanics into the semantic center of the Engine.

### Universal cloud resource model

Define one normalized VM, network, storage, and related model across every infrastructure domain.

Not currently proposed. This risks erasing meaningful provider/domain semantics and producing a lowest-common-denominator abstraction.

## Open questions

- Is `IResource` the right extensibility shape?
- How strongly typed should resource properties be?
- Can semantic models introduce their own CLR types while preserving graph interoperability?
- Does the IR contain only successfully resolved resources, or can it represent partially resolved intent for diagnostics?
- How are cross-semantic-model relationships represented?
- Which metadata belongs in the graph versus the compilation request or artifact manifest?