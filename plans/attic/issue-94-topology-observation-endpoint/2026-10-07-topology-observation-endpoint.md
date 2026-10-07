# Topology Observation Endpoint Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #94 — feat: topology observation REST endpoint for adaptive ops demo
**Issue group:** #94

**Goal:** Add a REST endpoint that returns current desired/actual topology state with adaptation info, making topology changes observable without psql access.

**Architecture:** New `OpsTopologyApi` class in the service module following the existing `@McpDomain`/`@PlatformQuery` pattern. Iterates all active reconciliation loop keys for the requesting tenant, returns desired nodes (with adaptation markers), actual statuses, and active situations. Adaptation snapshot data exposed via a new public method on `DeploymentAdaptiveSituationRecompiler` in the deployment module.

**Tech Stack:** Java 21, Quarkus CDI, desiredstate runtime API, RAS API

## Global Constraints

- Follow `@McpDomain` + `@PlatformQuery` + `@RestPath` pattern (not raw JAX-RS `@Path`)
- Use `@ContextParam("tenancyId")` for tenant scoping (resolved from `TenancyFilter`)
- Response types as records nested in the API class (matches existing pattern)
- No cross-repo changes — all modifications within casehub-ops

---

## Batch 1: Topology Observation Endpoint

### Task 1: Expose adaptation snapshot from DeploymentAdaptiveSituationRecompiler

**Files:**
- Modify: `deployment/src/main/java/io/casehub/ops/deployment/adaptation/DeploymentAdaptiveSituationRecompiler.java`
- Create: `deployment/src/test/java/io/casehub/ops/deployment/adaptation/DeploymentAdaptiveSituationRecompilerSnapshotTest.java`

**Interfaces:**
- Consumes: `TenantAdaptationState` (package-private, accessed internally)
- Produces: `DeploymentAdaptiveSituationRecompiler.getAdaptationSnapshot(String tenancyId)` returning `AdaptationSnapshot` record. `AdaptationSnapshot` is a public record: `(List<ActiveRuleInfo> activeRules, List<TrackedSituationInfo> trackedSituations)`. `ActiveRuleInfo` is: `(String ruleName, String situationId, boolean active)`. `TrackedSituationInfo` is: `(String situationId, double confidence, java.time.Instant since, java.time.Instant lastSignal)`.

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.ops.deployment.adaptation;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.desiredstate.api.DesiredStateGraphFactory;
import io.casehub.ops.api.deployment.AdaptationRuleSpec;
import io.casehub.ops.api.deployment.AdaptationTrigger;
import io.casehub.ops.api.deployment.DeploymentGoals;
import io.casehub.ops.deployment.DeploymentGoalCompiler;
import io.casehub.ras.api.ActiveSituation;
import io.micrometer.core.instrument.simple.SimpleMeterRegistry;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Duration;
import java.time.Instant;
import java.util.List;
import java.util.Map;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.mock;

class DeploymentAdaptiveSituationRecompilerSnapshotTest {

    private DeploymentAdaptiveSituationRecompiler recompiler;
    private DeploymentGoalCompiler compiler;
    private DesiredStateGraphFactory factory;
    private ObjectMapper mapper;

    @BeforeEach
    void setUp() {
        recompiler = new DeploymentAdaptiveSituationRecompiler();
        compiler = mock(DeploymentGoalCompiler.class);
        factory = mock(DesiredStateGraphFactory.class);
        mapper = new ObjectMapper();
        // Wire CDI fields via reflection for unit test
        TestUtil.setField(recompiler, "compiler", compiler);
        TestUtil.setField(recompiler, "mapper", mapper);
        TestUtil.setField(recompiler, "meterRegistry", new SimpleMeterRegistry());
    }

    @Test
    void snapshotReturnsEmptyWhenNoTenantRegistered() {
        var snapshot = recompiler.getAdaptationSnapshot("unknown-tenant");
        assertNotNull(snapshot);
        assertTrue(snapshot.activeRules().isEmpty());
        assertTrue(snapshot.trackedSituations().isEmpty());
    }

    @Test
    void snapshotReturnsRegisteredRulesAndTrackedSituations() {
        var trigger = new AdaptationTrigger("high-cpu", 0.7, null, null);
        var ruleSpec = new AdaptationRuleSpec("scale-up", trigger, List.of());
        var goals = new DeploymentGoals(List.of(), List.of(), List.of(),
                null, List.of(), List.of(), List.of(), List.of(ruleSpec), null);

        recompiler.register("tenant-1", goals, Map.of(), factory);

        // Before any situation signal — rules exist but none active
        var snapshot = recompiler.getAdaptationSnapshot("tenant-1");
        assertEquals(1, snapshot.activeRules().size());
        assertEquals("scale-up", snapshot.activeRules().get(0).ruleName());
        assertEquals("high-cpu", snapshot.activeRules().get(0).situationId());
        assertFalse(snapshot.activeRules().get(0).active());
        assertTrue(snapshot.trackedSituations().isEmpty());
    }
}
```

Note: `TestUtil.setField` is a helper for injecting CDI fields in unit tests.
If no such utility exists, use direct field injection:
```java
private static void setField(Object target, String fieldName, Object value) {
    try {
        var field = target.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(target, value);
    } catch (ReflectiveOperationException e) {
        throw new RuntimeException(e);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode -o test -pl deployment -Dtest="DeploymentAdaptiveSituationRecompilerSnapshotTest" -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — `getAdaptationSnapshot` method does not exist, `AdaptationSnapshot` record does not exist.

- [ ] **Step 3: Implement the snapshot method and records**

Add to `DeploymentAdaptiveSituationRecompiler.java`:

```java
// --- Public snapshot records ---

public record AdaptationSnapshot(
        List<ActiveRuleInfo> activeRules,
        List<TrackedSituationInfo> trackedSituations) {

    public AdaptationSnapshot {
        activeRules = List.copyOf(activeRules);
        trackedSituations = List.copyOf(trackedSituations);
    }
}

public record ActiveRuleInfo(String ruleName, String situationId, boolean active) {}

public record TrackedSituationInfo(
        String situationId, double confidence,
        java.time.Instant since, java.time.Instant lastSignal) {}

// --- Snapshot query ---

public AdaptationSnapshot getAdaptationSnapshot(String tenancyId) {
    TenantAdaptationState state = tenantStates.get(tenancyId);
    if (state == null) {
        return new AdaptationSnapshot(List.of(), List.of());
    }

    synchronized (state) {
        List<ActiveRuleInfo> ruleInfos = state.rules().stream()
                .map(rule -> new ActiveRuleInfo(
                        rule.name(),
                        rule.trigger().situation(),
                        state.isRuleActive(rule.name())))
                .toList();

        List<TrackedSituationInfo> sitInfos = state.trackedSituationSnapshot().stream()
                .map(sit -> new TrackedSituationInfo(
                        sit.situationId(), sit.confidence(),
                        sit.since(), sit.lastSignal()))
                .toList();

        return new AdaptationSnapshot(ruleInfos, sitInfos);
    }
}
```

Add to `TenantAdaptationState.java` (two new package-private query methods):

```java
boolean isRuleActive(String ruleName) {
    return activePerRule.getOrDefault(ruleName, false);
}

List<ActiveSituation> trackedSituationSnapshot() {
    return List.copyOf(trackedSituations.values());
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode -o test -pl deployment -Dtest="DeploymentAdaptiveSituationRecompilerSnapshotTest" -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add deployment/src/main/java/io/casehub/ops/deployment/adaptation/DeploymentAdaptiveSituationRecompiler.java
git add deployment/src/main/java/io/casehub/ops/deployment/adaptation/TenantAdaptationState.java
git add deployment/src/test/java/io/casehub/ops/deployment/adaptation/DeploymentAdaptiveSituationRecompilerSnapshotTest.java
git commit -m "feat(#94): expose adaptation snapshot query on DeploymentAdaptiveSituationRecompiler

Add getAdaptationSnapshot(tenancyId) returning active rules and tracked
situations. Adds isRuleActive() and trackedSituationSnapshot() to
TenantAdaptationState for internal query support.

Refs #94"
```

---

### Task 2: Create OpsTopologyApi endpoint

**Files:**
- Create: `service/src/main/java/io/casehub/ops/service/api/OpsTopologyApi.java`
- Create: `service/src/test/java/io/casehub/ops/service/api/OpsTopologyApiTest.java`

**Interfaces:**
- Consumes:
  - `ReconciliationLoop.getDesired(String compositeKey)` → `DesiredStateGraph`
  - `ReconciliationLoop.tenantIds()` → `Set<String>` (composite keys: `tenancyId:appId:clusterId`)
  - `ActualStateAdapterRouter.readActual(DesiredStateGraph, String tenancyId)` → `ActualState`
  - `DeploymentAdaptiveSituationRecompiler.getAdaptationSnapshot(String tenancyId)` → `AdaptationSnapshot`
- Produces: `GET /api/ops/topology` → `TopologySnapshot` (JSON via platform framework)

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.ops.service.api;

import io.casehub.desiredstate.api.ActualState;
import io.casehub.desiredstate.api.ActualStateAdapterRouter;
import io.casehub.desiredstate.api.DesiredNode;
import io.casehub.desiredstate.api.DesiredStateGraph;
import io.casehub.desiredstate.api.HumanGating;
import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.api.NodeSpec;
import io.casehub.desiredstate.api.NodeStatus;
import io.casehub.desiredstate.api.NodeType;
import io.casehub.desiredstate.api.TargetStatus;
import io.casehub.desiredstate.runtime.ReconciliationLoop;
import io.casehub.ops.deployment.adaptation.DeploymentAdaptiveSituationRecompiler;
import io.casehub.ops.deployment.adaptation.DeploymentAdaptiveSituationRecompiler.AdaptationSnapshot;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;
import java.util.Set;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;

class OpsTopologyApiTest {

    private OpsTopologyApi api;
    private ReconciliationLoop reconciliationLoop;
    private ActualStateAdapterRouter actualStateRouter;
    private DeploymentAdaptiveSituationRecompiler adaptationRecompiler;

    @BeforeEach
    void setUp() {
        api = new OpsTopologyApi();
        reconciliationLoop = mock(ReconciliationLoop.class);
        actualStateRouter = mock(ActualStateAdapterRouter.class);
        adaptationRecompiler = mock(DeploymentAdaptiveSituationRecompiler.class);
        setField(api, "reconciliationLoop", reconciliationLoop);
        setField(api, "actualStateRouter", actualStateRouter);
        setField(api, "adaptationRecompiler", adaptationRecompiler);
    }

    @Test
    void returnsEmptySnapshotWhenNoLoopsForTenant() {
        when(reconciliationLoop.tenantIds()).thenReturn(Set.of());
        when(adaptationRecompiler.getAdaptationSnapshot("tenant-1"))
                .thenReturn(new AdaptationSnapshot(List.of(), List.of()));

        var result = api.getTopology("tenant-1");

        assertNotNull(result);
        assertTrue(result.loops().isEmpty());
        assertTrue(result.adaptation().activeRules().isEmpty());
    }

    @Test
    void returnsDesiredAndActualNodesForMatchingLoops() {
        String key = "tenant-1:app-1:cluster-1";
        when(reconciliationLoop.tenantIds()).thenReturn(Set.of(key, "other:app:cluster"));

        NodeId agentId = NodeId.of("fraud-detector");
        NodeSpec spec = mockNodeSpec("agent");
        DesiredNode node = new DesiredNode(agentId, spec, HumanGating.NONE);
        DesiredStateGraph graph = mock(DesiredStateGraph.class);
        when(graph.nodes()).thenReturn(Map.of(agentId, node));
        when(reconciliationLoop.getDesired(key)).thenReturn(graph);

        ActualState actual = new ActualState(Map.of(agentId, NodeStatus.PRESENT));
        when(actualStateRouter.readActual(graph, "tenant-1")).thenReturn(actual);

        when(adaptationRecompiler.getAdaptationSnapshot("tenant-1"))
                .thenReturn(new AdaptationSnapshot(List.of(), List.of()));

        var result = api.getTopology("tenant-1");

        assertEquals(1, result.loops().size());
        var loop = result.loops().get(0);
        assertEquals(key, loop.loopKey());
        assertEquals(1, loop.desiredNodes().size());
        assertEquals("fraud-detector", loop.desiredNodes().get(0).nodeId());
        assertEquals("agent", loop.desiredNodes().get(0).nodeType());
        assertEquals("ACTIVE", loop.desiredNodes().get(0).targetStatus());
        assertFalse(loop.desiredNodes().get(0).adapted());
        assertEquals(1, loop.actualStatuses().size());
        assertEquals("PRESENT", loop.actualStatuses().get(0).status());
    }

    @Test
    void marksScaledNodesAsAdapted() {
        String key = "tenant-1:app-1:cluster-1";
        when(reconciliationLoop.tenantIds()).thenReturn(Set.of(key));

        NodeId baseId = NodeId.of("fraud-detector");
        NodeId scaledId = NodeId.of("fraud-detector~2");
        NodeSpec spec = mockNodeSpec("agent");
        DesiredNode baseNode = new DesiredNode(baseId, spec, HumanGating.NONE);
        DesiredNode scaledNode = new DesiredNode(scaledId, spec, HumanGating.NONE);
        DesiredStateGraph graph = mock(DesiredStateGraph.class);
        when(graph.nodes()).thenReturn(Map.of(baseId, baseNode, scaledId, scaledNode));
        when(reconciliationLoop.getDesired(key)).thenReturn(graph);
        when(actualStateRouter.readActual(graph, "tenant-1"))
                .thenReturn(new ActualState(Map.of()));
        when(adaptationRecompiler.getAdaptationSnapshot("tenant-1"))
                .thenReturn(new AdaptationSnapshot(List.of(), List.of()));

        var result = api.getTopology("tenant-1");

        var nodes = result.loops().get(0).desiredNodes();
        assertEquals(2, nodes.size());
        var base = nodes.stream().filter(n -> n.nodeId().equals("fraud-detector")).findFirst().orElseThrow();
        var scaled = nodes.stream().filter(n -> n.nodeId().equals("fraud-detector~2")).findFirst().orElseThrow();
        assertFalse(base.adapted());
        assertTrue(scaled.adapted());
    }

    private static NodeSpec mockNodeSpec(String nodeType) {
        NodeSpec spec = mock(NodeSpec.class);
        when(spec.nodeType()).thenReturn(NodeType.of(nodeType));
        when(spec.humanGating()).thenReturn(HumanGating.NONE);
        return spec;
    }

    private static void setField(Object target, String fieldName, Object value) {
        try {
            var field = target.getClass().getDeclaredField(fieldName);
            field.setAccessible(true);
            field.set(target, value);
        } catch (ReflectiveOperationException e) {
            throw new RuntimeException(e);
        }
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode -o test -pl service -Dtest="OpsTopologyApiTest" -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — `OpsTopologyApi` class does not exist.

- [ ] **Step 3: Implement OpsTopologyApi**

```java
package io.casehub.ops.service.api;

import io.casehub.desiredstate.api.ActualState;
import io.casehub.desiredstate.api.ActualStateAdapterRouter;
import io.casehub.desiredstate.api.DesiredNode;
import io.casehub.desiredstate.api.DesiredStateGraph;
import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.runtime.ReconciliationLoop;
import io.casehub.ops.deployment.adaptation.DeploymentAdaptiveSituationRecompiler;
import io.casehub.ops.deployment.adaptation.DeploymentAdaptiveSituationRecompiler.AdaptationSnapshot;
import io.casehub.platform.api.mcp.ContextParam;
import io.casehub.platform.api.mcp.McpDomain;
import io.casehub.platform.api.mcp.PlatformQuery;
import io.casehub.platform.api.mcp.RestPath;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.time.Instant;
import java.util.ArrayList;
import java.util.List;

@McpDomain(value = "ops/topology", app = "ops", basePath = "/api/ops/topology",
        summary = "Topology observation — desired/actual state and adaptation status")
@ApplicationScoped
public class OpsTopologyApi {

    @Inject ReconciliationLoop reconciliationLoop;
    @Inject ActualStateAdapterRouter actualStateRouter;
    @Inject DeploymentAdaptiveSituationRecompiler adaptationRecompiler;

    @PlatformQuery("Get current topology snapshot — desired nodes, actual statuses, adaptation state")
    @RestPath("/")
    public TopologySnapshot getTopology(@ContextParam("tenancyId") String tenancyId) {
        String prefix = tenancyId + ":";
        List<LoopTopology> loops = new ArrayList<>();

        for (String key : reconciliationLoop.tenantIds()) {
            if (!key.startsWith(prefix)) {
                continue;
            }

            DesiredStateGraph desired = reconciliationLoop.getDesired(key);
            if (desired == null) {
                continue;
            }

            List<TopologyNode> desiredNodes = new ArrayList<>();
            for (var entry : desired.nodes().entrySet()) {
                NodeId nodeId = entry.getKey();
                DesiredNode node = entry.getValue();
                boolean adapted = nodeId.value().contains("~");
                desiredNodes.add(new TopologyNode(
                        nodeId.value(),
                        node.type().value(),
                        node.targetStatus().name(),
                        adapted));
            }

            ActualState actual = actualStateRouter.readActual(desired, tenancyId);
            List<NodeStatusEntry> actualStatuses = new ArrayList<>();
            for (var entry : actual.statuses().entrySet()) {
                actualStatuses.add(new NodeStatusEntry(
                        entry.getKey().value(),
                        entry.getValue().name()));
            }

            loops.add(new LoopTopology(key, desiredNodes, actualStatuses));
        }

        AdaptationSnapshot adaptation = adaptationRecompiler.getAdaptationSnapshot(tenancyId);
        TopologyAdaptation topologyAdaptation = new TopologyAdaptation(
                adaptation.activeRules().stream()
                        .map(r -> new AdaptationRuleStatus(r.ruleName(), r.situationId(), r.active()))
                        .toList(),
                adaptation.trackedSituations().stream()
                        .map(s -> new SituationStatus(s.situationId(), s.confidence(), s.since(), s.lastSignal()))
                        .toList());

        return new TopologySnapshot(loops, topologyAdaptation);
    }

    // --- Response records ---

    public record TopologySnapshot(List<LoopTopology> loops, TopologyAdaptation adaptation) {}

    public record LoopTopology(String loopKey, List<TopologyNode> desiredNodes,
                               List<NodeStatusEntry> actualStatuses) {}

    public record TopologyNode(String nodeId, String nodeType, String targetStatus, boolean adapted) {}

    public record NodeStatusEntry(String nodeId, String status) {}

    public record TopologyAdaptation(List<AdaptationRuleStatus> activeRules,
                                     List<SituationStatus> trackedSituations) {}

    public record AdaptationRuleStatus(String ruleName, String situationId, boolean active) {}

    public record SituationStatus(String situationId, double confidence,
                                  Instant since, Instant lastSignal) {}
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode -o test -pl service -Dtest="OpsTopologyApiTest" -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 5: Run full build to verify no regressions**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```bash
git add service/src/main/java/io/casehub/ops/service/api/OpsTopologyApi.java
git add service/src/test/java/io/casehub/ops/service/api/OpsTopologyApiTest.java
git commit -m "feat(#94): topology observation REST endpoint

GET /api/ops/topology returns desired nodes (with adaptation markers),
actual statuses per reconciliation loop, and active adaptation rules and
tracked situations for the requesting tenant.

Refs #94"
```

## References

- [GitHub #94](https://github.com/casehubio/casehub-ops/issues/94) — focal issue
- `service/src/main/java/io/casehub/ops/service/api/OpsReconciliationApi.java` — existing API pattern for querying ReconciliationLoop
- `deployment/src/main/java/io/casehub/ops/deployment/adaptation/DeploymentAdaptiveSituationRecompiler.java` — adaptation state holder
- `deployment/src/main/java/io/casehub/ops/deployment/adaptation/TenantAdaptationState.java` — per-tenant rule/situation tracking
- `desiredstate/runtime-core/src/main/java/io/casehub/desiredstate/runtime/ReconciliationLoop.java:327` — getDesired() method
- `desiredstate/runtime/src/main/java/io/casehub/desiredstate/runtime/CdiActualStateAdapterRouter.java` — injectable actual state adapter
- `desiredstate/api/src/main/java/io/casehub/desiredstate/api/ActualStateAdapterRouter.java` — readActual interface
