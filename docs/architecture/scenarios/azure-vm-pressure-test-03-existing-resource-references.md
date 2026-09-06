# Azure VM Pressure Test 03: Existing Resource References and Brownfield Lowering

> **Status:** Worked architecture pressure test
>
> This document tests how existing infrastructure participates in semantic analysis and Backend lowering without being treated as infrastructure the current compilation should create.

## Question

For an incremental/brownfield build, how should an existing Resource Group, Virtual Network, Subnet, NIC, VM, or other resource participate in the Resource Graph?

## Scenario

```text
Existing
    ResourceGroup: network-rg
    VirtualNetwork: prod-vnet
    Subnet: app

Managed
    ResourceGroup: compute-rg
    NetworkInterface: web-02-nic
    VirtualMachine: web-02
```

The managed NIC attaches to the existing subnet. The managed VM attaches to the managed NIC.

## Refined lifecycle model

This pressure test originally used `ResourceParticipant` as a proposed CLR type for existing infrastructure. Later pressure testing refined that idea.

The accepted direction is now two orthogonal axes:

```text
Domain semantic contract              Lifecycle
------------------------              ---------
ISubnet                    +           IManagedResource
ISubnet                    +           IExistingResource

IVirtualMachine            +           IManagedResource
IVirtualMachine            +           IExistingResource
```

Conceptually:

```csharp
public interface IResourceNode
{
    ResourceIdentity Identity { get; }
    ResourceType Type { get; }
}

public interface IManagedResource : IResourceNode
{
}

public interface IExistingResource : IResourceNode
{
}
```

`ResourceParticipant` remains useful as an architectural description for anything participating in the semantic graph, but is not currently required as a distinct public CLR type.

## Invariant

Both managed and existing nodes participate in:

```text
identity
references
relationships
dependency resolution
semantic validation
Backend lowering
```

The difference is lifecycle intent:

- `IManagedResource` represents infrastructure the current compilation intends to manage;
- `IExistingResource` represents infrastructure asserted to already exist and is not scheduled for creation by this compilation.

No generic `Existing = true` flag is required.

## Existing-resource shape

An existing domain implementation intentionally carries the minimum semantic information necessary for the supported operation.

For example:

```csharp
public sealed record ExistingSubnet
    : ISubnet, IExistingResource
{
    public required ResourceIdentity Identity { get; init; }
    public required ResourceType Type { get; init; }
}
```

If identity/type are sufficient for semantic validation and Backend lowering, the engineer should not have to redescribe address spaces, delegations, route tables, or other properties.

If a concrete semantic or Target-lowering requirement needs additional Azure-domain information, that information must be supplied explicitly or by a future enrichment mechanism before Backend lowering.

## Typed references remain domain-oriented

A NIC references the semantic thing it needs:

```csharp
ResourceReference<ISubnet> Subnet
```

The reference may resolve to either:

```text
ISubnet + IManagedResource
```

or:

```text
ISubnet + IExistingResource
```

The NIC does not need two properties or a union merely because the subnet lifecycle differs.

## Relationships remain unchanged

```text
VirtualNetwork --contained-in--> network-rg
Subnet --contained-in---------> VirtualNetwork
NIC --attached-to-------------> Subnet
VM --attached-to--------------> NIC
VM --contained-in-------------> compute-rg
```

Greenfield versus brownfield is not a different Azure semantic model.

## Dependency semantics

The meaningful chain is:

```text
Existing Subnet -> Managed NIC -> Managed VM
Managed compute-rg ------------> Managed VM
```

The existing subnet is a semantic prerequisite for the NIC but is already satisfied for provisioning purposes.

Lifecycle does not determine whether an edge exists or its direction. Azure Integration semantics do. A subnet requires no reverse dependency on the NIC merely because the NIC is managed.

## Provisioning order

Semantic dependency traversal includes existing and managed nodes. Managed provisioning order schedules only managed nodes while treating existing prerequisites as satisfied boundaries.

Conceptually:

```text
Semantic graph
    Existing Subnet -> Managed NIC -> Managed VM

Managed provisioning projection
    Managed NIC -> Managed VM
```

The existing subnet remains visible for diagnostics, visualization, graph traversal, and Backend lowering.

## Target-specific lowering remains downstream

Engine does not encode Terraform data lookups, Bicep existing-resource declarations, ARM resource-ID expressions, CloudFormation import/reference mechanics, or equivalent Target constructs.

The flow remains:

```text
Intent
    |
    v
Integration semantic analysis
    |
    v
Resource Graph
    managed + existing typed domain nodes
    |
    v
Backend
    domain lifecycle + Target contract
    |
    v
Target
```

## No global greenfield/brownfield mode

A compilation may mix managed and existing resources, so a global binary mode cannot determine resource lifecycle.

Per-compilation context may still carry workflow metadata such as `incremental`, but lifecycle behavior comes from the resource's managed/existing contract.

## Brownfield authoring

The architecture should minimize brownfield authoring burden rather than requiring engineers to fully reproduce an existing environment.

Additional existing-resource information could eventually come from explicit Intent, cloud/API lookup, inventory, CMDB, state, snapshots, generated starter Intent, or custom tooling. No enrichment mechanism is required in the core Semantic Model contract yet.

## Result

The pressure test now resolves to:

> Domain semantic type and lifecycle are orthogonal. Existing infrastructure participates in the same graph semantics as managed infrastructure through `IExistingResource`, while typed references target the shared domain contract.

This supports mixed greenfield/brownfield graphs without duplicate relationship models, lifecycle-specific references, or global modes.

## Remaining questions

- What exact common `IResourceNode` contract is required beyond `Identity` and `Type`?
- How is a domain contract such as `ISubnet` associated with canonical `ResourceType` metadata?
- What minimum properties must an existing implementation expose when identity/type alone is insufficient?
- How should future enrichment work without making normal compilation dependent on live cloud connectivity?