# Azure VM Pressure Test 03: Existing Resource References and Brownfield Lowering

> **Status:** Worked architecture pressure test
>
> This document tests how existing infrastructure participates in semantic analysis and Backend lowering without being treated as infrastructure the current compilation should create.

## Question

For an incremental build, how should an existing Resource Group, Virtual Network, Subnet, NIC, VM, or other resource participate in the Resource Graph?

The architecture must support brownfield scenarios from the beginning without forcing Engine to own deployment-technology-specific brownfield mechanics or requiring engineers to fully redescribe existing infrastructure.

## Scenario

An engineer wants to deploy a new VM into an existing Azure network.

Conceptually:

```text
Existing
    ResourceGroup: network-rg
    VirtualNetwork: prod-vnet
    Subnet: app

New
    ResourceGroup: compute-rg
    NetworkInterface: web-02-nic
    VirtualMachine: web-02
```

The new NIC attaches to the existing subnet. The new VM attaches to the new NIC.

## Invariant

Both managed Resources and existing ResourceParticipants participate in:

```text
identity
references
relationships
dependency resolution
semantic validation
Backend lowering
```

The architectural difference is lifecycle intent: a Resource represents infrastructure described for management by the current compilation, while a ResourceParticipant represents infrastructure that already exists and participates semantically but must not be created by this compilation.

## Proposed semantic distinction

The pressure test adopts `ResourceParticipant` as the working type for existing infrastructure.

Conceptually:

```csharp
public interface IResourceParticipant
{
    ResourceIdentity Identity { get; }
    ResourceType Type { get; }
}
```

The exact API remains illustrative.

A `ResourceParticipant` intentionally carries the minimum semantic information necessary to participate in the graph. It does not require the complete property set of the corresponding managed Domain Resource unless a specific semantic rule genuinely requires additional information.

The type itself carries the core lifecycle meaning:

> This infrastructure exists and may be referenced, related, validated, and lowered, but it is not something this compilation should create.

Engine therefore does not need a generic `Existing = true` flag on every managed resource merely to distinguish brownfield resources.

## Resource versus ResourceParticipant

Conceptually:

```text
Resource
    full Integration-owned typed desired resource
    managed by this compilation

ResourceParticipant
    existing infrastructure participant
    minimum identity/type/domain context required
    not created by this compilation
```

Both participate in graph semantics.

For this scenario:

```text
ResourceParticipant ResourceGroup:network-rg
ResourceParticipant VirtualNetwork:network-rg/prod-vnet
ResourceParticipant Subnet:network-rg/prod-vnet/app

Resource ResourceGroup:compute-rg
Resource NetworkInterface:compute-rg/web-02-nic
Resource VirtualMachine:compute-rg/web-02
```

## Relationships remain unchanged

The semantic relationship should not change merely because one endpoint already exists.

```text
VirtualNetwork --contained-in--> network-rg
Subnet --contained-in---------> VirtualNetwork
NIC --attached-to-------------> Subnet
VM --attached-to--------------> NIC
VM --contained-in-------------> compute-rg
```

The NIC's relationship to the subnet remains `attached-to` whether the subnet is a new Resource or an existing ResourceParticipant.

This is important because greenfield versus brownfield is not a different Azure domain model.

## References remain typed

A managed resource may reference a ResourceParticipant through the same domain relationship expected for the corresponding resource type.

Conceptually, a NIC still requires a subnet identity/type. If the resolved target is a `ResourceParticipant` representing an Azure Subnet, Engine knows that:

```text
- the reference exists semantically;
- the relationship can be resolved;
- the subnet is not scheduled for creation;
- the Backend must represent the subnet as existing infrastructure for its Target.
```

The exact generic reference API may need to support both managed Resources and ResourceParticipants of the expected domain type. That API question remains open.

## Dependency semantics

The dependency chain can include ResourceParticipants normally:

```text
ResourceParticipant network-rg
    -> ResourceParticipant prod-vnet
    -> ResourceParticipant app subnet
    -> Resource web-02-nic
    -> Resource web-02

Resource compute-rg
    -> Resource web-02
```

The graph semantics remain meaningful:

```text
existing Subnet -> new NIC
```

means:

> The NIC requires this subnet to exist.

Because the prerequisite is a `ResourceParticipant`, Engine does not interpret the edge as an instruction to create or schedule that subnet. It is an already-existing prerequisite.

For managed-to-managed dependencies such as:

```text
new NIC -> new VM
```

Engine uses the dependency for provisioning/topological ordering normally.

This allows one dependency model to represent both availability prerequisites and managed-resource ordering without introducing synthetic relationships or separate brownfield dependency types.

## Topological behavior

For managed output ordering, ResourceParticipants are treated as already-satisfied prerequisites.

Conceptually:

```text
Existing participants:
    network-rg
    prod-vnet
    app subnet

Managed ordering:
    compute-rg
    web-02-nic
    web-02
```

The NIC cannot be semantically valid unless its subnet participant resolves, but the subnet itself is not emitted into the managed provisioning order.

The exact Resource Graph API may still expose ResourceParticipants in general traversal so diagnostics, visualization, relationship queries, and Backend lowering can see the complete semantic context.

## Semantic validation

`ResourceParticipant` should carry only the minimum information required to support the semantic rules that apply to it.

For many references that may be only:

```text
ResourceIdentity
ResourceType
```

For example, if an Azure NIC only needs the identity of an existing subnet for Backend lowering, the engineer should not need to provide the subnet's entire address space, delegation configuration, route table, and other properties.

If a semantic rule genuinely requires additional existing-resource information, the Integration may require that specific information or obtain it through a future enrichment mechanism.

The architecture SHALL NOT require a complete managed-resource description merely because the referenced infrastructure already exists.

## Target-specific lowering remains downstream

Engine should not encode Terraform data blocks, Bicep `existing` declarations, ARM resource-ID calculations, CloudFormation import/reference mechanics, or other deployment-technology-specific existing-resource constructs.

The flow remains:

```text
Intent
    |
    v
Integration semantic analysis
    |
    v
Resource Graph
    Resources + ResourceParticipants
    |
    v
Backend
    sees participant type and domain identity
    chooses Target-level representation
    |
    v
Target
```

For example, when a Backend receives an Azure Subnet `ResourceParticipant`, it knows that the subnet is semantically required but is not a managed output resource. The Backend / Target pair determines how that fact becomes Terraform, Bicep, ARM, CloudFormation, or another Target representation.

## Does Engine need a greenfield/brownfield mode?

Not for core graph semantics.

A compilation may be mixed:

```text
existing Resource Group
existing VNet
existing Subnet
new NIC
new VM
```

Therefore a global binary mode is not sufficient to determine lifecycle behavior.

Compilation context may still carry workflow information such as `incremental`, but the Resource versus ResourceParticipant type distinction is sufficient for the graph to know which infrastructure is existing and which is managed by the compilation.

## Brownfield authoring pain point

Supporting `ResourceParticipant` does not by itself solve engineer effort, but it prevents the architecture from requiring unnecessary detail.

A brownfield Intent should be able to identify existing infrastructure with the minimum information required for identity, semantic validation, and Backend lowering.

A future system may obtain additional existing-resource information from:

```text
explicit Intent
cloud/API lookup
inventory
CMDB
Terraform state
saved environment snapshot
generated starter Intent
custom engineer tooling
```

Those acquisition/enrichment mechanisms should not be required to define the core Semantic Model contract now.

## Result

The pressure test adopts the following working direction:

> `ResourceParticipant` is a first-class semantic graph type representing infrastructure that already exists.

A ResourceParticipant:

- has stable `ResourceIdentity` and `ResourceType`;
- carries only the minimum additional semantic information required;
- participates in typed references;
- participates in semantic relationships;
- participates in dependency resolution;
- participates in semantic validation;
- is visible to Backend lowering;
- is not created or managed by the current compilation by nature of its type.

Managed Resources and ResourceParticipants use the same semantic relationship model. The Target-specific representation of a ResourceParticipant remains a Backend / Target concern.

This allows greenfield and brownfield infrastructure to coexist in one Resource Graph without a global greenfield/brownfield switch and without forcing engineers to redescribe complete existing environments.

## Open questions exposed by this test

- What common base contract, if any, should `IResource` and `IResourceParticipant` share?
- How should `ResourceReference<TResource>` express that its target may resolve to either a managed resource or a ResourceParticipant representing the same domain resource type?
- How is the domain resource type of a ResourceParticipant associated with Domain Abstractions without requiring a full managed-resource instance?
- What is the exact minimum participant contract beyond `Identity` and `Type`?
- How does Engine expose ResourceParticipants in traversal and queries while excluding them from managed-resource provisioning order?
- How does an Integration request additional semantic information when identity/type alone is insufficient to validate an existing participant?
- How should external state acquisition/enrichment plug in later without making normal semantic compilation dependent on live cloud connectivity?
