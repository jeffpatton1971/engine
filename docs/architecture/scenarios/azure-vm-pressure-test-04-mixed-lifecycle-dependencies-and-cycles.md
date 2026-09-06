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

A dependency-only edge remains valid in a mixed-lifecycle graph.

For example, if Azure Integration identifies a platform prerequisite with no useful enduring semantic relationship:

```text
IExistingResource A -> IManagedResource B
```

Engine may retain that structural dependency without inventing a relationship.

Because `A` is existing, the prerequisite is considered available for scheduling purposes. The Backend still receives enough information to represent the existing prerequisite appropriately for its Target.

Likewise:

```text
IManagedResource A -> IManagedResource B
```

participates normally in managed topological ordering.

## Cycle case 1: managed-only cycle

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

## Cycle case 2: managed/existing cycle

Consider:

```text
Existing Subnet -> Managed NIC
Managed NIC -> Existing Subnet
```

The first edge is reasonable: the NIC requires the existing subnet.

The second edge claims that the already-existing subnet requires the new NIC.

This is not merely an ordering inconvenience. It contradicts the lifecycle assertion that the subnet already exists independently of the current compilation.

Expected result:

```text
Engine graph validation
    -> invalid lifecycle dependency / cycle diagnostic
    -> compilation fails
```

An `IExistingResource` may be a prerequisite of a managed resource, but a managed resource SHOULD NOT be required for the existence of an existing participant in the current compilation.

This suggests a useful invariant:

> Existing resources may satisfy prerequisites for managed resources, but managed resources cannot be prerequisites for the asserted pre-existence of existing resources within the same compilation.

The exact diagnostic classification remains open.

## Cycle case 3: existing-only dependency cycle

```text
Existing A -> Existing B
Existing B -> Existing A
```

This case is different.

Because neither node is scheduled for creation, the cycle does not create a provisioning-order problem.

However, dependency edges among existing participants are usually unnecessary if all they are intended to express is historical provisioning order. Existing infrastructure is already present; relationships can preserve domain meaning without reconstructing its creation sequence.

Therefore the preferred rule is:

> Do not derive dependency edges between existing participants merely to reproduce how the existing environment would have been provisioned.

If an Integration nevertheless emits an existing-to-existing dependency because it represents a current availability invariant rather than historical ordering, Engine may retain it for semantic traversal. A cycle among only existing participants should not automatically invalidate managed provisioning unless a semantic rule explicitly declares such a cycle invalid.

This keeps Engine from rejecting a brownfield compilation merely because an imported/existing environment contains dependency information irrelevant to the resources being managed now.

## Cycle detection therefore has two views

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

Additional lifecycle validation catches invalid edges such as:

```text
Managed -> Existing
```

when that edge claims the managed resource is required for an existing participant to exist.

## Important consequence

A simple cycle detector over every dependency edge in the full mixed graph would be too aggressive.

For greenfield graphs, semantic dependency cycles and provisioning cycles are effectively the same problem because every node is managed.

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
2. Existing -> Managed dependencies are valid and represent already-satisfied availability prerequisites.
3. Managed -> Managed dependencies drive provisioning order normally.
4. Managed -> Existing dependencies are suspicious and generally invalid because they contradict asserted pre-existence; they require explicit semantic justification if ever supported.
5. Existing -> Existing dependency edges should not be generated merely to reproduce historical creation order.
6. Managed provisioning cycle detection operates over the managed-resource projection.
7. Engine may perform additional semantic/lifecycle validation over the full graph without automatically treating every existing-only cycle as a provisioning failure.

## Open questions

- Should `Managed -> Existing` be forbidden by the Engine contract or merely rejected by Integration semantic conformance unless explicitly allowed?
- Is a separate API required for semantic dependency traversal versus managed provisioning order, or can one graph expose both views?
- Should existing-only dependency edges be allowed at all in the initial contract?
- How should provenance explain a dependency that crosses a managed/existing lifecycle boundary?
- What terminology should distinguish an availability prerequisite from a provisioning-order constraint without creating two competing dependency abstractions?
