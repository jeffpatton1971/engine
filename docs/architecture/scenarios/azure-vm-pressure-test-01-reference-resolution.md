# Azure VM Pressure Test 01: Typed Reference Resolution

> **Status:** Worked architecture pressure test
>
> This document tests missing and inconsistent typed references against the canonical identity model. Later brownfield pressure tests refined typed references to target domain contracts rather than managed implementation classes; this document reflects that refinement.

## Working contracts

```csharp
public readonly record struct ResourceIdentity(
    IntegrationId Integration,
    ResourceType Type,
    ResourceKey Key);

// Illustrative only
public readonly record struct ResourceReference<TDomain>(
    ResourceIdentity Identity);
```

`TDomain` represents an Integration-owned semantic contract such as `INetworkInterface`, not a lifecycle-specific managed implementation. The exact generic constraint and domain-contract-to-ResourceType metadata API remain open.

## Case 1: required reference omitted

If an Azure VM omits its required `networkInterface` field, Azure Integration reports a domain-local validation diagnostic.

> Integration validates that required references are declared.

Engine reference resolution is not responsible for discovering that Azure VM semantics require a NIC.

## Case 2: reference declared but NIC absent

Azure Integration may construct:

```text
ResourceReference<INetworkInterface>(
    azure | azure.network-interface | prod-rg/web-01-nic)
```

If no managed or existing node with that canonical identity exists, Engine reports an unresolved-reference diagnostic.

> Engine validates that declared canonical references resolve.

## Case 3: another resource type shares the key

A managed disk may legitimately have:

```text
azure | azure.managed-disk | prod-rg/web-01-nic
```

while the VM expects:

```text
azure | azure.network-interface | prod-rg/web-01-nic
```

These are different ResourceIdentity values because ResourceType participates in identity. Engine therefore does not accidentally resolve the NIC reference to the disk; it reports the missing NIC identity.

## Case 4: inconsistent typed reference

A genuine type inconsistency occurs if Integration code creates a reference whose domain contract and embedded ResourceType metadata disagree.

Conceptually:

```text
ResourceReference<INetworkInterface>
    Identity.Type = azure.managed-disk
```

This is an Integration-produced contract inconsistency rather than an ordinary missing user reference.

Engine should detect it through published Integration type metadata without knowing Azure CLR implementation classes.

Expected diagnostic:

```text
expected domain contract = INetworkInterface
identity resource type   = azure.managed-disk
-> inconsistent typed-reference diagnostic
```

## Managed and existing targets

The same reference:

```text
ResourceReference<INetworkInterface>
```

may resolve to:

```text
INetworkInterface + IManagedResource
```

or:

```text
INetworkInterface + IExistingResource
```

Lifecycle does not alter reference semantics.

## Ownership

```text
Integration
    declares that a NIC reference is required
    interprets domain scope/name
    constructs canonical NIC ResourceIdentity
    constructs ResourceReference<INetworkInterface>

Engine
    validates identity uniqueness
    validates typed-reference structural consistency
    resolves exact ResourceIdentity
    reports unresolved references
```

## Result

A normal reference to a nonexistent canonical identity is an unresolved-reference error. A wrong-type diagnostic refers specifically to an inconsistent typed-reference contract/metadata value, not to an unrelated resource sharing the same ResourceKey.

The remaining API question is how Integration-owned domain contracts are deterministically associated with canonical ResourceType metadata without making Engine domain-aware.