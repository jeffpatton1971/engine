# ADR-005: Infrastructure Integrations and Extension Ownership

- **Status:** Proposed
- **Date:** 2026-09-05
- **Decision Type:** Architecture exploration

## Context

Engine is intended to compile infrastructure Intent without requiring Engine Core to understand every infrastructure domain or deployment technology.

Infrastructure domains evolve independently. A platform may add resource types, capabilities, constraints, or relationships without any corresponding change to the compiler itself. Likewise, deployment targets such as Terraform, OpenTofu, Bicep, ARM, CloudFormation, or Ansible evolve independently of the infrastructure domains they can represent.

The architecture therefore needs an ownership boundary that allows an infrastructure-domain author to extend Engine without modifying Engine Core and without requiring Engine maintainers to own the lifecycle of that domain.

It also needs a compatibility boundary between an infrastructure Integration and a deployment Target. An Integration may know how its domain semantics map to Terraform, for example, but it should not own Terraform's common model, validation rules, or physical HCL emission. Conversely, a Terraform Target cannot know how every possible infrastructure domain maps into Terraform.

## Proposed decision

Introduce **Infrastructure Integration** as the extension and ownership boundary for an infrastructure domain.

An Infrastructure Integration is an independently developed .NET plugin assembly that implements published Engine integration contracts.

Conceptually:

```text
Engine
  owns compilation contracts and lifecycle

Infrastructure Integration
  owns domain semantics and domain-to-target mappings

Target
  owns the target model, target contract, validation, and emission
```

An Integration exposes at least:

- stable integration identity and version information;
- a Semantic Model for its infrastructure domain;
- one or more Backend implementations or Backend capabilities for supported Targets;
- compatibility metadata required by the Engine contracts.

An illustrative contract might resemble:

```csharp
public interface IInfrastructureIntegration
{
    IntegrationIdentity Identity { get; }
    ISemanticModel SemanticModel { get; }
    IReadOnlyCollection<IBackend> Backends { get; }
}
```

This example is illustrative rather than an accepted API contract.

## Integration ownership

Engine defines **what an Integration must expose**, not **how an Integration must implement it**.

An Integration author owns:

- the completeness and correctness of the domain Semantic Model;
- supported resource types and domain concepts;
- domain identities, constraints, defaults, relationships, and semantic rules;
- compatibility with changes to the upstream infrastructure platform;
- domain-to-target Backend mappings;
- tests proving the Integration satisfies Engine and Target contracts;
- the Integration's release and version lifecycle.

An Integration author may construct its Semantic Model using any appropriate implementation technique, including hand-written code, generated definitions, reflection, source generation, upstream API schemas, provider metadata, internal libraries, or a combination of these.

Engine SHALL NOT prescribe those implementation details.

Adding or changing a resource type in an infrastructure domain SHALL NOT require a change to Engine Core solely because the domain evolved.

## .NET plugin model

The initial extension model SHALL be an in-process .NET plugin assembly implementing published Engine abstractions.

The assembly itself is sufficient as the initial packaging boundary. A separate external plugin manifest is not required unless a concrete requirement emerges that cannot reasonably be satisfied through assembly metadata and the integration contracts.

The Engine may require standardized metadata such as:

- integration ID;
- integration version;
- Engine contract/API version;
- Semantic Model version;
- supported Target IDs and Target contract versions;
- author or vendor identity where useful.

How discovery and dependency isolation are implemented remains a separate decision.

## Targets

A **Target** represents a deployment technology understood by Engine through a stable target contract.

Initial target families are expected to include technologies such as:

- Terraform;
- OpenTofu;
- Azure Bicep;
- Azure Resource Manager (ARM);
- AWS CloudFormation;
- Ansible.

This list is illustrative rather than exhaustive or a commitment that all targets will be implemented initially.

A Target owns the concepts that are common to that deployment technology. Depending on the technology, this may include:

- a Target IR or equivalent target model;
- target-specific type and expression contracts;
- target validation;
- serialization and emission rules;
- one or more Emitters;
- a versioned compatibility contract that Backends can compile against.

For example:

```text
Terraform Target
    |
    +-- Terraform Target Contract
    +-- Terraform IR
    +-- Terraform validation
    +-- HCL Emitter
    +-- Terraform JSON Emitter
```

## Backend compatibility

A Backend is a mapping between an Infrastructure Integration's semantics and a specific Target contract.

It is therefore more precise to describe Backends as domain-to-target mappings:

```text
VCFA Integration -> Terraform Backend -> Terraform Target
GCP Integration  -> Terraform Backend -> Terraform Target
Azure Integration -> Bicep Backend    -> Bicep Target
AWS Integration   -> CloudFormation Backend -> CloudFormation Target
```

An Integration that implements a Terraform Backend SHALL compile against a published, versioned Terraform Target contract rather than relying on private implementation details of the Terraform Target.

This creates an explicit compatibility relationship:

```text
VCFA Integration
    |
    +-- Backend: terraform
            |
            +-- requires Terraform Target Contract X

Engine
    |
    +-- resolves compatible Terraform Target
            |
            +-- validates Backend/Target compatibility
```

The same principle applies to Bicep, ARM, CloudFormation, Ansible, OpenTofu, and future Targets.

The exact compatibility mechanism is intentionally undecided, but compatibility MUST be explicit and testable rather than assumed from matching target names.

## Contract testing

Published Target contracts SHOULD provide reusable conformance or contract tests that Integration authors can execute against their Backends.

An Integration author should be able to establish compatibility before publication without requiring the Engine team or Target maintainer to modify their projects.

Conceptually:

```text
Integration Backend
       |
       v
Target Contract Test Kit
       |
       +-- model validity
       +-- supported constructs
       +-- deterministic lowering
       +-- target compatibility
       +-- diagnostic expectations
```

Runtime compatibility checks complement these tests but do not replace them.

## Target independence and reuse

Targets and Integrations have independent ownership and release lifecycles.

A Target does not contain infrastructure-domain semantics. An Integration does not own the common implementation of a Target.

This allows multiple Integrations to reuse the same Target:

```text
VCFA ----+
GCP -----+----> Terraform Target ----> HCL
Azure ---+
```

and allows one Integration to support multiple Targets:

```text
Azure Integration
    |
    +--> Terraform Target
    +--> Bicep Target
    +--> ARM Target
```

Not every Integration is required to support every Target.

## Rationale

This boundary aligns responsibility with the party most capable of maintaining it.

Engine maintainers own a stable compiler and extension contract. Infrastructure-domain authors own their domain. Target authors own deployment-technology semantics and representation. Backend authors explicitly bridge the two contracts.

A versioned Target contract prevents a Backend from merely hoping that a particular Terraform, Bicep, or other Target implementation remains compatible. It gives Integration authors something concrete to compile against and test against.

The in-process .NET assembly model provides the simplest viable extensibility mechanism for the initial architecture while leaving room for stronger isolation or alternative packaging later if real requirements justify them.

## Consequences

### Positive

- Infrastructure domains can evolve without requiring Engine Core releases.
- Third parties can independently own and release Integrations.
- Integration implementation details remain private to the Integration author.
- Targets can be reused by multiple infrastructure domains.
- Backend/Target compatibility becomes explicit and testable.
- Target maintainers can evolve target implementations behind versioned contracts.
- The initial plugin mechanism remains straightforward for .NET developers.

### Negative / risks

- Engine and Target contracts become public compatibility commitments.
- Version negotiation must eventually handle incompatible combinations clearly.
- In-process plugins can create dependency-loading and isolation problems.
- Contract tests cannot prove every semantic mapping is correct; Integration authors still own domain correctness.
- Some technologies may not fit the same Target IR/Emitter shape cleanly and must not be forced into an artificial abstraction.

## Guardrails

- Engine Core SHALL NOT contain infrastructure-domain-specific resource definitions or target mappings.
- Target implementations SHALL NOT contain infrastructure-domain semantics.
- Integration authors SHALL NOT depend on private Target implementation details.
- Backend compatibility with a Target MUST be expressed through a published contract.
- Engine SHALL NOT require a separate manifest file merely to duplicate information available through the plugin contract.
- Engine SHALL NOT prescribe how an Integration internally constructs its Semantic Model.
- A new infrastructure resource type SHALL NOT require an Engine Core change unless it exposes a genuine missing capability in the Engine's generic contracts.
- A new Target SHALL be introducible without changing existing Integrations that do not support it.
- A new Integration SHALL be introducible without changing existing Targets or Engine Core when existing contracts are sufficient.

## Alternatives considered

### Engine owns all infrastructure domains

Rejected because it couples Engine releases to every supported platform and prevents independent ownership.

### Integration owns complete target generation

Allow each Integration to generate Terraform, Bicep, or other artifacts directly.

Rejected as the primary architecture because it duplicates common target modeling, validation, and emission and gives Integration authors no stable shared target contract.

### Target owns domain mappings

Place VCFA, GCP, Azure, and other domain mappings inside the Terraform or other Target implementation.

Rejected because the Target would accumulate knowledge of every infrastructure domain and become a second semantic-model layer.

### External manifest-driven plugin system

Require declarative plugin manifests in addition to plugin assemblies.

Not proposed initially. Assembly metadata and published .NET contracts are sufficient until a concrete requirement demonstrates otherwise.

### Out-of-process extensions

Run Integrations through RPC, services, containers, or another isolation boundary.

Not proposed initially. This adds substantial operational and compatibility complexity without a demonstrated requirement.

## Open questions

- What is the exact `IInfrastructureIntegration` contract?
- What metadata must be available without fully activating a plugin?
- How are plugin assemblies discovered and loaded?
- How are plugin dependency conflicts isolated?
- What is the Target contract/version compatibility model?
- Are Target contracts separate .NET packages from Target implementations?
- What should a reusable Target conformance test kit contain?
- Can a Backend support a compatible family of Targets, such as Terraform and OpenTofu, or should each be modeled explicitly?
- How should Targets that do not naturally use an IR-plus-Emitter architecture, potentially including Ansible, fit without weakening the core boundaries?
- What trust or signing requirements are needed if third-party plugins are eventually distributed broadly?