# Engine Glossary

> Status: Working Draft

This glossary defines the current canonical terminology for the Engine architecture exploration.

The terms below are intentionally provisional. They exist to make architecture discussions precise while the design is still being challenged and refined.

## Intent

A declarative description of desired infrastructure state or capability, independent of a specific deployment representation.

Intent describes what infrastructure should exist, not how a particular target language expresses it.

## Source

An external representation containing infrastructure intent.

Examples include YAML, JSON, API payloads, spreadsheets, or another system's data model.

## Adapter

A component that converts a supported Source representation into the Engine's parsed Intent representation.

Adapters parse and normalize input. They do not apply infrastructure semantics or generate deployment artifacts.

## Semantic Model

The authoritative definition of resource types and their meaning within an infrastructure domain.

A Semantic Model may define resource types, properties, types, constraints, defaults, identities, and valid relationships.

Examples may include VCFA, Azure, AWS, or another infrastructure domain.

## Semantic Analysis

The Engine phase that resolves and validates parsed Intent against one or more Semantic Models.

Semantic Analysis may include type resolution, identity resolution, defaulting, validation, reference resolution, relationship resolution, dependency analysis, and diagnostic production.

Semantic Analysis produces a resolved Infrastructure IR.

## Resource

A typed unit of infrastructure intent with stable identity, properties, and semantic relationships.

A Resource represents infrastructure meaning and is independent of a specific deployment target.

## Resource Graph

The graph structure formed by Resources and their resolved semantic relationships.

The graph expresses references and dependencies explicitly and deterministically rather than inferring them from document structure or collection order.

## Infrastructure IR

The Engine's canonical intermediate representation of resolved infrastructure intent.

The Infrastructure IR is represented as a deterministic Resource Graph and is independent of any deployment target.

The IR is the canonical semantic boundary between the Engine's analysis phases and target-specific lowering.

## Backend

A component that lowers Infrastructure IR into a target-specific intermediate representation.

A Backend understands how infrastructure semantics map to a target deployment model.

Examples may include Terraform, Bicep, CloudFormation, or other deployment targets.

## Target IR

An intermediate representation describing a deployment target without requiring the Engine's infrastructure semantics.

Examples may include Terraform IR, Bicep IR, or CloudFormation IR.

A Target IR is suitable for one or more Emitters.

## Emitter

A component that serializes a Target IR into physical deployment artifacts.

Emitters handle representation and physical emission. They do not make infrastructure-semantic decisions.

Examples include an HCL emitter, Bicep source emitter, JSON emitter, or YAML emitter.

## Artifact

A deterministic physical output produced by an Emitter.

Examples include source files, configuration files, manifests, or other target-native deployment artifacts.

## Artifact Bundle

The complete, versioned result of a compilation.

An Artifact Bundle may include generated artifacts, manifest metadata, diagnostics, source and target version information, and optional graph or provenance information.

## Compilation

The deterministic process of transforming infrastructure Intent into an Artifact Bundle.

Compilation includes adaptation, semantic analysis, Infrastructure IR construction, backend lowering, and emission.

## Engine

The runtime that coordinates adapters, semantic analysis, Infrastructure IR construction, backends, emitters, diagnostics, and artifact production.

The Engine is target-independent. Terraform, Bicep, CloudFormation, and other deployment representations are target concerns rather than Engine semantics.

## Cloud-Native

An operating and design posture rather than a requirement to use Kubernetes or microservices.

For this project, cloud-native principles include stateless execution, declarative contracts, immutable and versioned artifacts, portable runtime packaging, structured diagnostics, observability, API and CLI parity, and independently distributable extensions where useful.

## Provider

An intentionally avoided Engine architecture term because it is overloaded across cloud providers, Terraform providers, and service providers.

Use Semantic Model when referring to infrastructure-domain semantics, Backend when referring to lowering into a deployment target, and the specific external technology name when referring to an external provider.