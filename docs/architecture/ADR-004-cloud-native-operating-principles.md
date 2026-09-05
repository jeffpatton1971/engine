# ADR-004: Cloud-Native Operating Principles

- **Status:** Proposed
- **Date:** 2026-09-04
- **Decision Type:** Architecture exploration

## Context

The term "cloud-native" can refer to several different ideas: cloud-provider-native deployment technologies, modern application operating practices, Kubernetes-centric architectures, or simply software intended to run in cloud environments.

For Engine, the useful direction is not to mandate Kubernetes or a distributed system. It is to adopt operating and design principles that make the compiler portable, deterministic, automatable, observable, and suitable for modern cloud/platform workflows.

There is a related but separate target concern: a cloud-native deployment representation such as Bicep or CloudFormation may be a better backend for a particular domain than Terraform. The Engine architecture should allow that choice without conflating it with how the Engine itself is hosted.

## Proposed decision

Adopt **cloud-native operating principles** while keeping Engine Core independent of any required orchestrator or cloud platform.

The initial principles are:

### Stateless compilation

A compilation should be determined by explicit inputs, extension versions, Engine version, and compilation options.

The Engine should not require conversational/session state or hidden customer state to compile intent.

### Declarative contracts

Intent, semantic definitions, extension capabilities, compilation requests, diagnostics, and artifact manifests should use explicit versioned contracts where practical.

### Immutable and versioned outputs

A completed compilation produces an Artifact Bundle that represents an immutable result.

Given equivalent normalized inputs, semantic definitions, Engine version, backend/emitter versions, and options, compilation should be deterministic.

### Portable runtime

The same compiler should be usable from multiple execution surfaces without changing compilation semantics.

Potential surfaces include:

- CLI;
- API;
- container;
- CI/CD pipeline task;
- local developer tooling;
- scheduled or orchestrated job.

### API and CLI parity

The CLI and API should expose the same fundamental compiler capabilities rather than becoming independent implementations.

### Structured diagnostics

Diagnostics are part of the compiler contract and should be machine-readable, stable, and useful across local development, APIs, pipelines, and automation.

### Observability

Runtime surfaces should support appropriate logs, metrics, traces, correlation identifiers, and compilation metadata without allowing observability concerns to change compilation results.

### Independently distributable extensions

Adapters, Semantic Models, Backends, and Emitters should be capable of independent distribution where useful.

OCI artifacts are one possible future distribution mechanism but are not required by this decision.

## Explicit non-decisions

This ADR does **not** require:

- Kubernetes;
- microservices;
- service mesh;
- operators;
- controllers;
- CRDs;
- event-driven architecture;
- remote plugin execution;
- a database;
- a particular public cloud;
- OCI-based extension distribution.

These technologies may be introduced later when justified by concrete requirements.

## Cloud-native deployment targets

Backend selection is independent of Engine hosting.

For example, Azure infrastructure intent could reasonably lower to either:

```text
Azure Semantic Model -> Terraform Backend -> Terraform IR -> HCL
```

or:

```text
Azure Semantic Model -> Bicep Backend -> Bicep IR -> Bicep source
```

The preferred target can therefore reflect the capabilities and native ecosystem of the infrastructure domain without changing Engine Core.

## Rationale

This interpretation captures the useful properties of cloud-native software without prematurely introducing distributed-system complexity.

A deterministic compiler can remain a single process and still be highly cloud-native: stateless, containerizable, observable, API-driven, automation-friendly, and horizontally scalable at the execution layer.

## Consequences

### Positive

- Engine can run locally and in cloud-hosted environments with the same semantics.
- Hosting architecture can evolve independently from compiler architecture.
- CI/CD and GitOps-style workflows are natural consumers of immutable Artifact Bundles.
- The project avoids requiring infrastructure complexity before scale or operational needs justify it.

### Negative / risks

- "Cloud-native" remains an overloaded term and must be qualified in project documentation.
- Stateless compilation may require external systems to own state that some future use cases need.
- Extension distribution and runtime isolation may eventually require more infrastructure than the initial in-process model.

## Guardrails

Cloud-native technologies should be introduced to solve demonstrated operational requirements, not as architecture goals in themselves.

Compiler correctness, deterministic behavior, portability, and extension contracts take precedence over any particular hosting technology.

## Open questions

- Should deterministic compilation eventually use content-addressable build identities?
- Which provenance information belongs in the Artifact Bundle?
- What is the minimum observability contract for Engine Core versus API/CLI hosts?
- How should secrets and credentials be represented without becoming compilation state?
- Should extension packaging eventually adopt OCI artifacts or another registry model?