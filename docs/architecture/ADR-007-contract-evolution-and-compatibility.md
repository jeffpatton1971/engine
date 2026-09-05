# ADR-007: Contract Evolution and Compatibility

- **Status:** Proposed
- **Date:** 2026-09-05
- **Decision Type:** Architecture exploration

## Context

Engine and Target contracts are intended to permit independently developed components to evolve without forcing implementation-version changes to cascade across the ecosystem.

A shared abstraction can itself become a source of coupling if every implementation release requires downstream consumers to rebuild, or if compatibility is inferred from package versions rather than demonstrated. Major-version conventions alone are insufficient: a newer implementation may intentionally continue supporting an older contract, while a nominally compatible package version may still contain behavioral incompatibilities.

The architecture therefore needs explicit contract generations and executable evidence for every compatibility claim.

## Proposed decision

Implementation versions SHALL NOT imply contract compatibility.

Engine runtimes and Target implementations SHALL explicitly declare the contract generations they support.

A component SHALL only advertise support for a contract generation when it passes that generation's applicable conformance suite.

A contract generation is immutable in its required public shape and semantics once published.

Package revisions within that generation MAY contain non-breaking fixes, documentation, tooling, metadata, and other changes that do not invalidate an already-conformant consumer.

Breaking contract generations SHOULD be independently addressable so that multiple generations can coexist where an implementation intentionally supports them.

## Contract generations

Contracts are versioned independently from implementations.

For example:

```text
Terraform Target implementation: 4.2.0

Supported Target contracts:
    Terraform Contract V1
    Terraform Contract V2
```

A Backend compiled against Terraform Contract V1 does not require Terraform Target implementation 1.x. It requires an installed Terraform Target that explicitly supports Contract V1.

Likewise, an Engine implementation may evolve substantially while continuing to support an older Engine extension contract.

## Immutable contract generations

A contract generation defines a durable compatibility promise.

For example:

```text
Terraform.Target.Abstractions.V1 1.0.0
Terraform.Target.Abstractions.V1 1.0.1
Terraform.Target.Abstractions.V1 1.4.7
```

All of these revisions remain part of Contract V1 and SHALL preserve the same required public shape and semantics.

Within a contract generation, revisions MAY include:

- documentation corrections;
- analyzers, test helpers, tooling, or metadata improvements;
- non-breaking implementation fixes in helper code;
- packaging fixes;
- changes that preserve all existing public requirements and conformance behavior.

Within a contract generation, revisions SHALL NOT:

- remove required public members;
- change required member types or signatures;
- change the meaning of existing fields, members, or operations;
- make previously conformant consumers non-conformant;
- add new required interface members or required behaviors;
- silently redefine validity rules for existing contract constructs.

If a desired change cannot preserve those guarantees, a new contract generation is required.

New optional capabilities SHOULD be introduced alongside the immutable generation rather than by mutating its required surface. Examples include separate capability interfaces, optional model constructs, or companion abstractions that existing consumers are not required to implement or emit.

This means V1.0.0 and V1.4.7 may differ as packages, but they represent the same fundamental compatibility contract.

## Independently addressable breaking contracts

A breaking contract generation may be represented by a physically distinct package and assembly:

```text
Terraform.Target.Abstractions.V1.dll
Terraform.Target.Abstractions.V2.dll
```

This permits Backends compiled against different generations to coexist without depending on package-version range behavior or assuming binary compatibility between breaking generations.

For example:

```text
VCFA Terraform Backend
    -> Terraform.Target.Abstractions.V1

GCP Terraform Backend
    -> Terraform.Target.Abstractions.V2

Terraform Target
    -> supports V1
    -> supports V2
```

The exact naming convention remains open. The architectural requirement is that breaking generations can be distinguished and resolved explicitly.

## Compatibility is earned by testing

Every contract generation SHALL provide or identify an applicable versioned conformance suite.

If a newer implementation claims compatibility with an older contract generation, its CI SHALL execute the older generation's conformance suite against the newer implementation.

For example:

```text
Terraform.Target 4.2.0
    |
    +-- Terraform Contract V1 Conformance -> PASS
    +-- Terraform Contract V2 Conformance -> PASS
```

Only then may the implementation advertise:

```text
SupportedContracts = [V1, V2]
```

If V1 conformance fails while V2 passes, the implementation SHALL NOT advertise V1 compatibility.

Release validation SHOULD fail when advertised compatibility differs from demonstrated conformance results.

The conformance suite for an immutable generation becomes part of the durable definition of what support for that generation means. Revisions to the suite SHALL NOT introduce new mandatory behavior that would invalidate a previously conformant consumer unless the earlier suite was demonstrably incorrect relative to the published contract. Such corrections require explicit review and documentation because they alter compatibility evidence.

## Compatibility matrix

Compatibility documentation or metadata SHOULD be generated from conformance evidence rather than maintained as an unsupported assertion.

For example:

| Target implementation | Contract V1 | Contract V2 | Contract V3 |
| --- | --- | --- | --- |
| 1.x | Pass | - | - |
| 2.x | Pass | Pass | - |
| 3.x | Fail | Pass | Pass |

A newer implementation is therefore not assumed to support an older contract. Backward compatibility is explicit and testable.

## Engine contract compatibility

The same model applies to Engine extension contracts.

For example:

```text
Engine 5.0
    |
    +-- Engine Contract V1 Conformance -> PASS
    +-- Engine Contract V2 Conformance -> PASS
```

Engine 5.0 may advertise V1 and V2 support only while both suites pass.

This allows the Engine implementation to evolve without requiring every Integration or extension to move merely because the Engine implementation version changed.

## Target contract compatibility

Target implementation versions and Target contract generations remain independent.

For example:

```text
Backend:
    Target = terraform
    Contract = V1

Installed Target:
    Target = terraform
    Implementation = 4.2.0
    SupportedContracts = [V1, V2]

Resolution:
    compatible
```

Engine SHALL resolve compatibility from Target identity and explicitly supported contract generations rather than from implementation-version ranges.

## Contract extension without mutation

Once a generation is published, its required public contract is frozen.

The preferred way to add functionality without creating a breaking generation is to compose new optional capabilities around the existing contract rather than editing required members in place.

For example, instead of adding a new required member to an existing interface, the ecosystem may introduce a separate capability interface that implementations opt into.

```text
Contract V1
    stable required surface

Optional Capability A
    independent additive surface

Optional Capability B
    independent additive surface
```

Existing V1 consumers remain valid and do not need to rebuild merely because new optional capabilities exist.

A new contract generation is warranted when the existing generation's required shape or semantics must change.

## Conformance versus migration

Contract conformance and contract migration are distinct concerns.

**Conformance** answers:

> Can this implementation correctly support components built against this contract generation?

**Migration** answers:

> Can an artifact, Backend, or model built for one contract generation be transformed into another generation without changing its semantics?

This ADR requires conformance testing. It does not require migration tooling. Migration support may be introduced later if demonstrated use cases justify it.

## Dependency-cascade mitigation

The intended effect is that implementation releases do not automatically propagate through dependent components.

For example:

```text
Engine implementation change
    -> no Integration rebuild when supported Engine contract is unchanged

Terraform Target implementation change
    -> no Backend rebuild when supported Terraform contract is unchanged

VCFA Integration change
    -> no Engine or Target rebuild
```

Ecosystem-wide work should occur only when a public contract genuinely changes or when support for an older contract generation is intentionally retired.

Even then, supporting multiple contract generations can provide a migration window rather than forcing simultaneous upgrades.

## Guardrails

- Implementation version numbers SHALL NOT be used as proof of contract compatibility.
- A newer implementation SHALL NOT be assumed to support older contract generations.
- Every advertised contract generation SHALL have executable conformance evidence.
- Release validation SHOULD verify advertised compatibility against conformance results.
- Breaking contract generations SHOULD be independently addressable.
- Contract evolution SHOULD be substantially slower than implementation evolution.
- A published contract generation SHALL remain immutable in its required public shape and semantics.
- Package revisions within a generation SHALL NOT invalidate already-conformant consumers.
- New required behavior or required structure SHALL require a new contract generation.
- Optional capabilities SHOULD be composed alongside existing contracts rather than mutating required surfaces.
- Compatibility metadata SHALL describe demonstrated support, not desired or presumed support.

## Consequences

### Positive

- Engine and Target implementations can evolve without routine downstream rebuild cascades.
- Backward compatibility becomes measurable rather than assumed.
- Multiple contract generations can provide controlled migration windows.
- Third-party authors can understand exactly which contract they target.
- Compatibility failures can be caught before release.
- Package version ranges no longer carry semantic responsibilities they cannot reliably prove.
- Consumers can take compatible package revisions without accepting a changed public contract.
- Optional capabilities can evolve without forcing every existing consumer to move.

### Negative / risks

- Supporting multiple contract generations increases maintenance and testing cost for the implementation owner.
- Conformance suites become long-lived artifacts that must themselves be maintained carefully.
- In-process .NET loading of multiple contract assemblies requires deliberate dependency isolation and resolution.
- Old contract generations require an explicit retirement policy eventually.
- Compatibility adapters inside an implementation may accumulate complexity if generations live indefinitely.
- Strictly immutable contract generations may produce more major contract generations over time if the initial contracts are too narrow.

## Open questions

- What criteria justify creating a new contract generation?
- How long should an implementation normally support an older generation?
- What is the contract retirement/deprecation policy?
- How should contract-generation metadata be represented in plugin discovery?
- Should conformance evidence be embedded in release metadata or merely enforced by CI?
- How will the initial in-process assembly loader isolate dependencies when multiple generations coexist?
- What exact tests constitute the minimum Engine contract conformance suite?
- How should optional capability contracts themselves be versioned and tested?