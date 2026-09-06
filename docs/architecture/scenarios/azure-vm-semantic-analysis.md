# Azure VM Semantic Analysis Scenario

> **Status:** Worked architecture scenario
>
> This document is not an ADR. It pressure-tests ADR-008, ADR-009, and ADR-010 with a realistic Azure resource set before public interfaces are frozen.

## Purpose

This scenario exercises the current Engine architecture using a small Azure infrastructure model and asks whether each responsibility remains owned by the correct component.

The scenario should prove or challenge the following assumptions:

- Adapters translate source syntax only.
- Azure Integration owns Azure domain semantics and typed resource materialization.
- Azure Domain Abstractions define the concrete strongly typed resource contracts shared with Azure Backends.
- Engine owns identity registration, reference resolution, graph construction, dependency analysis, cycle detection, and deterministic ordering.
- Relationships express domain meaning; dependencies express prerequisite ordering.
- Backends consume the resolved graph without returning to raw Intent.
- Source declaration order has no semantic meaning.

## Scenario resource set

The initial model contains:

```text
Resource Group
Virtual Network
Subnet
Network Interface
Managed Disk
Virtual Machine
```

This is intentionally small but relational enough to exercise identity, typed references, semantic relationships, dependency derivation, and deterministic graph ordering.

## Representative Intent

The example uses YAML only as a convenient source representation. Equivalent JSON, API, or other Adapter input should produce equivalent Parsed Intent and Resource Graph semantics.

```yaml
resources:
  - type: azure.resource-group
    name: prod-rg
    location: eastus

  - type: azure.virtual-network
    name: prod-vnet
    resourceGroup: prod-rg
    addressSpace: 10.0.0.0/16

  - type: azure.subnet
    name: app
    resourceGroup: prod-rg
    virtualNetwork: prod-vnet
    addressPrefix: 10.0.1.0/24

  - type: azure.network-interface
    name: web-01-nic
    resourceGroup: prod-rg
    subnet: app

  - type: azure.managed-disk
    name: web-01-data
    resourceGroup: prod-rg
    sizeGb: 128

  - type: azure.virtual-machine
    name: web-01
    resourceGroup: prod-rg
    networkInterface: web-01-nic
    dataDisk: web-01-data
    size: Standard_D4s_v5
```

The exact Intent schema is illustrative. This scenario is testing architecture ownership, not committing to a YAML contract.

## Phase 0: Adapter translation

`Yaml.Adapter` parses and normalizes the external representation into Parsed Intent.

For the VM, Parsed Intent may conceptually resemble:

```text
IntentResource
    Type = azure.virtual-machine
    Name = web-01
    Properties:
        resourceGroup = prod-rg
        networkInterface = web-01-nic
        dataDisk = web-01-data
        size = Standard_D4s_v5
```

The Adapter does not:

- decide how Azure VM identity is scoped;
- construct `VirtualMachineResource`;
- determine whether the NIC or disk exists;
- decide whether a VM-to-disk relationship implies a dependency;
- construct Resource Graph edges.

## Phase 1: Integration materialization

`Azure.Integration` recognizes each Azure Intent resource and materializes a concrete resource type from `Azure.Abstractions`.

Conceptually, the VM may become:

```csharp
public sealed record VirtualMachineResource : IResource
{
    public required ResourceIdentity Identity { get; init; }
    public required VirtualMachineSize Size { get; init; }
    public required ResourceReference<ResourceGroupResource> ResourceGroup { get; init; }
    public required ResourceReference<NetworkInterfaceResource> NetworkInterface { get; init; }
    public required ResourceReference<ManagedDiskResource> DataDisk { get; init; }
}
```

An illustrative materialized value is:

```text
VirtualMachineResource
    Identity:
        Integration = azure
        Type = virtual-machine
        Key = prod-rg/web-01
    Size = Standard_D4s_v5
    ResourceGroup -> ResourceReference<ResourceGroupResource>(azure.resource-group.prod-rg)
    NetworkInterface -> ResourceReference<NetworkInterfaceResource>(azure.network-interface.prod-rg/web-01-nic)
    DataDisk -> ResourceReference<ManagedDiskResource>(azure.managed-disk.prod-rg/web-01-data)
```

The exact Azure ResourceKey format belongs to Azure Integration / Domain Abstractions. Engine only requires that the complete `ResourceIdentity` be canonical and unique.

### Phase 1 validation

Azure Integration performs resource-local and domain-local validation that does not require graph resolution.

Examples:

- required VM fields are present;
- VM name is valid for the Azure domain contract;
- `size` is syntactically/domain-valid;
- disk size is valid;
- required resource-group reference is present;
- address-space and subnet-prefix values are syntactically valid;
- mutually exclusive or cross-field Azure rules are satisfied when those checks do not require resolving another resource.

Azure Integration does not yet decide whether `web-01-nic`, `web-01-data`, or `prod-rg` actually exist in the compilation.

## Phase 2: Engine identity registration

Engine collects all successfully materialized resources and registers their authoritative identities.

Conceptually:

```text
azure.resource-group.prod-rg
azure.virtual-network.prod-rg/prod-vnet
azure.subnet.prod-rg/prod-vnet/app
azure.network-interface.prod-rg/web-01-nic
azure.managed-disk.prod-rg/web-01-data
azure.virtual-machine.prod-rg/web-01
```

Engine validates that the complete tuple:

```text
Integration + ResourceType + ResourceKey
```

is unique.

If two resources produce the same identity, Engine emits a deterministic duplicate-identity diagnostic. Engine does not repair, rename, or reinterpret the Integration-owned ResourceKey.

## Phase 3: Engine reference resolution

Engine resolves each typed `ResourceReference<TResource>` against the complete registered resource set.

For the VM:

```text
ResourceGroup
    -> ResourceGroupResource: prod-rg

NetworkInterface
    -> NetworkInterfaceResource: prod-rg/web-01-nic

DataDisk
    -> ManagedDiskResource: prod-rg/web-01-data
```

Engine owns validation that:

- the referenced identity exists;
- the target resource is of the expected type;
- duplicate identities did not make resolution ambiguous.

The Integration owns the meaning of the typed reference; Engine owns the lookup and resolution mechanics.

Source declaration order is irrelevant. The VM may appear before the NIC or disk in YAML and must resolve identically once all resources are registered.

## Phase 4: Azure semantic analysis

After reference resolution, Azure Integration / Semantic Model interprets the resolved resources and supplies domain semantics.

### Semantic relationships

Representative relationships may include:

```text
VirtualNetwork:prod-vnet
    --contained-in--> ResourceGroup:prod-rg

Subnet:app
    --contained-in--> VirtualNetwork:prod-vnet

NetworkInterface:web-01-nic
    --contained-in--> ResourceGroup:prod-rg

NetworkInterface:web-01-nic
    --attached-to--> Subnet:app

ManagedDisk:web-01-data
    --contained-in--> ResourceGroup:prod-rg

VirtualMachine:web-01
    --contained-in--> ResourceGroup:prod-rg

VirtualMachine:web-01
    --attached-to--> NetworkInterface:web-01-nic

VirtualMachine:web-01
    --uses-disk--> ManagedDisk:web-01-data
```

These relationships express Azure domain meaning. Engine does not invent their relationship types.

### Dependency semantics

Azure semantic rules may derive prerequisite edges such as:

```text
ResourceGroup:prod-rg
    -> VirtualNetwork:prod-vnet

ResourceGroup:prod-rg
    -> NetworkInterface:web-01-nic

ResourceGroup:prod-rg
    -> ManagedDisk:web-01-data

ResourceGroup:prod-rg
    -> VirtualMachine:web-01

VirtualNetwork:prod-vnet
    -> Subnet:app

Subnet:app
    -> NetworkInterface:web-01-nic

NetworkInterface:web-01-nic
    -> VirtualMachine:web-01

ManagedDisk:web-01-data
    -> VirtualMachine:web-01
```

The exact Azure provisioning rules here are illustrative and should be validated against the real Azure Integration when implemented. The architectural point is ownership: Azure Integration defines whether a relationship or platform rule implies a prerequisite; Engine owns the resulting structural dependency edge.

Dependency reasons or rule identities remain separate provenance. They are not fields on `ResourceDependency`.

## Phase 5: Engine graph construction and validation

Engine constructs the deterministic Resource Graph from:

- concrete typed Azure resource instances;
- resolved semantic relationship facts;
- resolved structural dependency facts.

Conceptually:

```text
Resources
    ResourceGroup:prod-rg
    VirtualNetwork:prod-vnet
    Subnet:app
    NetworkInterface:web-01-nic
    ManagedDisk:web-01-data
    VirtualMachine:web-01

Relationships
    VNet --contained-in--> RG
    Subnet --contained-in--> VNet
    NIC --contained-in--> RG
    NIC --attached-to--> Subnet
    Disk --contained-in--> RG
    VM --contained-in--> RG
    VM --attached-to--> NIC
    VM --uses-disk--> Disk

Dependencies
    RG -> VNet
    RG -> NIC
    RG -> Disk
    RG -> VM
    VNet -> Subnet
    Subnet -> NIC
    NIC -> VM
    Disk -> VM
```

Engine then validates graph integrity and performs deterministic dependency analysis.

A valid topological ordering might be:

```text
1. ResourceGroup:prod-rg
2. VirtualNetwork:prod-vnet
3. ManagedDisk:web-01-data
4. Subnet:app
5. NetworkInterface:web-01-nic
6. VirtualMachine:web-01
```

More than one topological ordering may be mathematically valid when independent nodes exist. Engine must define deterministic tie-breaking so equivalent graphs produce equivalent ordering.

## Backend consumption

The completed Resource Graph is then supplied to an Integration-owned Backend.

For Bicep:

```text
Azure typed Resource Graph
        |
        v
Azure.Backend.Bicep
        |
        v
Bicep.Target.Abstractions
```

For Terraform:

```text
Azure typed Resource Graph
        |
        v
Azure.Backend.Terraform
        |
        v
Terraform.Target.Abstractions
```

Both Backends consume the same resolved Azure domain model.

A Backend must not return to YAML or Parsed Intent to recover VM size, resource-group identity, NIC reference, disk reference, or other accepted semantic state.

## Failure-case pressure tests

The following cases should be walked through before the Semantic Model and Resource Graph APIs are frozen.

### 1. Missing NIC

Intent:

```text
VM web-01 references web-01-nic
web-01-nic is not present
```

Expected ownership:

```text
Azure Integration
    accepts/materializes the typed NIC reference if locally valid

Engine reference resolution
    fails because the target identity does not exist
```

Expected result: deterministic unresolved-reference diagnostic.

### 2. Wrong resource type

Intent causes a VM `NetworkInterface` reference to identify a resource that exists but is not a `NetworkInterfaceResource`.

Expected ownership: Engine reference resolution.

Expected result: deterministic wrong-reference-type diagnostic.

### 3. Duplicate ResourceIdentity

Two Azure resources materialize to:

```text
azure.virtual-machine.prod-rg/web-01
```

Expected ownership: Engine identity registration.

Expected result: deterministic duplicate-identity diagnostic.

### 4. Forward reference

VM appears in the source before its NIC and Managed Disk.

Expected ownership: no error merely because of declaration order.

Expected result: identical resolved graph to the equivalent source with dependencies declared first.

### 5. Invalid Azure-local field

VM is missing a required `size`, or another Azure-local invariant fails.

Expected ownership: Azure Integration materialization/local validation.

Expected result: domain-local diagnostic before graph semantics are considered valid for that resource.

### 6. Missing dependency target

A semantic rule produces a dependency whose prerequisite resource is absent.

Expected ownership: Engine graph validation, with Integration provenance available to explain why the dependency was requested.

Expected result: deterministic graph diagnostic.

### 7. Dependency cycle

Semantic rules produce:

```text
A -> B
B -> A
```

Expected ownership: Engine graph validation.

Expected result: deterministic cycle diagnostic. Domain provenance should make the derived edges explainable without changing dependency identity.

### 8. Same VM name in different resource groups

Intent contains:

```text
prod-rg/web-01
dr-rg/web-01
```

Expected ownership:

```text
Azure Integration
    creates distinct canonical ResourceKeys

Engine
    accepts the identities because the complete ResourceIdentity values differ
```

This pressure-tests the decision that domain-specific scope belongs inside the Integration-owned ResourceKey rather than Engine's identity schema.

## Questions this scenario should answer

As the scenario is refined, each step should be used to answer:

- Does Engine ever need Azure-specific knowledge? If so, the boundary is probably wrong.
- Does Azure Integration start implementing its own graph registry, traversal, or ordering? If so, graph ownership is leaking.
- Does Backend ever need raw or Parsed Intent? If so, the Infrastructure IR is probably lossy.
- Does source ordering change semantic output? If so, Semantic Analysis is incorrectly coupled to syntax.
- Do we invent synthetic resources or relationships solely to make ordering work? If so, graph semantics are being polluted.
- Is a validation rule being performed by a component that does not own that concern?
- Are the eventual interfaces driven by this actual lifecycle rather than speculative generality?

## Current observations

The scenario supports the current architecture direction so far:

```text
Adapter
    source translation

Integration / Semantic Model
    typed materialization + domain semantics

Engine
    identity + resolution + graph mechanics

Backend
    domain-to-Target lowering

Target
    Target validation + emission
```

No public interface shape should be considered accepted solely because it appears in this document. The next step is to challenge this scenario case-by-case and use the results to define the minimum Semantic Model and Resource Graph APIs.