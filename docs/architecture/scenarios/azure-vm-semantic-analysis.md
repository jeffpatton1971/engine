# Azure VM Semantic Analysis Scenario

> **Status:** Worked architecture scenario
>
> This document is not an ADR. It is the parent scenario for the Azure pressure-test series and summarizes the conclusions reached by the focused pressure tests.

## Purpose

Exercise Engine's semantic-analysis ownership using a small Azure model without freezing public APIs prematurely.

## Resource set

```text
Resource Group
Virtual Network
Subnet
Network Interface
Managed Disk
Virtual Machine
```

The representative YAML/source shape is illustrative only; equivalent Adapters should produce equivalent semantic results.

## Accepted ownership model from the pressure tests

```text
Adapter
    source translation only

Azure Integration / Semantic Model
    Azure domain semantics
    canonical identity/scoping
    managed/existing typed materialization
    domain-local validation
    relationship/dependency meaning

Engine
    identity registration
    canonical reference resolution
    graph construction/integrity
    semantic traversal
    deterministic managed provisioning order

Azure Backend
    Azure -> Target lowering
    Target representability validation

Target
    Target-model validation
    emission
```

## Semantic lifecycle

```text
Parsed Intent
     |
     v
1. Materialization                  Azure Integration
     |
     v
2. Identity Registration            Engine
     |
     v
3. Reference Resolution             Engine
     |
     v
4. Relationship/Dependency Analysis Azure Integration
     |
     v
5. Graph Construction/Validation    Engine
     |
     v
Infrastructure IR / Resource Graph
```

Source declaration order has no semantic significance.

## Typed domain and lifecycle model

The pressure tests refined the original managed-resource-only examples.

Domain semantic contracts and lifecycle are orthogonal:

```text
ISubnet + IManagedResource
ISubnet + IExistingResource

INetworkInterface + IManagedResource
INetworkInterface + IExistingResource
```

Typed references target the domain contract:

```text
ResourceReference<ISubnet>
ResourceReference<INetworkInterface>
ResourceReference<IManagedDisk>
```

rather than a lifecycle-specific implementation.

## Canonical identity

```text
IntegrationId + ResourceType + ResourceKey
```

Azure Integration owns canonical ResourceKey construction and Azure scope interpretation. Engine enforces uniqueness without knowing Resource Groups, VNets, subscriptions, or other Azure-specific scoping rules.

Two `app` subnets in different VNets must therefore receive distinct Azure canonical keys.

## Validation findings

Validation ownership is:

```text
Azure Integration -> valid Azure intent/domain semantics
Engine            -> valid resolved graph
Azure Backend     -> representable by selected Target generation
Target            -> valid Target model
```

Independent user-correctable failures should be aggregated when safe.

Examples:

- required NIC field absent -> Azure Integration diagnostic;
- canonical NIC identity declared but absent -> Engine unresolved-reference diagnostic;
- duplicate ResourceIdentity -> Engine diagnostic;
- ambiguous source name/scope preventing canonical identity -> Azure Integration diagnostic;
- valid Azure feature unsupported by Target generation -> Backend representability diagnostic;
- malformed Terraform/Bicep/etc. Target model -> Target diagnostic.

## Relationships and dependencies

Representative semantic relationships:

```text
VNet --contained-in--> Resource Group
Subnet --contained-in--> VNet
NIC --attached-to--> Subnet
VM --attached-to--> NIC
VM --uses-disk--> Managed Disk
VM --contained-in--> Resource Group
```

Representative prerequisites:

```text
Resource Group -> VNet
VNet -> Subnet
Subnet -> NIC
NIC -> VM
Managed Disk -> VM
VM Resource Group -> VM
```

Integration determines whether a dependency exists and its direction. Engine does not infer direction from lifecycle.

Relationships express domain meaning; dependencies express prerequisites/order. Integrations must not invent relationships or synthetic containers solely for ordering.

## Cross-resource-group result

A network chain can live in one Resource Group while the VM lives in another:

```text
network-rg -> VNet -> Subnet -> NIC -> VM
compute-rg -----------------------> VM
```

Identity-based dependency traversal preserves the correct prerequisite chain without redundant transitive edges or Engine knowledge of Resource Group boundaries.

## Brownfield result

Existing and managed nodes coexist in the same graph:

```text
Existing Subnet -> Managed NIC -> Managed VM
```

Existing prerequisites remain visible in semantic traversal and Backend lowering but are treated as already satisfied for managed provisioning.

No global greenfield/brownfield mode is required for graph semantics.

## Dependency-cycle result

Managed-to-managed cycles are provisioning failures detected by Engine.

Mixed lifecycle does not create a new dependency-direction rule. A dependency must first be semantically valid according to Azure Integration; Engine does not reject or invent edges merely because an endpoint is managed/existing.

## Semantically lossless Backend handoff

After Semantic Analysis succeeds, Parsed Intent is finished as a source of domain meaning.

Azure-to-Terraform and Azure-to-Bicep Backends should be able to lower the same resolved graph using only:

```text
Azure Domain Abstractions
Resource Graph
per-compilation Context
Target Abstractions
```

If a Backend must ask what the original YAML/JSON said, the IR is lossy.

Semantic losslessness does not require preservation of source formatting/comments/order.

## Compilation Context

Compilation Context carries defined information belonging to one Intent/compilation rather than one resource. It may include subscription/account, regional defaults, credentials/profile selection, naming context, workflow metadata, or Target selection.

Context is per compilation, not global Engine state, and must not expose raw Parsed Intent as an escape hatch.

## Target validation result

A valid Azure graph does not imply every Target can represent it.

```text
Azure Integration
    domain correctness

Engine
    graph correctness

Azure Backend
    Target representability

Target
    Target-model correctness
```

This keeps valid Azure semantics separate from limitations of a particular Target contract generation.

## Pressure-test documents

1. `azure-vm-pressure-test-01-reference-resolution.md`
2. `azure-vm-pressure-test-02-cross-resource-group-dependencies.md`
3. `azure-vm-pressure-test-03-existing-resource-references.md`
4. `azure-vm-pressure-test-04-mixed-lifecycle-dependencies-and-cycles.md`
5. `azure-vm-pressure-test-05-scoped-resource-identity.md`
6. `azure-vm-pressure-test-06-lossless-ir-backend-handoff.md`
7. `azure-vm-pressure-test-07-target-validation.md`

## Current result

The Azure series supports the architecture without requiring Engine to acquire Azure-specific semantics.

It also produced concrete contract requirements that were not obvious at the start: managed/existing lifecycle contracts, domain-oriented typed references, scoped canonical identity, semantic versus managed dependency views, semantically lossless IR, per-Intent compilation context, validation aggregation, and a Backend representability boundary.

The next design step should shape minimal public contracts from these findings rather than adding new abstraction without a concrete pressure case.