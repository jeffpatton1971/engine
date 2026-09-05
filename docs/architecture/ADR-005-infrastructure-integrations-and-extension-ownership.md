# ADR-005: Infrastructure Integrations and Extension Ownership

- **Status:** Proposed
- **Date:** 2026-09-05
- **Decision Type:** Architecture exploration

## Context

Engine is intended to compile infrastructure Intent without requiring Engine Core to understand every infrastructure domain or deployment technology.

Infrastructure domains evolve independently. A platform may add resource types, capabilities, constraints, or relationships without any corresponding change to the compiler itself. Likewise, deployment Targets such as Terraform, OpenTofu, Bicep, ARM, CloudFormation, or Ansible evolve independently of the infrastructure domains they can represent.

The architecture therefore needs an ownership boundary that allows an infrastructure-domain author to extend Engine without modifying Engine Core and without requiring Engine maintainers to own the lifecycle of that domain.

It also needs stable typed boundaries between an Integration's domain semantics, the Backends that lower those semantics, and the Targets that consume the resulting Target models.

## Proposed decision

Introduce **Infrastructure Integration** as the extension and ownership boundary for an infrastructure domain.

An Infrastructure Integration is an independently developed in-process .NET plugin that implements published Engine integration contracts.

Conceptually:

```text
Engine
  owns compiler orchestration and common graph contracts

Infrastructure Integration
  owns domain semantics
  owns Domain Abstractions
  owns domain-to-Target Backends

Target
  owns Target Abstractions
  owns Target validation and emission
  owns Backend conformance suite
```

An Integration exposes at least:

- stable Integration identity and implementation version;
- supported Engine contract generation information;
- a Semantic Model for its infrastructure domain;
- a Domain Abstractions generation containing public typed resource contracts;
- one or more Backend implementations for explicitly supported Targets;
- compatibility metadata required for composition.

An illustrative contract may resemble:

```csharp
public interface IInfrastructureIntegration
{
    IntegrationIdentity Identity { get; }
    ISemanticModel SemanticModel { get; }
    IReadOnlyCollection<IBackend> Backends { get; }
}
```

This remains illustrative rather than an accepted API contract.

## Integration ownership

Engine defines **what an Integration must expose**, not **how an Integration must implement it**.

An Integration author owns:

- completeness and correctness of the domain Semantic Model;
- supported resource types and domain concepts;
- Domain Abstractions containing the public strongly typed resource contracts shared with Backends;
- domain identities, constraints, defaults, relationships, and semantic rules;
- compatibility with changes to the upstream infrastructure platform;
- domain-to-Target Backend mappings;
- conformance of each Backend with each Target contract generation it claims to support;
- conformance of Integration implementations with supported Domain Abstractions generations;
- the Integration's implementation release lifecycle.

An Integration author may construct its Semantic Model using hand-written code, generated definitions, reflection, source generation, upstream API schemas, provider metadata, internal libraries, or any combination appropriate to the domain.

Engine SHALL NOT prescribe those implementation details.

Adding or changing a resource type in an infrastructure domain SHALL NOT require an Engine Core change solely because the domain evolved when existing generic Engine contracts are sufficient.

## Domain Abstractions

Each Integration SHOULD publish separately consumable Domain Abstractions containing the stable public resource types shared by the Integration and its Backends.

For example:

```text
SddcFlex.Abstractions
        ^            ^
        |            |
SddcFlex.Integration SddcFlex.Backend.Terraform
```

Domain Abstractions compose or implement Engine's common graph/resource contracts, but Engine does not reference Integration-specific Domain Abstractions at compile time.

The Resource Graph preserves the concrete Integration-owned typed resource instances. Engine reasons about them through its common resource contract; the Backend may consume the complete typed resource through its Domain Abstractions dependency.

See ADR-008 for the detailed typed Resource Graph decision.

## .NET plugin model

The initial extension model SHALL be in-process .NET assemblies implementing published Engine abstractions.

The assembly is sufficient as the initial packaging boundary. A separate external plugin manifest is not required unless a concrete requirement emerges that cannot reasonably be satisfied through assembly metadata and published contracts.

The principal independently loadable extension types are:

```text
Adapters
Integrations
Targets
```

A Semantic Model is an Integration responsibility rather than a required independent plugin. A Backend is Integration-owned and may be packaged with the Integration or separately according to release needs. An Emitter is a Target responsibility rather than a required independent plugin.

Discovery and dependency isolation remain separate runtime decisions.

## Targets

A **Target** represents a distinct deployment technology understood through a stable published Target contract.

Potential Targets include Terraform, OpenTofu, Azure Bicep, Azure Resource Manager, AWS CloudFormation, Ansible, and future technologies. This list is illustrative rather than an implementation commitment.

Each distinct deployment technology is modeled as its own Target. Shared syntax, ancestry, or implementation details do not imply a shared Target identity.

Terraform and OpenTofu are therefore separate Targets. If an Integration supports both, it explicitly provides Backend support for both:

```text
SddcFlex Integration
    |
    +-- SddcFlex -> Terraform Backend -> Terraform Target
    |
    +-- SddcFlex -> OpenTofu Backend  -> OpenTofu Target
```

Engine does not define a Terraform/OpenTofu Target family or compatibility profile merely to represent their overlap. Implementations remain free to share internal libraries and tests.

A Target owns, as appropriate to that technology:

- a separately consumable Target Abstractions contract;
- a Target model or Target IR;
- Target-specific types and expressions;
- Target validation;
- serialization and emission;
- one or more Emitters where useful;
- supported Target contract generations;
- versioned Backend conformance suites.

## Backend compatibility

A Backend is an Integration-owned mapping from one Integration's resolved typed domain resources into one specific Target contract.

```text
SddcFlex.Abstractions -> Terraform Backend      -> Terraform.Target.Abstractions
SddcFlex.Abstractions -> OpenTofu Backend       -> OpenTofu.Target.Abstractions
GCP.Abstractions      -> Terraform Backend      -> Terraform.Target.Abstractions
Azure.Abstractions    -> Bicep Backend          -> Bicep.Target.Abstractions
```

A Backend SHALL compile against the published Target Abstractions generation it supports and SHALL NOT reference the concrete Target implementation.

Compatibility is resolved from Target identity plus explicitly supported contract generation, not Target implementation-version ranges.

A Backend supporting multiple Targets must independently support and conform to each distinct Target contract.

## Contract generations and lifecycle

Engine, Domain, and Target contracts follow the contract-generation model in ADR-007.

A published contract generation is immutable in its required public shape and semantics. Breaking changes require a new independently addressable generation. Implementations may support multiple generations simultaneously when they pass each generation's applicable conformance suite.

The contract consumer decides when to migrate to a newer generation. The existence of a newer generation does not itself invalidate a consumer of an older generation.

The contract owner decides how long an older generation remains supported. Retirement may be justified by security, correctness, an unacceptable contract flaw, or an explicit support policy, but availability of V2 or V3 alone is not sufficient reason to force consumer migration.

## Conformance testing

Every published Target SHALL provide a versioned Backend conformance suite. Every Backend SHALL pass the applicable suite before claiming compatibility with that Target contract generation.

Every Domain Abstractions generation SHALL likewise have applicable conformance evidence for Integration implementations that claim to support it.

A newer implementation claiming compatibility with an older generation SHALL run that older generation's suite. Compatibility is demonstrated rather than inferred from implementation version numbers.

Passing Target conformance establishes compliance with the published Target contract. It does not prove every domain-specific mapping is correct; Integration authors remain responsible for their own semantic and mapping tests.

## Target independence and reuse

Targets and Integrations have independent ownership and release lifecycles.

A Target does not contain infrastructure-domain semantics. An Integration does not own common Target implementation behavior.

Multiple Integrations may independently support the same Target, and one Integration may support multiple Targets. Not every Integration is required to support every Target.

A new Target does not create automatic compatibility for existing Integrations. An Integration author explicitly adds a Backend when support is desired.

## Rationale

This boundary aligns responsibility with the party most capable of maintaining it.

Engine maintainers own stable compiler and graph contracts. Infrastructure-domain authors own domain semantics and typed Domain Abstractions. Target authors own deployment-technology contracts, validation, emission, and conformance. Backend authors explicitly bridge the Domain and Target contracts.

Contract generations and conformance prevent implementation-version churn from becoming an automatic dependency cascade.

The in-process .NET assembly model provides the simplest viable initial extensibility mechanism while leaving room for stronger isolation later if demonstrated requirements justify it.

## Consequences

### Positive

- Infrastructure domains can evolve without requiring Engine Core releases.
- Third parties can independently own and release Integrations.
- Backends receive strongly typed domain inputs and Target outputs.
- Targets can be reused by multiple infrastructure domains.
- Compatibility is explicit, versioned, and executable through conformance tests.
- Engine, Integration, and Target implementation releases can evolve independently behind supported contract generations.
- New Targets do not create implicit support obligations for existing Integrations.

### Negative / risks

- Engine, Domain, and Target contracts become long-lived compatibility commitments.
- Supporting multiple contract generations increases maintenance and testing cost.
- In-process plugins create dependency-loading, assembly-isolation, and CLR type-identity challenges.
- Integration authors supporting similar Targets may write multiple Backend adapters even when implementation can be shared internally.
- Conformance tests cannot prove every domain mapping is semantically correct.
- Some deployment technologies may not fit the same internal Target IR/Emitter architecture and must not be forced into one.

## Guardrails

- Engine Core SHALL NOT contain infrastructure-domain-specific resource definitions or Target mappings.
- Target implementations SHALL NOT contain infrastructure-domain semantics.
- Integration authors SHALL NOT depend on private Target implementation details.
- Backends SHALL depend on published Domain and Target Abstractions rather than implementation assemblies.
- Backend compatibility with a Target MUST be expressed through an explicit contract generation and conformance evidence.
- Distinct deployment technologies SHALL be represented as distinct Targets even when they share syntax or ancestry.
- Engine SHALL NOT introduce Target families or compatibility profiles solely to deduplicate similar Targets.
- Engine SHALL NOT require a separate manifest file merely to duplicate information available through plugin contracts or assembly metadata.
- Engine SHALL NOT prescribe how an Integration internally constructs its Semantic Model.
- A new infrastructure resource type SHALL NOT require an Engine Core change unless it exposes a genuine missing capability in generic Engine contracts.
- A new Target SHALL be introducible without changing existing Integrations that do not support it.
- A new Integration SHALL be introducible without changing existing Targets or Engine Core when existing contracts are sufficient.

## Alternatives considered

### Engine owns all infrastructure domains

Rejected because it couples Engine releases to every supported platform and prevents independent ownership.

### Integration owns complete Target generation

Rejected as the primary architecture because it duplicates common Target modeling, validation, and emission and gives Integration authors no stable shared Target contract.

### Target owns domain mappings

Rejected because the Target would accumulate knowledge of every infrastructure domain and become a second semantic-model layer.

### Target families or compatibility profiles

Not proposed. Similarity today does not guarantee compatible evolution tomorrow, and a profile/version abstraction adds complexity without demonstrated need.

### External manifest-driven plugin system

Not proposed initially. Assembly metadata and published .NET contracts are sufficient until a concrete requirement demonstrates otherwise.

### Out-of-process extensions

Not proposed initially. This adds substantial operational and compatibility complexity without a demonstrated requirement.

## Open questions

- What is the exact `IInfrastructureIntegration` contract?
- What is the exact `ISemanticModel` contract and how does Engine invoke semantic behavior?
- What metadata must be available without fully activating a plugin?
- How are plugin assemblies discovered and loaded?
- How are plugin dependency conflicts and multiple contract generations isolated?
- What is the minimum mandatory content of Engine, Domain, and Target conformance suites?
- How should Targets that do not naturally use an IR-plus-Emitter architecture, potentially including Ansible, fit without weakening the core boundaries?
- What trust or signing requirements are needed if third-party plugins are eventually distributed broadly?