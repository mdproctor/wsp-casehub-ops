# Level 04: Fleet Scaling — Deploy 3 LLM Instances

## Summary

Scale the hello-llm topology from a single LLM app to a fleet of 3
differentiated LLM instances. Same app image, different per-instance
configuration, all managed as desired state.

Part of epic #131. Builds on Level 01 (hello-llm) — same 3 platform
services (manifest, eidos, engine), replaces the single `my-llm` node
with a parameterized fleet.

## What This Level Teaches

- **Data-driven fleet composition:** The `data` block declares fleet
  membership as structured data (CSV), not duplicated YAML nodes
- **`forEach` expansion:** A single node template produces N instances,
  each with its own `applicationId` and config
- **Per-instance parameterization:** Each fleet member gets different
  capability config via CSV column values
- **Parameter flow to eidos:** Topology config entries travel through
  deployment registration, making them available to eidos for
  differentiated agent identity generation

## Deliverables

### 1. Topology YAML — `04-llm-fleet.yaml`

Location: `topology-tests/src/test/resources/topologies/deployment/04-llm-fleet.yaml`

```yaml
desiredState:
  namespace: casehub
  name: llm-fleet

data:
  llm-fleet:
    inline: |
      name:STRING,capability:STRING
      analyst,deep
      coder,fast
      assistant,balanced

nodes:
  manifest:
    type: casehub_application
    spec:
      applicationId: manifest
      image: casehubio/manifest:latest
      deploymentTarget: claudony
      replicas: 1

  eidos:
    type: casehub_application
    dependsOn: [manifest]
    spec:
      applicationId: eidos
      image: casehubio/eidos:latest
      deploymentTarget: claudony
      replicas: 1

  engine:
    type: casehub_application
    dependsOn: [eidos]
    spec:
      applicationId: engine
      image: casehubio/engine:latest
      deploymentTarget: claudony
      replicas: 1

  my-llm:
    type: casehub_application
    dependsOn: [engine]
    forEach:
      as: llm
      in: llm-fleet
    spec:
      applicationId: my-llm-${each.llm.name}
      image: casehubio/my-llm-app:latest
      deploymentTarget: claudony
      replicas: 1
      config:
        llm.capability: ${each.llm.capability}
        llm.identity-source: eidos
```

The ForEachExpander produces 3 nodes: `my-llm.analyst`,
`my-llm.coder`, `my-llm.assistant`. Each has a distinct
`applicationId` (`my-llm-analyst`, `my-llm-coder`,
`my-llm-assistant`) and capability config.

**Dependency graph:**

```
manifest ──> eidos ──> engine ──┬──> my-llm.analyst
                                ├──> my-llm.coder
                                └──> my-llm.assistant
```

The 3 LLM instances can deploy in parallel — they all depend only
on `engine`.

### 2. Eidos Descriptors

Location: `topology-tests/src/test/resources/META-INF/eidos/descriptors.yaml`

Add 3 entries for the fleet members alongside the existing `my-llm`
descriptor. Each has a distinct slot, capability tier, and briefing
matching the topology config:

```yaml
  - agentId: my-llm-analyst
    name: LLM Analyst
    slot: analyst
    disposition:
      conflictMode: collaborating
    capabilities:
      - name: conversation
        model:
          tier: DEEP
          capabilities: [text]
    briefing: >-
      You are an analyst agent specialising in deep reasoning
      and investigation. Part of a fleet of 3 LLM agents.

  - agentId: my-llm-coder
    name: LLM Coder
    slot: developer
    disposition:
      conflictMode: collaborating
    capabilities:
      - name: conversation
        model:
          tier: FAST
          capabilities: [text]
    briefing: >-
      You are a coding agent optimised for fast code generation
      and implementation. Part of a fleet of 3 LLM agents.

  - agentId: my-llm-assistant
    name: LLM Assistant
    slot: assistant
    disposition:
      conflictMode: collaborating
    capabilities:
      - name: conversation
        model:
          tier: BALANCED
          capabilities: [text]
    briefing: >-
      You are a general-purpose assistant providing balanced
      responses. Part of a fleet of 3 LLM agents.
```

These static descriptors are the Level A baseline — they work today.
The `llm.capability` config flowing through the topology (Level B)
demonstrates the parameter path that eidos will consume once it
supports parameterized identity generation (separate work).

### 3. Integration Test — `FleetScalingIntegrationTest.java`

Location: `topology-tests/src/test/java/io/casehub/ops/topology/integration/FleetScalingIntegrationTest.java`

Follows the `HelloLlmIntegrationTest` pattern:

- `@Tag("integration-live")` + `@ServiceCluster` with real claudony
- Compiles `04-llm-fleet.yaml` with the deployment type registry
- Deploys all nodes via real claudony
- Asserts:
  - Graph compilation produces exactly 6 nodes (3 platform + 3 LLM)
  - All 3 LLM instances deploy successfully (`DeployResult.Success`)
  - All 3 report healthy status (not `ABSENT`)
  - Each has a distinct `applicationId` (`my-llm-analyst`,
    `my-llm-coder`, `my-llm-assistant`)

## Scope Boundaries

**In scope:**
- `04-llm-fleet.yaml` topology with CSV data source + forEach
- 3 eidos descriptors (static, Level A)
- Integration test with real claudony

**Out of scope (future levels):**
- Eidos consuming topology config for dynamic identity (Level B
  completion — requires eidos changes)
- Pooling and scaling — dynamic instance count based on load
- LLM-as-deployer — an LLM that ad-hoc deploys other LLMs

## Prerequisites

- #139 — real instance integration test infrastructure (merged)

## References

- `topology-tests/src/test/resources/topologies/deployment/01-hello-llm.yaml` — base topology
- `topology-tests/src/test/resources/topologies/microservices/ha-multi-az-trading.yaml` — forEach pattern
- `desiredstate/yaml/runtime ForEachExpanderTest.java:252-271` — CSV data source expansion
- `claudony/src/main/java/io/casehub/ops/claudony/ClaudonyDeploymentTarget.java:103-114` — toAppSpec ignores replicas
- `topology-tests/src/test/java/io/casehub/ops/topology/integration/HelloLlmIntegrationTest.java` — test pattern
- `topology-tests/src/test/resources/META-INF/eidos/descriptors.yaml` — existing agent descriptor
