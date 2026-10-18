# Claudony Desiredstate SPI — Design Spec

**Issue:** casehubio/casehub-ops#140
**Date:** 2026-10-18
**Status:** Draft

## Overview

The claudony module becomes a standalone desiredstate domain managing the claudony fleet declaratively. It implements the full SPI Quad: GoalCompiler, ActualStateAdapter, NodeProvisioner, FaultPolicy, and EventSource.

ADR-0003 establishes a standalone SPI Quad for claudony fleet management alongside ADR-0002's deployment target pattern. ADR-0002 remains fully valid for its original scope (`casehub_application` nodes via `ApplicationDeploymentTarget`). The claudony desiredstate module introduces new node types (`claudony_app`, `claudony_pool`) that are distinct from `casehub_application` — the two patterns are complementary, not sequential.

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
    permits ClaudonyAppNodeSpec, ClaudonyPoolNodeSpec {
    String nodeId();
    @Override NodeType nodeType();
}
```

Sealed hierarchy follows ARC42STORIES section 4 Type Strategy Selection: apps and pools have genuinely different fields and provisioning semantics.

### ClaudonyAppNodeSpec

```java
@NodeTypeId("claudony_app")
public record ClaudonyAppNodeSpec(
    String name,
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

**No `namespace` field.** The claudony API namespace is determined by the reconciliation context (tenancyId), not the YAML declaration. The provisioner sets `namespace = tenancyId` when building `ClaudonyAppSpec`. This ensures coherence: the adapter searches `listApps(tenancyId)` in the same namespace the provisioner registers apps in. Without this enforcement, a YAML author setting `namespace: production` while the reconciliation loop runs as `tenancyId=staging` would cause silent infinite re-provisioning (app registered in `production`, adapter searching `staging` → perpetual ABSENT). This matches the existing `ClaudonyDeploymentTarget.toAppSpec()` which passes `null` for namespace (line 110), letting the deployment context determine placement.

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

### ClaudonyGoalEntry

```java
@JsonIgnoreProperties(ignoreUnknown = true)
public record ClaudonyGoalEntry<S extends ClaudonyNodeSpec>(S spec, List<String> dependsOn) {
    public ClaudonyGoalEntry {
        dependsOn = dependsOn != null ? List.copyOf(dependsOn) : List.of();
    }
}
```

Each domain defines its own goal entry type (`GoalEntry` is bound to `DeploymentNodeSpec`, `ComplianceGoalEntry` is compliance-specific).

### ClaudonyGoals

```java
public record ClaudonyGoals(
    List<ClaudonyGoalEntry<ClaudonyAppNodeSpec>> apps,
    List<ClaudonyGoalEntry<ClaudonyPoolNodeSpec>> pools
) {}
```

### ClaudonyReviewSpec

```java
public record ClaudonyReviewSpec(NodeId faultedNode, String reason) implements NodeSpec {
    public ClaudonyReviewSpec {
        Objects.requireNonNull(faultedNode, "faultedNode");
        Objects.requireNonNull(reason, "reason");
    }

    @Override
    public NodeType nodeType() { return NodeType.of("claudony_review"); }
}
```

Fault escalation target for `ClaudonyFaultPolicy`. Follows `ContainerReviewSpec` pattern — uses `NodeId` (matching `FaultEvent.node()` return type) and `String reason` (from `FaultEvent.detail()`). No `@NodeTypeId` annotation: review specs are created at runtime by `FaultPolicy.addReviewNode()`, not loaded from YAML via `NodeSpecFactoryProvider`. `failureCount` removed — `ReviewSpecFactory.create(FaultEvent, DesiredStateGraph)` does not receive failure count data (tracked internally by `ThresholdFaultPolicy` via `FaultCountStore`).

## SPI Implementations (claudony module)

All five SPI beans in `io.casehub.ops.claudony`.

### ClaudonyGoalCompiler

`GoalCompiler<ClaudonyGoals>`, `@ApplicationScoped`, `@DesiredStateQualifier(namespace = "claudony")`.

The `@DesiredStateQualifier` annotation (from `casehub-desiredstate-annotations`) is required by ARC42STORIES §2: GoalCompiler ambiguity across coexisting domain modules is resolved via this CDI qualifier. Existing domain GoalCompilers (`DeploymentGoalCompiler`, `ComplianceGoalCompiler`, `InfraGoalCompiler`) do not yet use the qualifier — adding it to claudony follows the documented architecture and sets the pattern. Backfilling the qualifier on existing compilers is out-of-scope for this issue.

- Iterates `goals.apps()`: creates `DesiredNode` per app with `HumanGating.NONE`
- Iterates `goals.pools()`: creates `DesiredNode` per pool, auto-adds `Dependency(poolId, appId)` linking each pool to its parent app via `poolSpec.appName()`
- User-declared `dependsOn` edges on `GoalEntry` are also added
- Returns `CompilationResult.single(factory.of(nodes, dependencies))`

### ClaudonyActualStateAdapter

`ActualStateAdapter`, `@ApplicationScoped`. Injects `ClaudonyFleetClient` and `ClaudonySpecHashStore`.

`handledTypes()` returns `{CLAUDONY_APP, CLAUDONY_POOL}`.

`readActual()`:
1. Batch query: `fleetClient.listApps(tenancyId)` — identity mapping: tenancyId IS the claudony namespace. Each reconciliation tenant corresponds directly to a claudony namespace. This matches the claudony fleet model where apps are partitioned by namespace.
2. Build `Map<String, ClaudonyAppInfo>` by app name
3. For each desired `claudony_app` node: look up by name, map `HealthStatus` to `NodeStatus` via `ClaudonyHealthMapper.toNodeStatus()`. Layer spec-hash check for declaration drift.
4. For each desired `claudony_pool` node: look up parent app by `appName`, find matching pool by `agentId` in `app.pools()`. ABSENT if parent app missing or pool not found. DRIFTED if `minActive`/`maxActive` differ from desired. PRESENT otherwise. Spec-hash layer on top.

### ClaudonyNodeProvisioner

`NodeProvisioner`, `@ApplicationScoped`. Injects `ClaudonyFleetClient` and `ClaudonySpecHashStore`.

`handledTypes()` returns `{CLAUDONY_APP, CLAUDONY_POOL}`.
`supportsStatefulLifecycle()` returns `true`.

**provision()** — switch on spec type:
- `ClaudonyAppNodeSpec`: build `ClaudonyAppSpec` with `namespace = context.tenancyId()` (no pools list — pools are separate nodes), call `registerApp()`. On 409 conflict, fall back to `updateApp()`. Record spec hash on success.
- `ClaudonyPoolNodeSpec`: read-modify-write. `getApp(appName)` — if `Optional.empty()`, return `ProvisionResult.Failed("parent app not registered: " + appName)` without attempting the read-modify-write. Otherwise, append/update pool in pools list, `updateApp()`. Record spec hash on success. (The pool → app dependency edge prevents this case during normal ordering, but the guard handles cross-cycle edge cases where app provision fails repeatedly.)

**deprovision()**:
- `ClaudonyAppNodeSpec`: `deregisterApp(name)`. 404 treated as success. Remove spec hash.
- `ClaudonyPoolNodeSpec`: read-modify-write. `getApp(appName)` — if `Optional.empty()`, treat as success (parent app gone implies pool is gone). Otherwise, remove pool from list, `updateApp()`. Remove spec hash.

**suspend() / resume()**:
- `ClaudonyAppNodeSpec`: `suspendApp(name)` / `resumeApp(name)`
- `ClaudonyPoolNodeSpec`: `Failed("suspend not supported for pools")`

### ClaudonyFaultPolicy

`FaultPolicy`, `@ApplicationScoped`.

Delegates to `ThresholdFaultPolicy` configured with:
- Fault types: `PROVISION_FAILED`
- Node types: `CLAUDONY_APP`, `CLAUDONY_POOL`
- Ignore types: `Set.of(NodeType.of("claudony_review"))` — belt-and-suspenders, matching `ContainerFaultPolicy` pattern. `ThresholdFaultPolicy` automatically adds tier output types to `ignoreTypes`, so this is technically redundant but consistent with existing code.
- Tier 3 threshold: after 3 failures, adds a `ClaudonyReviewSpec` review node via `FaultPolicy.addReviewNode((event, current) -> new ClaudonyReviewSpec(event.node(), event.detail()))`

### ClaudonyEventSource

`EventSource`, `@ApplicationScoped`.

Hot `Multi<StateEvent>` via emitter with broadcast to all subscribers. Exposes `emit(StateEvent)` and convenience `emitDrift(NodeId)` for external callers. Follows `ContainerEventSource` pattern.

### DomainRegistration (in BootstrapDomainRegistrar)

The claudony domain must be registered with `CrossDomainCompositionEngine` for cross-domain ordering. Following the established pattern in `BootstrapDomainRegistrar` (service module), a new `registerClaudonyDomain()` method:

```java
engine.registerDomain(DomainRegistration.builder(DomainId.of("claudony"),
    CompilationResult.single(emptyGraph))
    .provides(Set.of(ClaudonyNodeTypes.APP, ClaudonyNodeTypes.POOL))
    .build());
```

- `provides`: `claudony_app`, `claudony_pool` — the two goal-compiled node types
- `requires`: empty — claudony fleet management is standalone, no dependency on infrastructure or other domains
- `claudony_review` is NOT in `provides`: review nodes are fault-policy artifacts, not goal-compiled resources

Registration is centralized in `BootstrapDomainRegistrar`, not in the claudony domain module itself — this matches the existing pattern for infra and deployment domains.

### Heartbeat Integration

`ClaudonyFleetClient.heartbeat(String name)` is intentionally NOT called by `ClaudonyActualStateAdapter` or `ClaudonyNodeProvisioner`. The heartbeat endpoint is a claudony API feature for keeping apps alive during idle periods. In the desiredstate model, app liveness is managed declaratively:
- Present apps remain registered via their `ClaudonyAppNodeSpec` in the desired graph
- Absent apps are deprovisioned via `deregisterApp()`
- The reconciliation loop's continuous cycling inherently keeps the system active

If claudony adds idle-timeout semantics in the future, heartbeat integration would be a separate concern — likely a periodic CDI `@Scheduled` task calling `heartbeat()` for all registered apps, not a side-effect of `readActual()`.

### ClaudonyNodeSpecFactoryProvider

`NodeSpecFactoryProvider`, plain class (no CDI).

Scans `ClaudonyNodeSpec.class.getPermittedSubclasses()` for `@NodeTypeId` annotations, registers Jackson `ObjectMapper.convertValue()` deserialization factories. Follows `ContainerNodeSpecFactoryProvider` pattern.

## Shared Utilities

### ClaudonySpecHashStore

`@ApplicationScoped`. Module-local spec hash store for the claudony domain. Identical to `SpecHashStore` (deployment) and `ComplianceSpecHashStore` (compliance) — each domain module has its own CDI-scoped instance. The claudony module cannot inject the deployment module's `SpecHashStore` as that would create a cross-module dependency.

```java
@ApplicationScoped
public class ClaudonySpecHashStore {
    private final ConcurrentHashMap<NodeId, Integer> hashes = new ConcurrentHashMap<>();

    public void record(NodeId id, NodeSpec spec) { hashes.put(id, spec.hashCode()); }
    public void remove(NodeId id) { hashes.remove(id); }
    public boolean hasDrifted(NodeId id, NodeSpec spec) {
        Integer stored = hashes.get(id);
        return stored == null || !stored.equals(spec.hashCode());
    }
}
```

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

Standalone domain goal loader following `DeploymentGoalLoader` / `ComplianceGoalLoader` pattern. `@ApplicationScoped`.

- Classpath-first resource lookup, filesystem fallback
- Loads `claudony-fleet.yaml` files
- Parses into `ClaudonyGoals` record via Jackson YAML mapper
- Directory-based merging of multiple YAML files into one `ClaudonyGoals` record

This is Path 2 (domain-specific YAML format) — not the standard `desiredState:` YAML format processed by `TopologyCompilerService` / `YamlDesiredStateProcessor`. `ClaudonyNodeSpecFactoryProvider` independently handles Path 1 (standard topology YAML) by providing node type factories for `claudony_app` and `claudony_pool` — the two paths are complementary but separate.

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
| `api/src/main/java/io/casehub/ops/api/claudony/` | `ClaudonyNodeSpec`, `ClaudonyAppNodeSpec`, `ClaudonyPoolNodeSpec`, `ClaudonyNodeTypes`, `ClaudonyGoalEntry`, `ClaudonyGoals`, `ClaudonyReviewSpec` |
| `claudony/src/main/java/io/casehub/ops/claudony/` | `ClaudonyGoalCompiler`, `ClaudonyActualStateAdapter`, `ClaudonyNodeProvisioner`, `ClaudonyFaultPolicy`, `ClaudonyEventSource`, `ClaudonyNodeSpecFactoryProvider`, `ClaudonyGoalLoader`, `ClaudonyHealthMapper`, `ClaudonySpecHashStore` |
| `service/src/main/java/.../bootstrap/` | Update `BootstrapDomainRegistrar` — add `registerClaudonyDomain()` |
| `claudony/src/test/java/io/casehub/ops/claudony/` | Unit tests, integration test, extended HTTP client test |

### Documentation

| Artifact | Action |
|----------|--------|
| ADR-0003 | New ADR establishing standalone SPI Quad for claudony fleet management alongside ADR-0002 |
| ARC42STORIES.MD | Update section 5 (Building Block View) to include claudony domain module, update section 4 with claudony type strategy |
| Consumer guide | Document claudony fleet YAML format and boundary with deployment module |

### Claudony issues to file

| Issue | Purpose |
|-------|---------|
| Standalone pool CRUD endpoints | `POST/DELETE /api/claudony/apps/{name}/pools/{agentId}` — future improvement to replace read-modify-write |
| ETag/version support on app specs | Enables optimistic concurrency for read-modify-write bridge |

### casehub-ops issues to file

| Issue | Purpose |
|-------|---------|
| Backfill `@DesiredStateQualifier` on existing GoalCompilers | `DeploymentGoalCompiler`, `ComplianceGoalCompiler`, `InfraGoalCompiler` need the qualifier per ARC42STORIES §2 |
| Promote `SpecHashStore` to shared utility or extract interface | Three identical copies (deployment, compliance, claudony) — candidate for consolidation in `api/` or `casehub-desiredstate-api` |

## Trade-offs

- **Dual paths to ClaudonyFleetClient**: An app declared as both `casehub_application` (deployment) and `claudony_app` (fleet) would conflict. Operational misconfiguration, not architectural flaw. Documented in consumer guide.
- **Read-modify-write for pools**: Provisioning a pool requires reading and writing the entire app spec. Within a single tenant's reconciliation loop, processing is sequential — no intra-tenant race. Cross-tenant races on the same app are impossible because apps are partitioned by namespace (= tenancyId). However, external mutations (via claudony UI or another process) between `getApp()` and `updateApp()` ARE silently overwritten. Concrete failure mode: operator modifies pool config via claudony dashboard, next reconciliation cycle overwrites with desired state. This is the correct desired-state behavior (desired state wins), but the operator must understand that manual pool changes are ephemeral. Standalone pool endpoints (claudony issue) with ETag support are the long-term fix for finer-grained mutation.
- **No approval workflow**: The claudony domain deliberately omits `ApprovalEvaluator` integration. Fleet operations (app registration, pool scaling) are operational, not change-managed. `PlanStoreMapper` mixin updates are therefore not required — claudony `NodeSpec` subtypes will never pass through the approval pipeline. If approval is needed in the future, `PlanStoreMapper` mixin registration for `ClaudonyNodeSpec` subtypes would be required (ARC42STORIES §12 Technical Debt).
- **Two node types inline in adapter/provisioner**: If the claudony domain grows beyond 2-3 node types, the inline switch pattern may need refactoring to the drift checker / handler class pattern.

## References

- `docs/adr/0002-claudony-integration-via-deployment-target.md` — companion ADR for deployment target pattern (remains valid)
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
- `api/src/main/java/io/casehub/ops/api/compliance/ComplianceGoalEntry.java` — per-domain goal entry precedent
- casehubio/casehub-ops#140 — parent issue
- claudony#282 — app lifecycle API (landed)
- platform#615 — HealthStatus.SUSPENDED (landed)
