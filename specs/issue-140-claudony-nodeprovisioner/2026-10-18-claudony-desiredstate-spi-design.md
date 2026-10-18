# Claudony Desiredstate SPI — Design Spec

**Issue:** casehubio/casehub-ops#140
**Date:** 2026-10-18
**Status:** Draft

## Overview

The claudony module becomes a standalone desiredstate domain managing the claudony fleet declaratively. It implements the full SPI Quad: GoalCompiler, ActualStateAdapter, NodeProvisioner, FaultPolicy, and EventSource.

This supersedes ADR-0002 for the fleet management use case while leaving ADR-0002's `ApplicationDeploymentTarget` decision valid for the deployment module's multi-target routing. The two patterns are complementary, not conflicting.

## Domain Boundary

| Concern | Module | Node types | Path to claudony API |
|---------|--------|-----------|---------------------|
| Deploy a CaseHub app to claudony | deployment | `casehub_application` | DeploymentNodeProvisioner -> CaseHubAppProvisionHandler -> ClaudonyDeploymentTarget -> ClaudonyFleetClient |
| Manage claudony fleet declaratively | claudony | `claudony_app`, `claudony_pool` | ClaudonyNodeProvisioner -> ClaudonyFleetClient |

`ClaudonyDeploymentTarget` stays as-is. Both paths share `ClaudonyFleetClient`. Documentation must make the boundary clear: don't declare the same app in both deployment YAML (as `casehub_application` with `deploymentTargetId: claudony`) and claudony fleet YAML (as `claudony_app`).

## Type Model (ops-api)

New package `io.casehub.ops.api.claudony` in the `api/` module.

### ClaudonyNodeSpec sealed interface

```java
public sealed interface ClaudonyNodeSpec extends NodeSpec
    permits ClaudonyAppNodeSpec, ClaudonyPoolNodeSpec {}
```

Sealed hierarchy follows ARC42STORIES section 4 Type Strategy Selection: apps and pools have genuinely different fields and provisioning semantics.

### ClaudonyAppNodeSpec

```java
@NodeTypeId("claudony_app")
public record ClaudonyAppNodeSpec(
    String name,
    String namespace,
    String displayName,
    String description,
    String image,
    Map<String, String> config
) implements ClaudonyNodeSpec {
    public NodeType nodeType() { return ClaudonyNodeTypes.APP; }
    public String nodeId() { return "claudony_app:" + name; }
}
```

Not a duplicate of `CaseHubApplicationSpec`. Different domain model: no `source`, `runtime`, `dependencies`, `replicas`. Claudony uses pools (separate nodes), not replicas.

### ClaudonyPoolNodeSpec

```java
@NodeTypeId("claudony_pool")
public record ClaudonyPoolNodeSpec(
    String appName,
    String agentId,
    int minActive,
    int maxActive
) implements ClaudonyNodeSpec {
    public NodeType nodeType() { return ClaudonyNodeTypes.POOL; }
    public String nodeId() { return "claudony_pool:" + appName + ":" + agentId; }
}
```

`appName` links the pool to its parent app for dependency wiring in the compiler.

### ClaudonyNodeTypes

```java
public final class ClaudonyNodeTypes {
    public static final NodeType APP = NodeType.of("claudony_app");
    public static final NodeType POOL = NodeType.of("claudony_pool");
}
```

### ClaudonyGoals

```java
public record ClaudonyGoals(
    List<GoalEntry<ClaudonyAppNodeSpec>> apps,
    List<GoalEntry<ClaudonyPoolNodeSpec>> pools
) {}
```

Reuses `GoalEntry` from the deployment module for `dependsOn` edge support.

### ClaudonyReviewSpec

```java
@NodeTypeId("claudony_review")
public record ClaudonyReviewSpec(
    String failedNodeId,
    int failureCount,
    String lastError
) implements NodeSpec {
    public NodeType nodeType() { return NodeType.of("claudony_review"); }
}
```

Fault escalation target for `ClaudonyFaultPolicy`.

## SPI Implementations (claudony module)

All five SPI beans in `io.casehub.ops.claudony`.

### ClaudonyGoalCompiler

`GoalCompiler<ClaudonyGoals>`, `@ApplicationScoped`.

- Iterates `goals.apps()`: creates `DesiredNode` per app with `HumanGating.NONE`
- Iterates `goals.pools()`: creates `DesiredNode` per pool, auto-adds `Dependency(poolId, appId)` linking each pool to its parent app via `poolSpec.appName()`
- User-declared `dependsOn` edges on `GoalEntry` are also added
- Returns `CompilationResult.single(factory.of(nodes, dependencies))`

### ClaudonyActualStateAdapter

`ActualStateAdapter`, `@ApplicationScoped`. Injects `ClaudonyFleetClient` and `SpecHashStore`.

`handledTypes()` returns `{CLAUDONY_APP, CLAUDONY_POOL}`.

`readActual()`:
1. Batch query: `fleetClient.listApps(tenancyId)` where tenancyId maps to namespace
2. Build `Map<String, ClaudonyAppInfo>` by app name
3. For each desired `claudony_app` node: look up by name, map `HealthStatus` to `NodeStatus` via `ClaudonyHealthMapper.toNodeStatus()`. Layer spec-hash check for declaration drift.
4. For each desired `claudony_pool` node: look up parent app by `appName`, find matching pool by `agentId` in `app.pools()`. ABSENT if parent app missing or pool not found. DRIFTED if `minActive`/`maxActive` differ from desired. PRESENT otherwise. Spec-hash layer on top.

### ClaudonyNodeProvisioner

`NodeProvisioner`, `@ApplicationScoped`. Injects `ClaudonyFleetClient` and `SpecHashStore`.

`handledTypes()` returns `{CLAUDONY_APP, CLAUDONY_POOL}`.
`supportsStatefulLifecycle()` returns `true`.

**provision()** — switch on spec type:
- `ClaudonyAppNodeSpec`: build `ClaudonyAppSpec` (no pools list — pools are separate nodes), call `registerApp()`. On 409 conflict, fall back to `updateApp()`. Record spec hash on success.
- `ClaudonyPoolNodeSpec`: read-modify-write. `getApp(appName)`, append/update pool in pools list, `updateApp()`. Record spec hash on success.

**deprovision()**:
- `ClaudonyAppNodeSpec`: `deregisterApp(name)`. 404 treated as success. Remove spec hash.
- `ClaudonyPoolNodeSpec`: read-modify-write. `getApp(appName)`, remove pool from list, `updateApp()`. Remove spec hash.

**suspend() / resume()**:
- `ClaudonyAppNodeSpec`: `suspendApp(name)` / `resumeApp(name)`
- `ClaudonyPoolNodeSpec`: `Failed("suspend not supported for pools")`

### ClaudonyFaultPolicy

`FaultPolicy`, `@ApplicationScoped`.

Delegates to `ThresholdFaultPolicy` configured with:
- Fault types: `PROVISION_FAILED`
- Node types: `CLAUDONY_APP`, `CLAUDONY_POOL`
- Ignore types: `claudony_review` (prevents infinite escalation)
- Tier 3 threshold: after 3 failures, adds a `ClaudonyReviewSpec` review node via `FaultPolicy.addReviewNode()`

### ClaudonyEventSource

`EventSource`, `@ApplicationScoped`.

Hot `Multi<StateEvent>` via emitter with broadcast to all subscribers. Exposes `emit(StateEvent)` and convenience `emitDrift(NodeId)` for external callers. Follows `ContainerEventSource` pattern.

### ClaudonyNodeSpecFactoryProvider

`NodeSpecFactoryProvider`, plain class (no CDI).

Scans `ClaudonyNodeSpec.class.getPermittedSubclasses()` for `@NodeTypeId` annotations, registers Jackson `ObjectMapper.convertValue()` deserialization factories. Follows `ContainerNodeSpecFactoryProvider` pattern.

## Shared Utilities

### ClaudonyHealthMapper

Extract `ClaudonyDeploymentTarget.mapHealth()` to a shared utility:

```java
public final class ClaudonyHealthMapper {
    public static NodeStatus toNodeStatus(HealthStatus health) {
        return switch (health) {
            case HEALTHY -> NodeStatus.PRESENT;
            case DEGRADED -> NodeStatus.DRIFTED;
            case DOWN -> NodeStatus.ABSENT;
            case SUSPENDED -> NodeStatus.SUSPENDED;
        };
    }
}
```

Both `ClaudonyDeploymentTarget` and `ClaudonyActualStateAdapter` delegate to this.

### ClaudonyGoalLoader

Follows established pattern (`DeploymentGoalLoader`, `ComplianceGoalLoader`):
- Classpath-first resource lookup, filesystem fallback
- Loads `claudony-fleet.yaml` files
- Parses into `ClaudonyGoals` record
- Integrates with `TopologyCompilerService` via `NodeSpecFactoryProvider` registration

## Test Strategy

Three layers matching the container module pattern:

| Layer | What | Pattern |
|-------|------|---------|
| Unit | Each SPI impl in isolation | `StubFleetClient` returns canned data. Test compiler graph topology (nodes, edges, pool-to-app dependencies). Test adapter status mapping for all HealthStatus values. Test provisioner call sequences via `RecordingFleetClient`. Test fault policy threshold escalation. |
| Integration | Full reconciliation loop | `SimulatingFleetClient` with stateful `ConcurrentHashMap` of apps. Wire all SPI impls manually (no CDI). Scenarios: green-field provision and converge, drift detection and re-provision, deprovision, pool add/remove via read-modify-write, suspend/resume lifecycle. |
| HTTP | HttpClaudonyFleetClient | Real `HttpServer` with stubbed endpoints. Extend existing test to cover pool mutation scenarios. |

Test stubs implement `ClaudonyFleetClient` interface directly (already an interface — no `super(null, null)` hacks needed).

## Deliverables

### Code

| Location | Artifact |
|----------|----------|
| `api/src/main/java/io/casehub/ops/api/claudony/` | `ClaudonyNodeSpec`, `ClaudonyAppNodeSpec`, `ClaudonyPoolNodeSpec`, `ClaudonyNodeTypes`, `ClaudonyGoals`, `ClaudonyReviewSpec` |
| `claudony/src/main/java/io/casehub/ops/claudony/` | `ClaudonyGoalCompiler`, `ClaudonyActualStateAdapter`, `ClaudonyNodeProvisioner`, `ClaudonyFaultPolicy`, `ClaudonyEventSource`, `ClaudonyNodeSpecFactoryProvider`, `ClaudonyGoalLoader`, `ClaudonyHealthMapper` |
| `claudony/src/test/java/io/casehub/ops/claudony/` | Unit tests, integration test, extended HTTP client test |

### Documentation

| Artifact | Action |
|----------|--------|
| ADR-0003 | New ADR superseding ADR-0002 for fleet management use case |
| ARC42STORIES.MD | Update section 5 (Building Block View) to include claudony domain module, update section 4 with claudony type strategy |
| Consumer guide | Document claudony fleet YAML format and boundary with deployment module |

### Claudony issues to file

| Issue | Purpose |
|-------|---------|
| Standalone pool CRUD endpoints | `POST/DELETE /api/claudony/apps/{name}/pools/{agentId}` — future improvement to replace read-modify-write |
| ETag/version support on app specs | Enables optimistic concurrency for read-modify-write bridge |

## Trade-offs

- **Dual paths to ClaudonyFleetClient**: An app declared as both `casehub_application` (deployment) and `claudony_app` (fleet) would conflict. Operational misconfiguration, not architectural flaw. Documented in consumer guide.
- **Read-modify-write for pools**: Provisioning a pool requires reading and writing the entire app spec. Sequential reconciliation prevents races within the loop, but external mutations could be overwritten. Acceptable for expected scale; standalone pool endpoints (claudony issue) are the long-term fix.
- **Two node types inline in adapter/provisioner**: If the claudony domain grows beyond 2-3 node types, the inline switch pattern may need refactoring to the drift checker / handler class pattern.

## References

- `docs/adr/0002-claudony-integration-via-deployment-target.md` — ADR being superseded for fleet management
- `ARC42STORIES.MD` section 4 — type strategy selection rationale
- `ARC42STORIES.MD` section 8 — SPI Quad pattern
- `container/src/main/java/io/casehub/ops/container/ContainerGoalCompiler.java` — closest structural analogue for GoalCompiler
- `container/src/main/java/io/casehub/ops/container/PodmanActualStateAdapter.java` — closest structural analogue for ActualStateAdapter
- `container/src/main/java/io/casehub/ops/container/PodmanNodeProvisioner.java` — closest structural analogue for NodeProvisioner
- `container/src/main/java/io/casehub/ops/container/ContainerFaultPolicy.java` — FaultPolicy with ThresholdFaultPolicy escalation
- `deployment/src/main/java/io/casehub/ops/deployment/DeploymentGoalCompiler.java` — GoalEntry pattern reference
- `claudony/src/main/java/io/casehub/ops/claudony/ClaudonyFleetClient.java` — existing REST client interface
- `claudony/src/main/java/io/casehub/ops/claudony/ClaudonyDeploymentTarget.java` — existing deployment target (coexists)
- `claudony/src/main/java/io/casehub/ops/claudony/ClaudonyAppSpec.java` — claudony API data types
- `api/src/main/java/io/casehub/ops/api/deployment/GoalEntry.java` — reused for dependsOn edges
- casehubio/casehub-ops#140 — parent issue
- claudony#282 — app lifecycle API (landed)
- platform#615 — HealthStatus.SUSPENDED (landed)
