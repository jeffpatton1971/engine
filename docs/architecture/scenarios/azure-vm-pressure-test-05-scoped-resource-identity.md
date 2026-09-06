# Azure VM Pressure Test 05: Scoped Resource Identity

> **Status:** Worked architecture pressure test
>
> This document tests whether `ResourceIdentity` and typed references remain unambiguous when multiple resources have the same local name but exist in different infrastructure scopes.

## Scenario

Two Azure Virtual Networks each contain a subnet named `app`.

```text
Resource Group: network-east-rg
    VNet: east-vnet
        Subnet: app

Resource Group: network-west-rg
    VNet: west-vnet
        Subnet: app
```

A NIC must attach specifically to the `app` subnet in `east-vnet`.

This scenario is intentionally ordinary. Local resource names are not guaranteed to be globally unique, and Engine must not require Integration authors or Intent authors to invent artificial globally unique names.

## Existing identity model

The working Engine identity is:

```text
ResourceIdentity
    IntegrationId
    ResourceType
    ResourceKey
```

Engine treats the complete tuple as authoritative and enforces uniqueness.

The Integration owns construction of the canonical `ResourceKey` because only the Integration understands the infrastructure domain's identity and scoping rules.

## Incorrect key construction

If Azure Integration constructed both subnet identities as:

```text
Integration = azure
Type        = azure.subnet
Key         = app
```

Engine would correctly report a duplicate identity.

Engine must not attempt to repair this by inspecting Azure properties, parent relationships, source order, or resource names.

The error would mean Azure Integration failed to encode enough domain scope into its canonical key.

## Correct scoped keys

Azure Integration can encode the domain's required scope into `ResourceKey`.

Illustratively:

```text
azure | azure.subnet | network-east-rg/east-vnet/app
azure | azure.subnet | network-west-rg/west-vnet/app
```

The exact string format is owned by Azure Integration and is not an Engine convention.

The identities are now distinct even though both resources have the local name `app`.

## Typed reference

The NIC's Domain Abstraction can continue to express:

```csharp
ResourceReference<ISubnet> Subnet
```

The reference contains the canonical identity for the intended subnet:

```text
Integration = azure
Type        = azure.subnet
Key         = network-east-rg/east-vnet/app
```

Engine resolves that exact identity.

There is no search for "a subnet named app" and no ambiguity algorithm in Engine.

## Managed and existing variants

The scoped identity behaves the same regardless of lifecycle.

For example:

```text
IExistingResource + ISubnet
    network-east-rg/east-vnet/app

IManagedResource + ISubnet
    network-west-rg/west-vnet/app
```

A NIC referencing the east subnet resolves to the `IExistingResource` implementation of `ISubnet`. A NIC referencing the west subnet resolves to the `IManagedResource` implementation.

The typed reference describes what semantic resource type is required. The resolved node's lifecycle type determines provisioning behavior.

No alternate identity scheme is needed for brownfield resources.

## Same key across different resource types

Because `ResourceType` is part of `ResourceIdentity`, this is also valid:

```text
azure | azure.virtual-network | network-east-rg/app
azure | azure.subnet          | network-east-rg/east-vnet/app
```

Even if some local names happen to match, the complete identities remain distinct.

## Same key across different Integrations

Likewise:

```text
azure | azure.subnet | network-rg/vnet/app
gcp   | gcp.subnet   | network-rg/vnet/app
```

cannot collide because `IntegrationId` is part of identity.

Engine therefore needs no universal cloud naming or scoping taxonomy.

## Reference authoring versus canonical identity

The pressure test reveals an important separation.

Engine resolution should always operate on canonical `ResourceIdentity`, but source Intent does not necessarily need to expose the canonical key syntax directly.

An Azure Adapter / Integration might accept author-friendly fields such as:

```yaml
subnet:
  resourceGroup: network-east-rg
  virtualNetwork: east-vnet
  name: app
```

and Azure Integration can canonicalize those domain inputs into:

```text
network-east-rg/east-vnet/app
```

Alternatively, another source format may already provide a canonical identifier.

The rule is:

> Source syntax may be convenient and domain-oriented; graph identity must be canonical and unambiguous before Engine identity registration and reference resolution.

## Failure cases

### Insufficient scope

If Intent says only:

```yaml
subnet: app
```

while both `east-vnet/app` and `west-vnet/app` are possible and Azure semantics cannot determine the intended scope from surrounding resource context, the Integration should report an ambiguity / insufficient-scope diagnostic.

Engine should not choose one.

### Canonical reference does not exist

If Azure Integration constructs:

```text
network-east-rg/east-vnet/missing
```

and no matching managed or existing `ISubnet` node is registered, Engine reports the unresolved reference during reference resolution.

### Duplicate canonical identity

If two materialized nodes produce the same complete identity:

```text
azure | azure.subnet | network-east-rg/east-vnet/app
```

Engine reports a deterministic duplicate-identity diagnostic during identity registration.

It does not merge them or decide which declaration wins.

## Result

The existing identity model survives the scoped-name pressure test without modification.

Working rules:

1. Local resource names do not need to be globally unique.
2. Integration authors own canonical `ResourceKey` construction and must encode whatever domain scope is necessary for uniqueness.
3. Engine treats `IntegrationId + ResourceType + ResourceKey` as authoritative identity.
4. Engine does not understand Azure Resource Groups, VNets, subscriptions, projects, regions, accounts, or equivalent domain scoping constructs merely to resolve identity.
5. Typed references target domain contracts such as `ISubnet` and carry canonical resource identity.
6. Managed and existing implementations of the same domain contract use the same identity scheme.
7. Ambiguous source-level references are an Integration semantic/input diagnostic; unresolved canonical references are an Engine graph-resolution diagnostic.
8. Source syntax need not expose canonical key formatting directly.

## Architectural conclusion

> Scope is a domain identity concern, not an Engine graph concern.

Engine requires globally unambiguous identities within a compilation, but the Integration decides how infrastructure-domain scope becomes a canonical `ResourceKey`.

This preserves a generic Engine while supporting resource models in which identical local names legitimately exist under different parents or scopes.
