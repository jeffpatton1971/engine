# Azure VM Pressure Test 01: Typed Reference Resolution

> **Status:** Worked architecture pressure test
>
> This document refines the Azure VM semantic-analysis scenario by testing missing and wrong-type resource references against the current `ResourceIdentity` and `ResourceReference<TResource>` decisions.

## Question

What happens when an Azure VM declares a required Network Interface reference but the referenced resource is either absent or is actually a different resource type?

The current candidate contracts are conceptually:

```csharp
public readonly record struct ResourceIdentity(
    IntegrationId Integration,
    ResourceType Type,
    ResourceKey Key);

public readonly record struct ResourceReference<TResource>(
    ResourceIdentity Identity)
    where TResource : IResource;
```

`ResourceType` is therefore already part of authoritative graph identity.

## Case 1: required reference omitted

Intent:

```yaml
- type: azure.virtual-machine
  name: web-01
  resourceGroup: prod-rg
  size: Standard_D4s_v5
```

There is no `networkInterface` value.

Expected behavior:

```text
Azure Integration / Semantic Model
    detects that a required VM field is absent
    -> local/domain validation failure
```

Engine reference resolution is not responsible for discovering that the domain requires a NIC reference.

### Result

This supports the existing validation rule:

> Integration validates that required references are declared.

## Case 2: reference declared but NIC object absent

Intent:

```yaml
- type: azure.virtual-machine
  name: web-01
  resourceGroup: prod-rg
  networkInterface: web-01-nic
  size: Standard_D4s_v5
```

Azure Integration can materialize a typed reference conceptually equivalent to:

```text
ResourceReference<NetworkInterfaceResource>(
    azure.network-interface.prod-rg/web-01-nic)
```

but no resource with that identity exists in the materialized resource set.

Expected behavior:

```text
Azure Integration
    successfully creates the required typed reference

Engine reference resolution
    cannot find the referenced ResourceIdentity
    -> unresolved-reference failure
```

### Result

This supports the existing validation rule:

> Engine validates that declared references resolve.

## Case 3: a Managed Disk happens to use the same ResourceKey

Assume Intent contains:

```yaml
- type: azure.managed-disk
  name: web-01-nic
  resourceGroup: prod-rg
  sizeGb: 128

- type: azure.virtual-machine
  name: web-01
  resourceGroup: prod-rg
  networkInterface: web-01-nic
  size: Standard_D4s_v5
```

The disk may have identity:

```text
azure.managed-disk.prod-rg/web-01-nic
```

while the VM's typed NIC reference identifies:

```text
azure.network-interface.prod-rg/web-01-nic
```

These are different `ResourceIdentity` values because `ResourceType` participates in identity.

Therefore Engine does **not** resolve the NIC reference to the Managed Disk and then discover a type mismatch. It simply finds that the requested Network Interface identity does not exist.

Expected behavior:

```text
Engine reference resolution
    lookup azure.network-interface.prod-rg/web-01-nic
    -> not found
    -> unresolved-reference failure
```

### Result

The current identity design prevents cross-resource-type accidental resolution by construction.

This is desirable because a resource with the same `ResourceKey` but a different `ResourceType` is not the referenced resource.

## Case 4: malformed typed reference

A true wrong-type inconsistency can still occur if Integration code constructs a reference whose generic type and embedded identity disagree:

```csharp
new ResourceReference<NetworkInterfaceResource>(
    new ResourceIdentity(
        Azure,
        AzureResourceTypes.ManagedDisk,
        new ResourceKey("prod-rg/web-01-data")));
```

This is not an ordinary user reference-resolution failure. It is an internally inconsistent Integration-produced value.

Engine can detect this before or during resolution by validating that the `ResourceType` expected by `TResource` is compatible with `ResourceReference<TResource>.Identity.Type`.

Expected behavior:

```text
Engine / contract validation
    typed reference expectation = NetworkInterfaceResource
    identity resource type = managed-disk
    -> inconsistent typed-reference diagnostic
```

The exact mechanism by which Engine knows the `ResourceType` corresponding to `TResource` remains an API-design question. It must not require Engine to understand Azure-specific resource classes.

## Architectural finding

The pressure test changes the way we should describe wrong-type reference failures.

There are two materially different cases:

```text
Expected resource type exists only in the reference,
but no resource of that identity exists
    -> unresolved reference

ResourceReference<TResource> itself contains an identity
whose ResourceType contradicts TResource
    -> malformed/inconsistent typed reference
```

Because `ResourceType` participates in `ResourceIdentity`, an unrelated resource with the same `ResourceKey` cannot accidentally satisfy the reference.

## Ownership

```text
Integration / Semantic Model
    declares that NetworkInterface is required
    constructs ResourceReference<NetworkInterfaceResource>
    constructs the expected Network Interface ResourceIdentity

Engine
    validates identity uniqueness
    validates typed-reference structural consistency
    resolves the exact ResourceIdentity
    reports unresolved references
```

## Consequence for the ResourceReference contract

This test strengthens the case for a typed reference, but it also exposes a contract question:

> How is the expected Engine-level `ResourceType` associated with `TResource` without requiring Engine to know Integration-specific CLR types?

Possible approaches should be explored later rather than chosen here. Examples include Domain Abstractions exposing stable type metadata, a generic resource-type contract implemented by resource types, or Integration-provided reference descriptors.

The important requirement is that the association be deterministic and testable without making Engine domain-aware.

## Result

The existing architecture survives the pressure test, with one refinement:

> A normal reference to a nonexistent resource of the expected type is an unresolved-reference error. A "wrong type" error should refer specifically to an internally inconsistent typed reference or equivalent contract violation, not to an unrelated resource that merely shares the same key.

This distinction should inform the eventual Resource Graph diagnostic contract and the design of `ResourceReference<TResource>`.