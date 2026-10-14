# Deployment Topology Integration Tests

**Date:** 2026-10-14
**Status:** Draft
**Branch:** TBD (one branch per level)

## Problem

CaseHub has a mature desiredstate runtime, a deployment module with a full SPI quad, and a claudony
fleet deployment target — but no integration tests that deploy CaseHub apps through the full
YAML → compile → reconcile → provision pipeline. The existing topology-tests module validates
infra-level K8s topologies. The deployment module's integration tests are Java-programmatic, not
YAML-driven.

There is no way to demonstrate "here's a YAML, deploy and run a CaseHub app" end-to-end.

## Goal

A data-driven integration test suite in `topology-tests` where each YAML file is both a test case
and a tutorial step. Adding a new topology means dropping a YAML file — no Java changes needed.
The suite progressively demonstrates CaseHub's deployment capabilities from "deploy your first
LLM app" to "manage a monitored fleet with a chat UI."

## Design

### Architecture

Extend `topology-tests` with a parallel directory for deployment topologies:

```
topology-tests/src/test/
├── resources/topologies/
│   ├── infra/                (existing — K8s node types)
│   └── deployment/           (NEW — CaseHub app topologies)
│       ├── INDEX.md
│       ├── 01-hello-llm.yaml
│       ├── 02-accountability.yaml
│       ├── 03-case-management.yaml
│       ├── 04-llm-fleet.yaml
│       ├── 05-config-drift.yaml
│       ├── 06-lifecycle.yaml
│       ├── 07-topology-mutations.yaml
│       ├── 08-fleet-monitoring.yaml
│       └── 09-chat-ui.yaml
└── java/.../topology/
    ├── TopologyTestBase.java               (existing)
    ├── ReconciliationTestBase.java         (existing)
    ├── DeploymentTopologyTestBase.java     (NEW)
    ├── DeploymentCompilationTest.java      (NEW)
    └── DeploymentReconciliationTest.java   (NEW)
```

### Test Framework

**`DeploymentTopologyTestBase`** extends `TopologyTestBase`:
- Registers all `DeploymentNodeSpec` sealed subtypes in `NodeSpecRegistry` (agent, channel,
  case_type, trust_policy, endpoint, detection, pool, casehub_application)
- Configures WireMock for the claudony fleet REST API (`/api/claudony/apps`)
- Wires `ClaudonyDeploymentTarget` pointed at WireMock
- Wires stub registries from `testing` module for agent/channel/case_type/trust/endpoint
- Provides `DeploymentActualStateAdapter` backed by stubs + WireMock status queries

**`DeploymentCompilationTest`** — JUnit 5 `@ParameterizedTest` that auto-discovers
`topologies/deployment/*.yaml`. For each file: parse → compile → verify correct node count,
types, dependency edges.

**`DeploymentReconciliationTest`** — JUnit 5 `@ParameterizedTest` that auto-discovers same
YAMLs. For each file: compile → plan from empty → execute against WireMock/stubs → verify
all nodes provisioned → re-plan → verify idempotent (no transitions).

### Claudony WireMock

WireMock stubs for the claudony fleet REST API:

| Endpoint | Stub behavior |
|----------|--------------|
| `POST /api/claudony/apps` | Accept registration, return 201 |
| `PUT /api/claudony/apps/{name}` | Accept update, return 200 |
| `DELETE /api/claudony/apps/{name}` | Accept deregistration, return 204 |
| `GET /api/claudony/apps/{name}` | Return app info with HEALTHY status |
| `POST /api/claudony/apps/{name}/suspend` | Accept, return 200 |
| `POST /api/claudony/apps/{name}/resume` | Accept, return 200 |

Tests verify WireMock received the expected calls with correct payloads — this validates
that the reconciliation loop correctly invokes the claudony deployment target.

### YAML Format

Each topology YAML is a complete app definition — topology AND behavior:

```yaml
# ============================================================
# Level 01: Hello LLM
# ============================================================
# The minimum viable CaseHub app. Deploys five services via
# claudony: manifest, eidos, engine, blocks runtime, and an
# LLM app. After deployment, submits a task and verifies the
# LLM executes its block-defined workflow.
#
# Prerequisites: None — this is where you start.
# What's new: Everything. YAML topology, claudony deployment,
#             block-defined LLM behavior.
# ============================================================

desiredState:
  namespace: deployment
  name: hello-llm

nodes:
  eidos:
    type: casehub_application
    spec:
      applicationId: eidos
      deploymentTarget: claudony
      # ...

  engine:
    type: casehub_application
    dependsOn: [eidos]
    spec:
      applicationId: engine
      deploymentTarget: claudony
      # ...

  my-llm:
    type: casehub_application
    dependsOn: [engine]
    spec:
      applicationId: my-llm
      deploymentTarget: claudony
      # ...
```

Exact schema details for blocks, agents, and behavior definitions will be refined during
level 01 implementation — the YAML format must align with the engine and blocks modules'
actual APIs.

### Progression

Each level is a separate issue, branch, and YAML file. Reassess after each.

| Level | File | Services deployed | Capability demonstrated |
|-------|------|-------------------|------------------------|
| 01 | `01-hello-llm.yaml` | Manifest + eidos + engine + blocks + LLM via claudony | Deploy and run a complete YAML-defined LLM app. Submit a task, verify execution. |
| 02 | `02-accountability.yaml` | Level 01 + ledger + work | Deployment is audited. Task execution creates work items. |
| 03 | `03-case-management.yaml` | Level 02 + qhorus | Tasks managed as cases. Full case lifecycle. |
| 04 | `04-llm-fleet.yaml` | Level 03 + 2 more LLM instances | Same app scaled to 3 instances. Load distributed. |
| 05 | `05-config-drift.yaml` | Level 04 + config change | Change a config value. Only the diff is reconciled. |
| 06 | `06-lifecycle.yaml` | Level 04 | Suspend and resume the fleet. Stateful lifecycle. |
| 07 | `07-topology-mutations.yaml` | Level 04 + app add/remove | Declaratively add or remove apps. Prune before grow. |
| 08 | `08-fleet-monitoring.yaml` | Level 04 + RAS | Fleet health monitoring and supervision. |
| 09 | `09-chat-ui.yaml` | Level 03 + chat-app | Standalone chat UI viewing qhorus conversations. |

### What "actually doing something" means per level

Each test goes beyond "nodes were provisioned" — it verifies the capability that level
introduces:

- **Level 01:** WireMock received register calls for all 5 services in dependency order.
  Claudony status queries return HEALTHY. Reconciliation loop closes.
- **Level 02:** Ledger service is registered before the LLM. Work service dependency
  resolved. Verify register payloads include audit trail configuration.
- **Level 03:** Qhorus registered. LLM app config references qhorus for case management.
- **Level 04:** Three LLM instances registered with distinct instance IDs. All report HEALTHY.
- **Level 05:** Config change detected as drift. Only the changed node is re-provisioned
  (WireMock receives update, not register).
- **Level 06:** Suspend calls hit WireMock for all fleet instances. Resume restores them.
  Status transitions: HEALTHY → SUSPENDED → HEALTHY.
- **Level 07:** New app added (register call). Existing app removed (deregister call).
  Dependency ordering respected.
- **Level 08:** RAS service registered. Fleet health aggregation configured.
- **Level 09:** Chat-app registered with qhorus URL in its config.

### Tutorial Structure

Each YAML file has a header comment block:
- Level name and number
- What this level demonstrates (one sentence)
- Prerequisites (which prior levels)
- What's new (concepts introduced)

`INDEX.md` lists all levels with one-line descriptions and reading order.

### Dependencies

| Dependency | Source | Status |
|-----------|--------|--------|
| `DeploymentNodeSpec` sealed hierarchy | deployment module | Exists |
| `ClaudonyDeploymentTarget` | claudony module | Exists |
| `HttpClaudonyFleetClient` | claudony module | Exists |
| `DeploymentGoalCompiler` | deployment module | Exists |
| `CaseHubApplicationSpec` | deployment module | Exists |
| WireMock test dependency | topology-tests pom.xml | Needs adding |
| `DeploymentTopologyTestBase` | topology-tests | Needs creating |
| YAML topology files | topology-tests resources | Needs creating |

### Out of scope

- Real claudony/qhorus instances in tests (later tier)
- Podman/container-based deployment
- K8s/production deployment
- Chat-app internal wiring (level 09 concern)
- CI pipeline changes

## Execution

Iterative — one level per issue/branch. Level 01 first. Reassess the design after each
level completes. The spec captures the vision; implementation may diverge as we learn.

Invoke `writing-plans` for level 01 only after this spec is approved.

## References

- `topology-tests/` module — existing data-driven test framework
- `deployment/src/test/resources/topology.yaml` — existing YAML format
- `claudony/src/main/java/.../ClaudonyDeploymentTarget.java` — deployment target
- `claudony/src/main/java/.../HttpClaudonyFleetClient.java` — REST client
- `docs/specs/2026-08-29-canonical-deployment-topologies-design.md` — infra topology spec
- `docs/specs/issue-101-yaml-apps-design-spec/` — YAML apps vision
- `docs/specs/issue-117-deployment-profiles-bootstrap/` — deployment profiles
