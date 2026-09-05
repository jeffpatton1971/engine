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

## Additive evolution within a generation

Contract generations SHOULD be intentionally slow-moving and conservative.

Within a generation, evolution SHOULD prefer additive changes that do not invalidate existing consumers. Public contracts SHOULD avoid large interfaces whose routine extension forces downstream implementations to change.

Where practical, new optional constructs, new value types, additional metadata, or narrowly scoped capability interfaces are preferable to modifying existing required members.

A new breaking generation is warranted when compatibility cannot be preserved without weakening the contract or introducing misleading behavior.

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
- Existing contract generations SHOULD remain unchanged where additive extension is sufficient.
- Compatibility metadata SHALL describe demonstrated support, not desired or presumed support.

## Consequences

### Positive

- Engine and Target implementations can evolve without routine downstream rebuild cascades.
- Backward compatibility becomes measurable rather than assumed.
- Multiple contract generations can provide controlled migration windows.
- Third-party authors can understand exactly which contract they target.
- Compatibility failures can be caught before release.
- Package version ranges no longer carry semantic responsibilities they cannot reliably prove.

### Negative / risks

- Supporting multiple contract generations increases maintenance and testing cost for the implementation owner.
- Conformance suites become long-lived artifacts that must themselves be maintained carefully.
- In-process .NET loading of multiple contract assemblies requires deliberate dependency isolation and resolution.
- Old contract generations require an explicit retirement policy eventually.
- Compatibility adapters inside an implementation may accumulate complexity if generations live indefinitely.

## Open questions

- What criteria justify creating a new contract generation?
- How long should an implementation normally support an older generation?
- What is the contract retirement/deprecation policy?
- How should contract-generation metadata be represented in plugin discovery?
- Should conformance evidence be embedded in release metadata or merely enforced by CI?
- How will the initial in-process assembly loader isolate dependencies when multiple generations coexist?
- Should compatible additive revisions exist inside a generation, or should a generation be immutable once published?
- What exact tests constitute the minimum Engine contract conformance suite?