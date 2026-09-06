# Azure VM Pressure Test 03: Existing Resource References and Brownfield Lowering

> **Status:** Worked architecture pressure test
>
> This document tests whether existing infrastructure should be represented as full Resource Graph nodes or as external references that participate in semantic resolution but are lowered differently by the Backend / Target.

## Question

For an incremental build, should an existing Resource Group, Virtual Network, Subnet, NIC, or VM be represented as a normal Resource Graph node, or can the Engine model only the reference and allow the Backend / Target to decide how that existing dependency is represented?

The architecture must support brownfield scenarios from the beginning without forcing Engine to own deployment-technology-specific brownfield mechanics.

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

Greenfield and brownfield resources participate in the same semantic concerns:

```text
identity
references
relationships
dependency resolution
semantic validation
Backend lowering
```

The architectural difference is not whether the resource has semantic meaning. The difference is whether the Backend / Target should treat it as something managed by this compilation or as existing infrastructure that must only be referenced.

## Variant A: existing infrastructure as full graph nodes

In this model, Engine receives or materializes Resource Graph nodes for both existing and new resources.

Conceptually:

```text
ResourceGraph

Existing ResourceGroup:network-rg
Existing VirtualNetwork:network-rg/prod-vnet
Existing Subnet:network-rg/prod-vnet/app

Managed ResourceGroup:compute-rg
Managed NetworkInterface:compute-rg/web-02-nic
Managed VirtualMachine:compute-rg/web-02
```

The resulting semantic relationships remain normal:

```text
VirtualNetwork --contained-in--> network-rg
Subnet --contained-in---------> VirtualNetwork
NIC --attached-to-------------> Subnet
VM --attached-to--------------> NIC
VM --contained-in-------------> compute-rg
```

The dependency chain is also normal:

```text
existing network-rg
    -> existing VNet
    -> existing Subnet
    -> new NIC
    -> new VM

new compute-rg
    -> new VM
```

### Advantages

- All referenced infrastructure is represented uniformly in the graph.
- Relationship traversal is straightforward.
- Semantic validation can inspect existing-resource properties if they are available.
- Provenance and visualization can show the complete environment context.
- Backends receive one coherent graph model.

### Problems

- Engine now needs some way to distinguish resources that participate in semantics from resources that are managed by this compilation.
- Existing infrastructure may need to be fully described or discovered merely to create a valid graph node.
- The model risks making brownfield ergonomics depend on how much existing state the engineer can manually reproduce in Intent.
- If existing nodes are incomplete projections of real resources, it becomes unclear whether they are authoritative semantic resources or merely reference stubs.

## Variant B: existing infrastructure as external references

In this model, only resources managed by the compilation become full graph nodes.

Existing infrastructure is represented through resolved external references with stable domain identity and whatever domain context is needed for semantic analysis / lowering.

Conceptually:

```text
ResourceGraph nodes
    ResourceGroup:compute-rg
    NetworkInterface:web-02-nic
    VirtualMachine:web-02

External references
    azure.resource-group.network-rg
    azure.virtual-network.network-rg/prod-vnet
    azure.subnet.network-rg/prod-vnet/app
```

The NIC can still declare:

```text
NIC:web-02-nic
    --attached-to--> existing Subnet:network-rg/prod-vnet/app
```

and the Backend can lower that external dependency appropriately for the Target.

### Advantages

- Existing infrastructure does not need to masquerade as resources owned by the compilation.
- Engine does not need to emit or lifecycle-manage existing infrastructure.
- The Backend / Target remains responsible for target-native existing-resource mechanics.
- Intent can potentially reference an existing resource using much less detail than would be required to reconstruct the full resource.

### Problems

- The Resource Graph now contains edges whose targets are not normal graph nodes unless external references are treated as first-class graph participants.
- Semantic validation may need properties from the existing resource that are not present in the reference.
- Traversal, diagnostics, visualization, and provenance become more complex if graph nodes and external reference endpoints behave differently.
- Transitive dependency ordering through existing infrastructure may become misleading: an existing VNet and Subnet do not need to be "ordered" for creation, but a new NIC still depends on their existence.

## Key architectural distinction

The pressure test suggests that the important concept is not strictly:

```text
Resource node versus non-resource reference
```

but rather:

```text
semantic participation versus lifecycle ownership
```

An existing subnet has real semantic identity and participates in:

```text
reference resolution
relationship semantics
validation
Backend lowering
```

but it is not necessarily a deployment unit owned by this compilation.

This leads to a candidate model in which Engine can represent a semantic resource participant without deciding how that participant is managed by a deployment Target.

## Candidate direction: graph participant + disposition/context

A possible future shape is conceptually:

```text
ResourceParticipant
    Identity
    ResourceType
    semantic state
    disposition/context
```

where disposition might eventually express something like:

```text
managed by this compilation
existing / externally managed
```

The exact API and vocabulary are deliberately not decided here.

The important requirement is:

> Existing infrastructure may participate fully in semantic resolution without being treated as deployment output owned by the current compilation.

## Does Engine need to know greenfield versus brownfield?

Not necessarily as a compilation-wide mode.

A mixed scenario is common:

```text
existing Resource Group
existing VNet
existing Subnet
new NIC
new VM
```

Therefore a global binary value such as:

```text
mode = greenfield | brownfield
```

is insufficient as the semantic model.

Compilation context may still contain an informational workflow mode such as `incremental`, but graph semantics should not depend on a global greenfield/brownfield switch.

Instead, the semantic state of each participating resource/reference should carry enough information for Backend lowering to determine whether it represents new managed infrastructure or an existing dependency.

## Target-specific lowering remains downstream

Engine should not encode Terraform data blocks, Bicep `existing` declarations, ARM calculated resource IDs, CloudFormation import/reference mechanics, or other deployment-technology-specific existing-resource constructs.

The flow remains:

```text
Intent / context
    |
    v
Integration semantic analysis
    |
    v
Resource Graph
    existing + managed semantic participants
    |
    v
Backend
    chooses Target-level representation
    |
    v
Target
```

The same semantic relationship:

```text
NIC --attached-to--> Subnet
```

should remain valid regardless of whether the subnet is newly managed or pre-existing.

## Dependency semantics for existing resources

An existing resource does not need to be emitted before a managed resource, but the managed resource still requires the existing resource to be available.

Therefore "dependency" may need to distinguish at least conceptually between:

```text
provisioning order among managed resources

and

availability prerequisite involving an existing resource
```

This does not necessarily require two public edge types. It may be derivable from the participant disposition during graph analysis / Backend lowering.

For example:

```text
existing Subnet -> new NIC
```

means:

> The NIC requires the subnet to exist, but Engine must not schedule creation of the subnet because it is externally managed.

A topological ordering over managed resources can treat existing participants as already satisfied prerequisites while preserving the relationship for validation and lowering.

## Brownfield authoring pain point

Supporting existing-resource semantics does not by itself solve engineer effort.

A poor brownfield experience would require the engineer to fully redescribe every existing resource just so a new resource can reference it.

A future system may obtain existing-resource information from:

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

The architectural requirement is only that the Resource Graph and Backend boundaries do not prevent such enrichment later.

## Result

Variant A is too strong if it requires every existing dependency to be reconstructed as a complete managed-style graph node.

Variant B is too weak if external references cannot participate normally in relationships, semantic validation, diagnostics, and Backend lowering.

The preferred direction is therefore:

> Treat existing infrastructure as first-class semantic graph participants, while keeping lifecycle/disposition separate from resource identity and domain semantics.

Greenfield and brownfield should use the same identity, reference, relationship, semantic-validation, and Backend-lowering model.

The Target-specific representation of an existing dependency remains a Backend / Target concern.

## Open questions exposed by this test

- What is the minimum semantic information required for an existing graph participant?
- Does lifecycle/disposition belong on the resource participant, the compilation context, the reference, or separate graph metadata?
- Can a participant be identity-only, or must Domain Abstractions define a typed existing-resource representation?
- How does Engine validate semantic constraints that require properties from an existing resource when those properties were not supplied?
- Should existing participants be included in general graph traversal but excluded from managed-resource topological output ordering?
- How does Backend lowering distinguish "emit/create" from "reference existing" without contaminating Domain Abstractions with Target concepts?
- How should external state acquisition/enrichment plug in later without making semantic compilation dependent on live cloud connectivity?
