# Azure VM Pressure Test 02: Cross-Resource-Group Dependency Ordering

> **Status:** Worked architecture pressure test
>
> This document pressure-tests transitive dependency ordering when a virtual machine consumes networking resources that live in a different Azure Resource Group.

## Question

If a VM is deployed into one Resource Group while its Virtual Network, Subnet, and potentially NIC live in another Resource Group, does the Resource Graph still derive and preserve the correct prerequisite order?

The architecture should not rely on common Resource Group scope to make ordering work.

## Representative Intent

```yaml
resources:
  - type: azure.resource-group
    name: network-rg
    location: eastus

  - type: azure.resource-group
    name: compute-rg
    location: eastus

  - type: azure.virtual-network
    name: prod-vnet
    resourceGroup: network-rg
    addressSpace: 10.0.0.0/16

  - type: azure.subnet
    name: app
    resourceGroup: network-rg
    virtualNetwork: prod-vnet
    addressPrefix: 10.0.1.0/24

  - type: azure.network-interface
    name: web-01-nic
    resourceGroup: network-rg
    subnet: app

  - type: azure.virtual-machine
    name: web-01
    resourceGroup: compute-rg
    networkInterface: web-01-nic
    size: Standard_D4s_v5
```

The exact Intent schema remains illustrative.

## Domain facts

Azure permits a Network Interface to exist in a different Resource Group from the VM to which it is attached and from the Virtual Network to which it connects, subject to Azure's subscription/location constraints.

The Integration therefore cannot infer dependency ordering merely from resources sharing the same Resource Group.

Instead, ordering must follow resolved semantic references and prerequisite rules.

## Canonical identities

Azure Integration might materialize identities conceptually resembling:

```text
azure.resource-group.network-rg
azure.resource-group.compute-rg
azure.virtual-network.network-rg/prod-vnet
azure.subnet.network-rg/prod-vnet/app
azure.network-interface.network-rg/web-01-nic
azure.virtual-machine.compute-rg/web-01
```

Resource Group scope remains an Azure Integration concern represented in canonical ResourceKeys where needed. Engine does not understand Resource Group semantics.

## Semantic relationships

Representative domain relationships are:

```text
VirtualNetwork:network-rg/prod-vnet
    --contained-in--> ResourceGroup:network-rg

Subnet:network-rg/prod-vnet/app
    --contained-in--> VirtualNetwork:network-rg/prod-vnet

NetworkInterface:network-rg/web-01-nic
    --contained-in--> ResourceGroup:network-rg

NetworkInterface:network-rg/web-01-nic
    --attached-to--> Subnet:network-rg/prod-vnet/app

VirtualMachine:compute-rg/web-01
    --contained-in--> ResourceGroup:compute-rg

VirtualMachine:compute-rg/web-01
    --attached-to--> NetworkInterface:network-rg/web-01-nic
```

The VM-to-NIC relationship crosses Resource Group scope without changing its semantic meaning.

## Derived dependency graph

Azure semantic rules may derive:

```text
ResourceGroup:network-rg
    -> VirtualNetwork:network-rg/prod-vnet

VirtualNetwork:network-rg/prod-vnet
    -> Subnet:network-rg/prod-vnet/app

Subnet:network-rg/prod-vnet/app
    -> NetworkInterface:network-rg/web-01-nic

ResourceGroup:network-rg
    -> NetworkInterface:network-rg/web-01-nic

ResourceGroup:compute-rg
    -> VirtualMachine:compute-rg/web-01

NetworkInterface:network-rg/web-01-nic
    -> VirtualMachine:compute-rg/web-01
```

The key observation is that the VM does not need a direct dependency edge to every transitive prerequisite merely so ordering works.

The path:

```text
network-rg
    -> prod-vnet
    -> app subnet
    -> web-01-nic
    -> web-01 VM
```

is sufficient for Engine's topological sort to require those resources to precede the VM.

Separately:

```text
compute-rg
    -> web-01 VM
```

ensures the VM's own Resource Group is also created first.

Engine can therefore derive a valid deterministic ordering such as:

```text
1. ResourceGroup:compute-rg
2. ResourceGroup:network-rg
3. VirtualNetwork:network-rg/prod-vnet
4. Subnet:network-rg/prod-vnet/app
5. NetworkInterface:network-rg/web-01-nic
6. VirtualMachine:compute-rg/web-01
```

The relative order of the two independent Resource Groups is not semantically important, so Engine's deterministic tie-breaking rule decides which appears first.

## Why transitive dependencies matter

A tempting implementation would add all prerequisite ancestors directly to the VM:

```text
network-rg -> VM
VNet       -> VM
Subnet     -> VM
NIC        -> VM
compute-rg -> VM
```

That is unnecessary for ordering when the graph already contains:

```text
network-rg -> VNet -> Subnet -> NIC -> VM
```

Engine should preserve the direct prerequisite facts supplied by Integration semantics and rely on graph traversal/topological ordering for transitive closure.

This avoids bloating the graph with redundant dependency edges and preserves a clearer explanation of why each dependency exists.

## Validation ownership

Azure Integration validates Azure-local cross-resource constraints that require domain knowledge, such as whether referenced networking resources satisfy subscription/location rules.

Engine validates graph mechanics:

- every referenced `ResourceIdentity` resolves;
- all dependency endpoints exist;
- no dependency cycle is introduced;
- deterministic ordering respects the resolved edges.

The Integration owns why the edges exist. Engine owns ordering over those edges.

## Failure variant: network Resource Group omitted

If Intent declares the VNet, Subnet, and NIC under `network-rg` but does not declare a Resource Group resource for `network-rg`, behavior depends on the eventual Intent contract.

If Resource Groups represented in the compilation are required to be explicit resources, then:

```text
network-rg reference declared
ResourceGroup:network-rg absent
```

is an Engine unresolved-reference failure after Azure Integration creates the typed Resource Group reference.

If a future Integration supports references to externally managed resources, that is a separate architectural capability and must be explicit rather than silently treating a missing graph node as pre-existing infrastructure.

## Result

The current architecture handles the cross-Resource-Group scenario cleanly.

The required ordering emerges from direct semantic dependencies:

```text
Network Resource Group
    -> Virtual Network
    -> Subnet
    -> NIC
    -> VM

VM Resource Group
    -> VM
```

Resource Group boundaries do not weaken graph ordering because dependencies are identity-based rather than inferred from containment or source layout.

This pressure test strengthens two requirements:

1. Engine topological ordering must honor transitive dependency paths without requiring redundant transitive edges.
2. External/pre-existing resource references, if supported later, must be modeled explicitly rather than inferred from missing Intent resources.