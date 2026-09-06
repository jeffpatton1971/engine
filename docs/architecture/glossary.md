# Engine Glossary

> Status: Working Draft

This glossary defines the current canonical terminology for the Engine architecture exploration.

The terms below are intentionally provisional. They exist to make architecture discussions precise while the design is still being challenged and refined.

## Intent

A declarative description of desired infrastructure state or capability, independent of a specific deployment representation.

Intent describes what infrastructure should exist, not how a particular Target expresses it.

## Source

An external representation containing infrastructure Intent.

Examples include YAML, JSON, API payloads, spreadsheets, or another system's data model.

## Adapter

A component that converts a supported Source representation into Engine's Parsed Intent representation.

Adapters parse and normalize input. They do not apply infrastructure-domain semantics, construct the canonical Resource Graph, or generate deployment artifacts.

## Infrastructure Integration

The independently owned extension boundary for one infrastructure domain.

An Integration owns its Semantic Model, Domain Abstractions, resource materialization and domain-local validation behavior, domain identity and relationship semantics, and the Backends that map its domain into explicitly supported Targets.

The initial extension model is an in-process .NET plugin implementing published Engine contracts.

## Domain Abstractions

The stable, separately consumable strongly typed resource and domain contracts shared between an Integration and its Backends.

Domain Abstractions compose or implement common Engine resource contracts, but Engine does not take compile-time dependencies on Integration-specific Domain Abstractions.

A Backend references its Integration's Domain Abstractions so it can consume the concrete typed resources retained in the Resource Graph.

## Semantic Model

The versioned semantic contract an Infrastructure Integration exposes to Engine to give Parsed Intent infrastructure-domain meaning.

A Semantic Model owns domain concepts, resource types, canonical identity rules, properties, constraints, defaults, relationship semantics, prerequisite semantics, and domain validation rules. It is broader than a collection of resource schemas.

Engine defines the semantic operations and lifecycle an Integration must provide, not the Integration's internal modeling technique. An Integration may implement its model using hand-written code, generated definitions, upstream schemas, reflection, metadata, internal libraries, or other suitable mechanisms.

A Semantic Model remains independent of deployment Targets. It may define that a virtual machine uses a network and that this relationship creates a prerequisite, but it does not define the Terraform address, Bicep expression, CloudFormation reference, or other Target representation.

## Semantic Analysis

The multi-phase process that combines Parsed Intent with Integration-owned semantics to produce the deterministic Resource Graph.

Current lifecycle:

1. **Materialization - Integration:** recognize resource types, construct concrete typed resources, canonical identities and typed references, and perform resource-local/domain-local validation.
2. **Identity Registration - Engine:** collect resources and enforce `ResourceIdentity` uniqueness.
3. **Reference Resolution - Engine:** resolve typed references and validate target existence/type.
4. **Semantic Analysis - Integration:** interpret resolved resources/references and declare relationship and prerequisite semantics.
5. **Graph Construction and Validation - Engine:** construct/deduplicate edges, validate graph integrity, detect cycles, and establish deterministic ordering.

Source declaration order has no semantic significance.

## Resource

A concrete Integration-owned strongly typed unit of infrastructure meaning that participates in the Resource Graph through a deliberately small common Engine resource contract.

A Resource is independent of a deployment Target and retains the domain state required by conformant Backends.

## ResourceIdentity

The authoritative graph identity of a Resource.

Conceptually:

```text
IntegrationId + ResourceType + ResourceKey
```

The Integration owns construction and canonicalization of `ResourceKey` according to its domain's identity/scoping semantics. Engine enforces uniqueness of the complete identity and does not manufacture cloud-specific keys.

## ResourceKey

The Integration-owned canonical key portion of a `ResourceIdentity`.

A ResourceKey may encode whatever domain scope is required for uniqueness. Its syntax and canonicalization rules belong to the Integration rather than Engine.

## ResourceReference<TResource>

A strongly typed, identity-based reference from one domain resource to another expected resource type.

A resource reference does not hold a live CLR object pointer, does not by itself define relationship meaning, and does not automatically imply a dependency. The Integration creates the typed reference; Engine resolves it against the complete resource set.

## Relationship

A resolved graph edge expressing infrastructure-domain meaning between resources, such as `uses-network`, `uses-disk`, or `contained-in`.

The Integration / Semantic Model owns relationship semantics. Engine owns construction and storage of the resolved relationship edge in the Resource Graph.

A relationship may imply a dependency, but the concepts are not synonymous.

## Dependency

A resolved structural graph edge expressing a prerequisite or ordering constraint between resources.

A dependency may be derived from a semantic relationship or may exist independently when the platform imposes a genuine prerequisite without a useful enduring relationship. Integrations must not invent semantic relationships solely to encode ordering.

The dependency edge remains structural. Domain explanations or rule identities belong to separate provenance rather than dependency identity.

## Resource Graph

Engine's deterministic graph of concrete Integration-owned typed Resources plus Engine-owned resolved relationships and dependency edges.

Engine owns identity enforcement, reference resolution, graph construction, traversal, cycle detection, graph integrity validation, and deterministic ordering. Integrations own the semantic meaning that causes relationships and dependencies to be derived.

The graph is lossless for supported Backend needs: a Backend must not return to raw or Parsed Intent merely to recover accepted information that should have survived Semantic Analysis.

## Infrastructure IR

Engine's canonical intermediate representation of resolved infrastructure Intent.

Infrastructure IR is the deterministic Resource Graph. It is independent of deployment Targets and is the semantic boundary between Engine analysis and Integration-owned Target lowering.

## Backend

An Integration-owned component that lowers that Integration's resolved typed Resource Graph into one specific published Target contract.

A Backend bridges two strongly typed contracts:

```text
Domain Abstractions -> Backend -> Target Abstractions
```

A Backend does not redefine domain semantics, reinterpret raw Intent, or own Target serialization.

Examples include an Azure-to-Bicep Backend, an SDDC Flex-to-Terraform Backend, or a GCP-to-Terraform Backend.

## Target

A distinct deployment technology represented through a stable published Target contract.

Examples include Terraform, OpenTofu, Bicep, ARM, CloudFormation, and potentially Ansible. Similar syntax or ancestry does not make two technologies the same Target; Terraform and OpenTofu are separate Targets.

A Target owns its Target Abstractions, target model, target-specific validation, emission, supported contract generations, and Backend conformance suites.

## Target Abstractions / Target Contract

The stable, separately consumable public model and compatibility contract that a Backend produces for a specific Target.

A Backend depends on the Target Abstractions generation it supports and does not reference the concrete Target implementation.

## Target Model / Target IR

The Target-shaped representation produced by a Backend and consumed by a Target.

A Target may use a rich IR when appropriate, but Engine does not require every deployment technology to have the same internal IR-plus-Emitter architecture.

## Emitter

A Target-owned responsibility that serializes a Target model into physical deployment artifacts.

An Emitter does not make infrastructure-domain semantic decisions. It is not required to be an independently loadable Engine plugin.

## Contract Generation

An independently addressable generation of a published Engine, Domain, or Target contract.

A published generation is immutable in its required public shape and semantics. Non-breaking package revisions may occur within a generation; breaking required changes require a new generation. Consumers choose when to migrate while the supplying implementation continues to support and conform to the older generation.

## Conformance Suite

A versioned executable test suite used to demonstrate that an implementation or consumer satisfies a published contract generation.

Every published Target provides a Backend conformance suite. Compatibility is demonstrated through conformance evidence rather than inferred from implementation version numbers.

## Artifact

A deterministic physical output produced by Target emission.

Examples include source files, configuration files, manifests, or other Target-native deployment artifacts.

## Artifact Bundle

The complete, versioned result of a compilation.

An Artifact Bundle may include generated artifacts, manifest metadata, diagnostics, source and Target version information, contract-generation information, and optional graph or provenance information.

## Compilation

The deterministic process of transforming infrastructure Intent into an Artifact Bundle.

Compilation includes adaptation, Integration materialization and semantic analysis, Engine graph construction and validation, Backend lowering, Target validation, and Target emission.

## Engine

The runtime/compiler that coordinates Adapters, Integrations, Semantic Analysis, Resource Graph construction, Backends, Targets, diagnostics, and Artifact Bundle production.

Engine owns common orchestration and graph mechanics while remaining independent of infrastructure-domain and Target-specific semantics.

## Cloud-Native

An operating and design posture rather than a requirement to use Kubernetes or microservices.

For this project, cloud-native principles include stateless execution, declarative contracts, immutable and versioned artifacts, portable runtime packaging, structured diagnostics, observability, API and CLI parity, and independently distributable extensions where useful.

## Provider

An intentionally avoided Engine architecture term because it is overloaded across cloud providers, Terraform providers, and service providers.

Use Infrastructure Integration when referring to an infrastructure-domain extension, Semantic Model when referring to that Integration's domain semantics, Backend when referring to domain-to-Target lowering, and the specific external technology name when referring to an external provider.