# Azure VM Pressure Test 04: Mixed-Lifecycle Dependencies and Cycle Detection

> **Status:** Worked architecture pressure test
>
> This document tests dependency traversal, dependency-only edges, cycle detection, and managed-resource ordering when the Resource Graph contains both managed resources and existing resource participants.

## Working type model

The pressure test assumes two lifecycle markers sharing domain contracts:

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

Domain contracts remain orthogonal to lifecycle:

```text
ISubnet + IManagedResource
ISubnet + IExistingResource

INetworkInterface + IManagedResource
INetworkInterface + IExistingResource
```

A typed reference targets the domain contract rather than a lifecycle-specific implementation:

```csharp
ResourceReference<ISubnet>
```

The exact API remains illustrative.

## Scenario

Existing networking infrastructure:

```text
IExistingResource ResourceGroup:network-rg
IExistingResource VirtualNetwork:network-rg/prod-vnet
IExistingResource Subnet:network-rg/prod-vnet/app
```

Managed infrastructure:

```text
IManagedResource ResourceGroup:compute-rg
IManagedResource NetworkInterface:compute-rg/web-02-nic
IManagedResource VirtualMachine:compute-rg/web-02
```

Semantic relationships:

```text
VNet --contained-in--> network-rg
Subnet --contained-in--> VNet
NIC --attached-to--> Subnet
VM --attached-to--> NIC
VM --contained-in--> compute-rg
```

Relevant prerequisite facts:

```text
existing Subnet -> managed NIC
managed NIC -> managed VM
managed compute-rg -> managed VM
```

## Dependency traversal

Engine should allow general dependency traversal across both lifecycle kinds.

For example, asking for prerequisites of `web-02` may yield:

```text
web-02
    <- web-02-nic      [managed]
        <- app subnet  [existing]

web-02
    <- compute-rg      [managed]
```

This is useful for diagnostics, explainability, visualization, and Backend lowering.

The fact that a prerequisite is existing does not remove it from semantic dependency traversal.

## Managed provisioning order

Provisioning order is a projection of the dependency graph onto managed resources.

Existing participants are treated as already-satisfied prerequisites rather than scheduled units.

For the scenario above, managed ordering is:

```text
compute-rg
web-02-nic
web-02
```

subject to deterministic tie-breaking where independent managed nodes exist.

The existing subnet does not appear in managed output ordering, but its edge to the NIC remains semantically visible.

The governing distinction is:

> Dependency traversal operates over the semantic graph; provisioning order operates over managed resources while treating existing prerequisites as already satisfied.

## Dependency-only edges

A dependency-only edge remains valid in a mixed-lifecycle graph when the Integration's domain semantics legitimately produce it.

For example:

```text
IExistingResource A -> IManagedResource B
```

may represent an availability prerequisite with no useful enduring semantic relationship.

Likewise:

```text
IManagedResource A -> IManagedResource B
```

may participate normally in managed topological ordering.

Engine does not decide whether an edge is semantically valid merely from lifecycle type. The Integration owns whether a dependency exists and its direction.

## Managed-resource cycles

```text
Managed A -> Managed B
Managed B -> Managed A
```

This is an ordinary provisioning cycle.

Expected result:

```text
Engine graph validation
    -> cycle diagnostic
    -> compilation fails
```

## Mixed-lifecycle cycles

A mixed-lifecycle cycle should not be invented as a synthetic Engine rule. Each edge must first be justified by Integration semantics.

For example, in the Azure networking scenario:

```text
Existing Subnet -> Managed NIC
```

is meaningful because the NIC requires the subnet.

The reverse edge:

```text
Managed NIC -> Existing Subnet
```

is not a valid Azure dependency in this scenario because a subnet does not depend on the NIC to exist. Azure Integration should therefore never produce that edge.

The governing rule is:

> Integration determines whether a dependency exists and its direction. Engine validates and operates on the resulting graph; lifecycle type only affects provisioning behavior.

If another infrastructure domain legitimately produces a Managed -> Existing dependency, Engine should not reject it merely because of lifecycle direction. Its validity belongs to that Integration's semantic contract.

## Existing-only dependencies

Dependency edges among existing participants should not be generated merely to reproduce historical creation order. Existing infrastructure is already present, and relationships can preserve domain meaning without reconstructing how it was originally provisioned.

If an Integration emits an existing-to-existing dependency because it represents a current availability invariant, Engine may retain it for semantic traversal.

Because neither node is scheduled for creation, such edges do not participate in managed provisioning order unless they affect a path to managed resources in a semantically meaningful way.

## Cycle detection has two views

The pressure test suggests separating:

```text
Semantic dependency graph
    all prerequisite facts across managed + existing nodes

Managed provisioning dependency graph
    managed nodes only
    existing prerequisites treated as satisfied boundaries
```

Engine can use the semantic graph for traversal, diagnostics, and Backend context.

Engine uses the managed provisioning graph for deterministic topological ordering and ordinary provisioning-cycle detection.

A full-graph semantic cycle may still be diagnostically relevant, but Engine should not infer domain invalidity from lifecycle direction alone. Any semantic-cycle rule beyond managed provisioning cycles must be justified by the Integration's domain semantics.

## Important consequence

For greenfield graphs, semantic dependency cycles and provisioning cycles are often effectively the same problem because every node is managed.

For brownfield/mixed graphs, existing participants create a boundary:

```text
existing prerequisite
    -> managed resource
```

is semantically meaningful but does not represent a scheduled creation step for the prerequisite.

Therefore Engine should not equate "all dependency edges" with "all nodes that must participate in topological provisioning order."

## Result

The managed/existing type split survives the pressure test with one important refinement:

> Dependency semantics are shared across managed and existing nodes, but provisioning order is a managed-resource projection of the graph.

Working rules:

1. Managed and existing nodes both participate in dependency traversal.
2. Existing -> Managed dependencies are valid when produced by Integration semantics and represent already-satisfied availability prerequisites for provisioning.
3. Managed -> Managed dependencies drive provisioning order normally.
4. Managed -> Existing dependencies are neither inherently valid nor inherently invalid; the Integration owns whether such an edge has real domain meaning.
5. Existing -> Existing dependency edges should not be generated merely to reproduce historical creation order.
6. Managed provisioning cycle detection operates over the managed-resource projection.
7. Engine does not reverse, invent, or reject dependency direction based solely on lifecycle type.

## Open questions

- Is a separate API required for semantic dependency traversal versus managed provisioning order, or can one graph expose both views?
- Should existing-only dependency edges be allowed in the initial contract or simply tolerated when emitted by a conformant Integration?
- How should provenance explain a dependency that crosses a managed/existing lifecycle boundary?
- What terminology should distinguish an availability prerequisite from a provisioning-order constraint without creating two competing dependency abstractions?
