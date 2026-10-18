# Claudony Desiredstate SPI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #140 — feat: claudony NodeProvisioner — GoalCompiler + ActualStateAdapter targeting claudony fleet API
**Issue group:** #140

**Goal:** Make the claudony module a standalone desiredstate domain with full SPI Quad (GoalCompiler, ActualStateAdapter, NodeProvisioner, FaultPolicy, EventSource) for declarative fleet management.

**Architecture:** Two node types (`claudony_app`, `claudony_pool`) as a sealed hierarchy in ops-api. All SPI implementations in the claudony module using the existing `ClaudonyFleetClient`. Pools are provisioned via read-modify-write on the parent app spec until claudony adds standalone pool CRUD. The existing `ClaudonyDeploymentTarget` (deployment module's SPI) coexists unchanged — different concern, different callers.

**Tech Stack:** Java 21, Quarkus CDI, casehub-desiredstate-api SPIs, Jackson YAML, JUnit 5, AssertJ

## Global Constraints

- Java 21, Maven build: `mvn --batch-mode install`
- Module type: Jandex library — no Quarkus extension, no deployment module
- CDI activation: classpath presence triggers bean discovery via `@ApplicationScoped`
- `handledTypes()` sets MUST be disjoint from all other domain modules — runtime throws on overlap
- tenancyId == claudony namespace (identity mapping)
- No `namespace` field on `ClaudonyAppNodeSpec` — provisioner sets `namespace = context.tenancyId()`
- Test scope: `casehub-ops-testing` is never compile or runtime

---

## Batch 1: Foundation — API types and shared utilities

### Task 1: API types, ClaudonySpecHashStore, ClaudonyHealthMapper

**Files:**
- Create: `api/src/main/java/io/casehub/ops/api/claudony/ClaudonyNodeSpec.java`
- Create: `api/src/main/java/io/casehub/ops/api/claudony/ClaudonyAppNodeSpec.java`
- Create: `api/src/main/java/io/casehub/ops/api/claudony/ClaudonyPoolNodeSpec.java`
- Create: `api/src/main/java/io/casehub/ops/api/claudony/ClaudonyNodeTypes.java`
- Create: `api/src/main/java/io/casehub/ops/api/claudony/ClaudonyGoalEntry.java`
- Create: `api/src/main/java/io/casehub/ops/api/claudony/ClaudonyGoals.java`
- Create: `api/src/main/java/io/casehub/ops/api/claudony/ClaudonyReviewSpec.java`
- Create: `claudony/src/main/java/io/casehub/ops/claudony/ClaudonySpecHashStore.java`
- Create: `claudony/src/main/java/io/casehub/ops/claudony/ClaudonyHealthMapper.java`
- Modify: `claudony/src/main/java/io/casehub/ops/claudony/ClaudonyDeploymentTarget.java` — delegate `mapHealth()` to `ClaudonyHealthMapper`
- Test: `claudony/src/test/java/io/casehub/ops/claudony/ClaudonyNodeSpecTest.java`
- Test: `claudony/src/test/java/io/casehub/ops/claudony/ClaudonySpecHashStoreTest.java`

**Interfaces:**
- Consumes: `NodeSpec` from `casehub-desiredstate-api`, `NodeId` from `casehub-desiredstate-api`, `HealthStatus` from `casehub-platform-api`, `NodeStatus` from `casehub-desiredstate-api`
- Produces: `ClaudonyNodeSpec` sealed interface, `ClaudonyAppNodeSpec` record, `ClaudonyPoolNodeSpec` record, `ClaudonyNodeTypes` constants (`APP = NodeType.of("claudony_app")`, `POOL = NodeType.of("claudony_pool")`), `ClaudonyGoalEntry<S extends ClaudonyNodeSpec>` record, `ClaudonyGoals` record, `ClaudonyReviewSpec` record, `ClaudonySpecHashStore` with `record(NodeId, NodeSpec)`, `remove(NodeId)`, `hasDrifted(NodeId, NodeSpec)`, `ClaudonyHealthMapper.toNodeStatus(HealthStatus)`

- [ ] **Step 1: Write tests for API types**

```java
// ClaudonyNodeSpecTest.java
package io.casehub.ops.claudony;

import io.casehub.ops.api.claudony.*;
import org.junit.jupiter.api.Test;
import java.util.Map;
import static org.assertj.core.api.Assertions.assertThat;

class ClaudonyNodeSpecTest {

    @Test
    void appNodeSpec_nodeType_and_nodeId() {
        var spec = new ClaudonyAppNodeSpec("my-app", "My App", "desc", "img:1.0", Map.of("k", "v"));
        assertThat(spec.nodeType()).isEqualTo(ClaudonyNodeTypes.APP);
        assertThat(spec.nodeId()).isEqualTo("claudony_app:my-app");
    }

    @Test
    void poolNodeSpec_nodeType_and_nodeId() {
        var spec = new ClaudonyPoolNodeSpec("my-app", "agent-1", 1, 3);
        assertThat(spec.nodeType()).isEqualTo(ClaudonyNodeTypes.POOL);
        assertThat(spec.nodeId()).isEqualTo("claudony_pool:my-app:agent-1");
    }

    @Test
    void sealedInterface_permits_only_app_and_pool() {
        var permitted = ClaudonyNodeSpec.class.getPermittedSubclasses();
        assertThat(permitted).containsExactlyInAnyOrder(
            ClaudonyAppNodeSpec.class, ClaudonyPoolNodeSpec.class);
    }

    @Test
    void goalEntry_defensiveCopy_dependsOn() {
        var entry = new ClaudonyGoalEntry<>(
            new ClaudonyAppNodeSpec("a", "A", null, "img", Map.of()), null);
        assertThat(entry.dependsOn()).isEmpty();
    }

    @Test
    void reviewSpec_rejectsNulls() {
        org.junit.jupiter.api.Assertions.assertThrows(NullPointerException.class,
            () -> new ClaudonyReviewSpec(null, "reason"));
    }

    @Test
    void appNodeSpec_defensiveCopy_config() {
        var mutable = new java.util.HashMap<String, String>();
        mutable.put("k", "v");
        var spec = new ClaudonyAppNodeSpec("a", "A", null, "img", mutable);
        mutable.put("k2", "v2");
        assertThat(spec.config()).doesNotContainKey("k2");
    }
}
```

- [ ] **Step 2: Run tests — verify they fail (classes don't exist yet)**

Run: `mvn --batch-mode -o test -pl claudony -Dtest=ClaudonyNodeSpecTest`
Expected: Compilation failure — types not found

- [ ] **Step 3: Create API types**

Create `ClaudonyNodeSpec.java`:
```java
package io.casehub.ops.api.claudony;

import io.casehub.desiredstate.api.NodeSpec;
import io.casehub.desiredstate.api.NodeType;

public sealed interface ClaudonyNodeSpec extends NodeSpec
    permits ClaudonyAppNodeSpec, ClaudonyPoolNodeSpec {
    String nodeId();
    @Override NodeType nodeType();
}
```

Create `ClaudonyAppNodeSpec.java`:
```java
package io.casehub.ops.api.claudony;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
import io.casehub.desiredstate.api.NodeType;
import io.casehub.desiredstate.api.annotations.NodeTypeId;
import java.util.Map;

@NodeTypeId("claudony_app")
@JsonIgnoreProperties(ignoreUnknown = true)
public record ClaudonyAppNodeSpec(
    String name,
    String displayName,
    String description,
    String image,
    Map<String, String> config
) implements ClaudonyNodeSpec {
    public ClaudonyAppNodeSpec {
        if (name == null || name.isBlank()) throw new IllegalArgumentException("name required");
        config = config != null ? Map.copyOf(config) : Map.of();
    }
    @Override public NodeType nodeType() { return ClaudonyNodeTypes.APP; }
    @Override public String nodeId() { return "claudony_app:" + name; }
}
```

Create `ClaudonyPoolNodeSpec.java`:
```java
package io.casehub.ops.api.claudony;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
import io.casehub.desiredstate.api.NodeType;
import io.casehub.desiredstate.api.annotations.NodeTypeId;

@NodeTypeId("claudony_pool")
@JsonIgnoreProperties(ignoreUnknown = true)
public record ClaudonyPoolNodeSpec(
    String appName,
    String agentId,
    int minActive,
    int maxActive
) implements ClaudonyNodeSpec {
    public ClaudonyPoolNodeSpec {
        if (appName == null || appName.isBlank()) throw new IllegalArgumentException("appName required");
        if (agentId == null || agentId.isBlank()) throw new IllegalArgumentException("agentId required");
    }
    @Override public NodeType nodeType() { return ClaudonyNodeTypes.POOL; }
    @Override public String nodeId() { return "claudony_pool:" + appName + ":" + agentId; }
}
```

Create `ClaudonyNodeTypes.java`:
```java
package io.casehub.ops.api.claudony;

import io.casehub.desiredstate.api.NodeType;

public final class ClaudonyNodeTypes {
    public static final NodeType APP = NodeType.of("claudony_app");
    public static final NodeType POOL = NodeType.of("claudony_pool");
    private ClaudonyNodeTypes() {}
}
```

Create `ClaudonyGoalEntry.java`:
```java
package io.casehub.ops.api.claudony;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
import java.util.List;

@JsonIgnoreProperties(ignoreUnknown = true)
public record ClaudonyGoalEntry<S extends ClaudonyNodeSpec>(S spec, List<String> dependsOn) {
    public ClaudonyGoalEntry {
        dependsOn = dependsOn != null ? List.copyOf(dependsOn) : List.of();
    }
}
```

Create `ClaudonyGoals.java`:
```java
package io.casehub.ops.api.claudony;

import java.util.List;

public record ClaudonyGoals(
    List<ClaudonyGoalEntry<ClaudonyAppNodeSpec>> apps,
    List<ClaudonyGoalEntry<ClaudonyPoolNodeSpec>> pools
) {
    public ClaudonyGoals {
        apps = apps != null ? List.copyOf(apps) : List.of();
        pools = pools != null ? List.copyOf(pools) : List.of();
    }
}
```

Create `ClaudonyReviewSpec.java`:
```java
package io.casehub.ops.api.claudony;

import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.api.NodeSpec;
import io.casehub.desiredstate.api.NodeType;
import java.util.Objects;

public record ClaudonyReviewSpec(NodeId faultedNode, String reason) implements NodeSpec {
    public ClaudonyReviewSpec {
        Objects.requireNonNull(faultedNode, "faultedNode");
        Objects.requireNonNull(reason, "reason");
    }
    @Override public NodeType nodeType() { return NodeType.of("claudony_review"); }
}
```

- [ ] **Step 4: Write tests for ClaudonySpecHashStore**

```java
// ClaudonySpecHashStoreTest.java
package io.casehub.ops.claudony;

import io.casehub.desiredstate.api.NodeId;
import io.casehub.ops.api.claudony.ClaudonyAppNodeSpec;
import org.junit.jupiter.api.Test;
import java.util.Map;
import static org.assertj.core.api.Assertions.assertThat;

class ClaudonySpecHashStoreTest {

    private final ClaudonySpecHashStore store = new ClaudonySpecHashStore();

    @Test
    void unknownNode_hasDrifted_returnsTrue() {
        var id = NodeId.of("claudony_app:x");
        var spec = new ClaudonyAppNodeSpec("x", "X", null, "img", Map.of());
        assertThat(store.hasDrifted(id, spec)).isTrue();
    }

    @Test
    void recordedNode_sameSpec_notDrifted() {
        var id = NodeId.of("claudony_app:x");
        var spec = new ClaudonyAppNodeSpec("x", "X", null, "img", Map.of());
        store.record(id, spec);
        assertThat(store.hasDrifted(id, spec)).isFalse();
    }

    @Test
    void recordedNode_differentSpec_hasDrifted() {
        var id = NodeId.of("claudony_app:x");
        var spec1 = new ClaudonyAppNodeSpec("x", "X", null, "img:1", Map.of());
        var spec2 = new ClaudonyAppNodeSpec("x", "X", null, "img:2", Map.of());
        store.record(id, spec1);
        assertThat(store.hasDrifted(id, spec2)).isTrue();
    }

    @Test
    void removedNode_hasDrifted_returnsTrue() {
        var id = NodeId.of("claudony_app:x");
        var spec = new ClaudonyAppNodeSpec("x", "X", null, "img", Map.of());
        store.record(id, spec);
        store.remove(id);
        assertThat(store.hasDrifted(id, spec)).isTrue();
    }
}
```

- [ ] **Step 5: Create ClaudonySpecHashStore and ClaudonyHealthMapper**

Create `ClaudonySpecHashStore.java`:
```java
package io.casehub.ops.claudony;

import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.api.NodeSpec;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.concurrent.ConcurrentHashMap;

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

Create `ClaudonyHealthMapper.java`:
```java
package io.casehub.ops.claudony;

import io.casehub.desiredstate.api.NodeStatus;
import io.casehub.platform.api.registry.HealthStatus;

public final class ClaudonyHealthMapper {
    private ClaudonyHealthMapper() {}

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

- [ ] **Step 6: Update ClaudonyDeploymentTarget to delegate to ClaudonyHealthMapper**

In `ClaudonyDeploymentTarget.java`, change `mapHealth()`:
```java
static NodeStatus mapHealth(final HealthStatus health) {
    return ClaudonyHealthMapper.toNodeStatus(health);
}
```

- [ ] **Step 7: Run all tests — verify they pass**

Run: `mvn --batch-mode -o test -pl api,claudony`
Expected: All tests pass, including existing `ClaudonyDeploymentTargetTest`

- [ ] **Step 8: Commit**

```bash
git add api/src/main/java/io/casehub/ops/api/claudony/ claudony/src/main/java/io/casehub/ops/claudony/ClaudonySpecHashStore.java claudony/src/main/java/io/casehub/ops/claudony/ClaudonyHealthMapper.java claudony/src/main/java/io/casehub/ops/claudony/ClaudonyDeploymentTarget.java claudony/src/test/java/io/casehub/ops/claudony/ClaudonyNodeSpecTest.java claudony/src/test/java/io/casehub/ops/claudony/ClaudonySpecHashStoreTest.java
git commit -m "feat(#140): claudony API types, spec hash store, and health mapper"
```

---

## Batch 2: Core SPI — GoalCompiler

### Task 2: ClaudonyGoalCompiler

**Files:**
- Create: `claudony/src/main/java/io/casehub/ops/claudony/ClaudonyGoalCompiler.java`
- Test: `claudony/src/test/java/io/casehub/ops/claudony/ClaudonyGoalCompilerTest.java`

**Interfaces:**
- Consumes: `ClaudonyGoals`, `ClaudonyGoalEntry`, `ClaudonyAppNodeSpec`, `ClaudonyPoolNodeSpec` from Task 1
- Produces: `ClaudonyGoalCompiler` implementing `GoalCompiler<ClaudonyGoals>` with `compile(ClaudonyGoals, DesiredStateGraphFactory)` returning `CompilationResult`

- [ ] **Step 1: Write failing tests**

```java
package io.casehub.ops.claudony;

import io.casehub.desiredstate.api.*;
import io.casehub.ops.api.claudony.*;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import static org.assertj.core.api.Assertions.assertThat;

class ClaudonyGoalCompilerTest {

    private final ClaudonyGoalCompiler compiler = new ClaudonyGoalCompiler();
    private final DesiredStateGraphFactory factory = new DefaultDesiredStateGraphFactory();

    @Test
    void compiles_apps_into_desiredNodes() {
        var goals = new ClaudonyGoals(
            List.of(new ClaudonyGoalEntry<>(
                new ClaudonyAppNodeSpec("app-a", "App A", null, "img:1", Map.of()), List.of())),
            List.of());

        var result = compiler.compile(goals, factory);
        var graph = result.graphs().getFirst();

        assertThat(graph.nodes()).hasSize(1);
        assertThat(graph.nodes().get(NodeId.of("claudony_app:app-a"))).isNotNull();
        assertThat(graph.nodes().get(NodeId.of("claudony_app:app-a")).spec())
            .isInstanceOf(ClaudonyAppNodeSpec.class);
    }

    @Test
    void compiles_pools_with_dependency_on_parent_app() {
        var goals = new ClaudonyGoals(
            List.of(new ClaudonyGoalEntry<>(
                new ClaudonyAppNodeSpec("app-a", "App A", null, "img:1", Map.of()), List.of())),
            List.of(new ClaudonyGoalEntry<>(
                new ClaudonyPoolNodeSpec("app-a", "agent-1", 1, 3), List.of())));

        var result = compiler.compile(goals, factory);
        var graph = result.graphs().getFirst();

        assertThat(graph.nodes()).hasSize(2);
        var poolId = NodeId.of("claudony_pool:app-a:agent-1");
        var appId = NodeId.of("claudony_app:app-a");
        assertThat(graph.dependenciesOf(poolId)).contains(appId);
    }

    @Test
    void honours_explicit_dependsOn_edges() {
        var goals = new ClaudonyGoals(
            List.of(
                new ClaudonyGoalEntry<>(
                    new ClaudonyAppNodeSpec("app-a", "A", null, "img", Map.of()), List.of()),
                new ClaudonyGoalEntry<>(
                    new ClaudonyAppNodeSpec("app-b", "B", null, "img", Map.of()),
                    List.of("claudony_app:app-a"))),
            List.of());

        var result = compiler.compile(goals, factory);
        var graph = result.graphs().getFirst();

        var bId = NodeId.of("claudony_app:app-b");
        var aId = NodeId.of("claudony_app:app-a");
        assertThat(graph.dependenciesOf(bId)).contains(aId);
    }

    @Test
    void empty_goals_produce_empty_graph() {
        var goals = new ClaudonyGoals(List.of(), List.of());
        var result = compiler.compile(goals, factory);
        assertThat(result.graphs().getFirst().nodes()).isEmpty();
    }
}
```

- [ ] **Step 2: Run tests — verify compilation failure**

Run: `mvn --batch-mode -o test -pl claudony -Dtest=ClaudonyGoalCompilerTest`
Expected: Compilation failure — `ClaudonyGoalCompiler` not found

- [ ] **Step 3: Implement ClaudonyGoalCompiler**

```java
package io.casehub.ops.claudony;

import io.casehub.desiredstate.api.*;
import io.casehub.desiredstate.annotations.DesiredStateQualifier;
import io.casehub.ops.api.claudony.*;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.ArrayList;
import java.util.List;

@ApplicationScoped
@DesiredStateQualifier(namespace = "claudony")
public class ClaudonyGoalCompiler implements GoalCompiler<ClaudonyGoals> {

    @Override
    public CompilationResult compile(ClaudonyGoals goals, DesiredStateGraphFactory factory) {
        List<DesiredNode> nodes = new ArrayList<>();
        List<Dependency> dependencies = new ArrayList<>();

        for (var entry : goals.apps()) {
            var spec = entry.spec();
            nodes.add(new DesiredNode(NodeId.of(spec.nodeId()), spec, HumanGating.NONE));
            for (String dep : entry.dependsOn()) {
                dependencies.add(new Dependency(NodeId.of(spec.nodeId()), NodeId.of(dep)));
            }
        }

        for (var entry : goals.pools()) {
            var spec = entry.spec();
            var poolId = NodeId.of(spec.nodeId());
            var appId = NodeId.of("claudony_app:" + spec.appName());
            nodes.add(new DesiredNode(poolId, spec, HumanGating.NONE));
            dependencies.add(new Dependency(poolId, appId));
            for (String dep : entry.dependsOn()) {
                dependencies.add(new Dependency(poolId, NodeId.of(dep)));
            }
        }

        return CompilationResult.single(factory.of(nodes, dependencies));
    }
}
```

- [ ] **Step 4: Run tests — verify they pass**

Run: `mvn --batch-mode -o test -pl claudony -Dtest=ClaudonyGoalCompilerTest`
Expected: All 4 tests pass

- [ ] **Step 5: Commit**

```bash
git add claudony/src/main/java/io/casehub/ops/claudony/ClaudonyGoalCompiler.java claudony/src/test/java/io/casehub/ops/claudony/ClaudonyGoalCompilerTest.java
git commit -m "feat(#140): ClaudonyGoalCompiler — YAML goals to DesiredStateGraph"
```

---

## Batch 3: Core SPI — ActualStateAdapter and NodeProvisioner

### Task 3: ClaudonyActualStateAdapter

**Files:**
- Create: `claudony/src/main/java/io/casehub/ops/claudony/ClaudonyActualStateAdapter.java`
- Test: `claudony/src/test/java/io/casehub/ops/claudony/ClaudonyActualStateAdapterTest.java`

**Interfaces:**
- Consumes: `ClaudonyFleetClient`, `ClaudonySpecHashStore`, `ClaudonyHealthMapper`, `ClaudonyNodeTypes`, `ClaudonyAppNodeSpec`, `ClaudonyPoolNodeSpec` from Tasks 1-2
- Produces: `ClaudonyActualStateAdapter` implementing `ActualStateAdapter` with `readActual(DesiredStateGraph, String)` returning `ActualState` and `handledTypes()` returning `{CLAUDONY_APP, CLAUDONY_POOL}`

- [ ] **Step 1: Write failing tests**

```java
package io.casehub.ops.claudony;

import io.casehub.desiredstate.api.*;
import io.casehub.ops.api.claudony.*;
import io.casehub.platform.api.registry.HealthStatus;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.*;
import static org.assertj.core.api.Assertions.assertThat;

class ClaudonyActualStateAdapterTest {

    @Test
    void handledTypes_returns_app_and_pool() {
        var adapter = new ClaudonyActualStateAdapter(
            new StubFleetClient(), new ClaudonySpecHashStore());
        assertThat(adapter.handledTypes()).containsExactlyInAnyOrder(
            ClaudonyNodeTypes.APP, ClaudonyNodeTypes.POOL);
    }

    @Test
    void app_healthy_returns_present() {
        var client = new StubFleetClient();
        client.apps.put("my-app", appInfo("my-app", HealthStatus.HEALTHY));
        var adapter = new ClaudonyActualStateAdapter(client, new ClaudonySpecHashStore());

        var graph = graphWith(new ClaudonyAppNodeSpec("my-app", "My App", null, "img", Map.of()));
        var actual = adapter.readActual(graph, "ns1");

        assertThat(actual.statuses().get(NodeId.of("claudony_app:my-app")))
            .isEqualTo(NodeStatus.PRESENT);
    }

    @Test
    void app_absent_returns_absent() {
        var adapter = new ClaudonyActualStateAdapter(
            new StubFleetClient(), new ClaudonySpecHashStore());

        var graph = graphWith(new ClaudonyAppNodeSpec("missing", "M", null, "img", Map.of()));
        var actual = adapter.readActual(graph, "ns1");

        assertThat(actual.statuses().get(NodeId.of("claudony_app:missing")))
            .isEqualTo(NodeStatus.ABSENT);
    }

    @Test
    void app_suspended_returns_suspended() {
        var client = new StubFleetClient();
        client.apps.put("my-app", appInfo("my-app", HealthStatus.SUSPENDED));
        var adapter = new ClaudonyActualStateAdapter(client, new ClaudonySpecHashStore());

        var graph = graphWith(new ClaudonyAppNodeSpec("my-app", "App", null, "img", Map.of()));
        var actual = adapter.readActual(graph, "ns1");

        assertThat(actual.statuses().get(NodeId.of("claudony_app:my-app")))
            .isEqualTo(NodeStatus.SUSPENDED);
    }

    @Test
    void pool_present_when_matching() {
        var client = new StubFleetClient();
        client.apps.put("my-app", appInfoWithPool("my-app", "agent-1", 1, 3));
        var store = new ClaudonySpecHashStore();
        var poolSpec = new ClaudonyPoolNodeSpec("my-app", "agent-1", 1, 3);
        store.record(NodeId.of(poolSpec.nodeId()), poolSpec);
        var adapter = new ClaudonyActualStateAdapter(client, store);

        var graph = graphWith(poolSpec);
        var actual = adapter.readActual(graph, "ns1");

        assertThat(actual.statuses().get(NodeId.of("claudony_pool:my-app:agent-1")))
            .isEqualTo(NodeStatus.PRESENT);
    }

    @Test
    void pool_drifted_when_minActive_differs() {
        var client = new StubFleetClient();
        client.apps.put("my-app", appInfoWithPool("my-app", "agent-1", 2, 3));
        var store = new ClaudonySpecHashStore();
        var poolSpec = new ClaudonyPoolNodeSpec("my-app", "agent-1", 1, 3);
        store.record(NodeId.of(poolSpec.nodeId()), poolSpec);
        var adapter = new ClaudonyActualStateAdapter(client, store);

        var graph = graphWith(poolSpec);
        var actual = adapter.readActual(graph, "ns1");

        assertThat(actual.statuses().get(NodeId.of("claudony_pool:my-app:agent-1")))
            .isEqualTo(NodeStatus.DRIFTED);
    }

    @Test
    void pool_absent_when_parent_app_missing() {
        var adapter = new ClaudonyActualStateAdapter(
            new StubFleetClient(), new ClaudonySpecHashStore());

        var graph = graphWith(new ClaudonyPoolNodeSpec("missing-app", "agent-1", 1, 3));
        var actual = adapter.readActual(graph, "ns1");

        assertThat(actual.statuses().get(NodeId.of("claudony_pool:missing-app:agent-1")))
            .isEqualTo(NodeStatus.ABSENT);
    }

    // --- helpers ---

    private DesiredStateGraph graphWith(ClaudonyNodeSpec spec) {
        var factory = new DefaultDesiredStateGraphFactory();
        return factory.of(
            List.of(new DesiredNode(NodeId.of(spec.nodeId()), spec, HumanGating.NONE)),
            List.of());
    }

    private ClaudonyAppInfo appInfo(String name, HealthStatus health) {
        return new ClaudonyAppInfo(name, "ns1", name, null, "img", Map.of(),
            health, Instant.now(), List.of());
    }

    private ClaudonyAppInfo appInfoWithPool(String name, String agentId, int min, int max) {
        return new ClaudonyAppInfo(name, "ns1", name, null, "img", Map.of(),
            HealthStatus.HEALTHY, Instant.now(),
            List.of(new ClaudonyAppInfo.AppPoolInfo(agentId, min, max)));
    }

    static class StubFleetClient implements ClaudonyFleetClient {
        final Map<String, ClaudonyAppInfo> apps = new HashMap<>();

        @Override public ClaudonyAppInfo registerApp(ClaudonyAppSpec spec) { return null; }
        @Override public ClaudonyAppInfo updateApp(String name, ClaudonyAppSpec spec) { return null; }
        @Override public void deregisterApp(String name) {}
        @Override public Optional<ClaudonyAppInfo> getApp(String name) { return Optional.ofNullable(apps.get(name)); }
        @Override public List<ClaudonyAppInfo> listApps(String namespace) { return List.copyOf(apps.values()); }
        @Override public void suspendApp(String name) {}
        @Override public void resumeApp(String name) {}
        @Override public void heartbeat(String name) {}
    }
}
```

- [ ] **Step 2: Run tests — verify compilation failure**

Run: `mvn --batch-mode -o test -pl claudony -Dtest=ClaudonyActualStateAdapterTest`
Expected: Compilation failure — `ClaudonyActualStateAdapter` not found

- [ ] **Step 3: Implement ClaudonyActualStateAdapter**

```java
package io.casehub.ops.claudony;

import io.casehub.desiredstate.api.*;
import io.casehub.ops.api.claudony.*;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.*;

@ApplicationScoped
public class ClaudonyActualStateAdapter implements ActualStateAdapter {

    private static final Set<NodeType> HANDLED = Set.of(ClaudonyNodeTypes.APP, ClaudonyNodeTypes.POOL);

    private final ClaudonyFleetClient fleetClient;
    private final ClaudonySpecHashStore specHashStore;

    @Inject
    public ClaudonyActualStateAdapter(ClaudonyFleetClient fleetClient,
                                       ClaudonySpecHashStore specHashStore) {
        this.fleetClient = fleetClient;
        this.specHashStore = specHashStore;
    }

    @Override
    public Set<NodeType> handledTypes() { return HANDLED; }

    @Override
    public ActualState readActual(DesiredStateGraph desired, String tenancyId) {
        var appsByName = new HashMap<String, ClaudonyAppInfo>();
        for (var info : fleetClient.listApps(tenancyId)) {
            appsByName.put(info.name(), info);
        }

        var statuses = new HashMap<NodeId, NodeStatus>();
        for (var entry : desired.nodes().entrySet()) {
            var node = entry.getValue();
            if (!HANDLED.contains(node.type())) continue;
            statuses.put(entry.getKey(), checkNode(node, appsByName));
        }
        return new ActualState(statuses);
    }

    private NodeStatus checkNode(DesiredNode node, Map<String, ClaudonyAppInfo> appsByName) {
        return switch (node.spec()) {
            case ClaudonyAppNodeSpec spec -> checkApp(node, spec, appsByName);
            case ClaudonyPoolNodeSpec spec -> checkPool(node, spec, appsByName);
            default -> NodeStatus.UNKNOWN;
        };
    }

    private NodeStatus checkApp(DesiredNode node, ClaudonyAppNodeSpec spec,
                                 Map<String, ClaudonyAppInfo> appsByName) {
        var info = appsByName.get(spec.name());
        if (info == null) return NodeStatus.ABSENT;

        var external = ClaudonyHealthMapper.toNodeStatus(info.health());
        if (external == NodeStatus.ABSENT) return NodeStatus.ABSENT;
        if (external == NodeStatus.DRIFTED) return NodeStatus.DRIFTED;
        if (external == NodeStatus.PRESENT && specHashStore.hasDrifted(node.id(), spec)) {
            return NodeStatus.DRIFTED;
        }
        return external;
    }

    private NodeStatus checkPool(DesiredNode node, ClaudonyPoolNodeSpec spec,
                                  Map<String, ClaudonyAppInfo> appsByName) {
        var appInfo = appsByName.get(spec.appName());
        if (appInfo == null) return NodeStatus.ABSENT;

        var actualPool = appInfo.pools().stream()
            .filter(p -> spec.agentId().equals(p.agentId()))
            .findFirst();
        if (actualPool.isEmpty()) return NodeStatus.ABSENT;

        var pool = actualPool.get();
        if (pool.minActive() != spec.minActive() || pool.maxActive() != spec.maxActive()) {
            return NodeStatus.DRIFTED;
        }

        if (specHashStore.hasDrifted(node.id(), spec)) {
            return NodeStatus.DRIFTED;
        }
        return NodeStatus.PRESENT;
    }
}
```

- [ ] **Step 4: Run tests — verify they pass**

Run: `mvn --batch-mode -o test -pl claudony -Dtest=ClaudonyActualStateAdapterTest`
Expected: All 7 tests pass

- [ ] **Step 5: Commit**

```bash
git add claudony/src/main/java/io/casehub/ops/claudony/ClaudonyActualStateAdapter.java claudony/src/test/java/io/casehub/ops/claudony/ClaudonyActualStateAdapterTest.java
git commit -m "feat(#140): ClaudonyActualStateAdapter — fleet state to drift detection"
```

### Task 4: ClaudonyNodeProvisioner

**Files:**
- Create: `claudony/src/main/java/io/casehub/ops/claudony/ClaudonyNodeProvisioner.java`
- Test: `claudony/src/test/java/io/casehub/ops/claudony/ClaudonyNodeProvisionerTest.java`

**Interfaces:**
- Consumes: `ClaudonyFleetClient`, `ClaudonySpecHashStore`, `ClaudonyAppSpec`, `ClaudonyAppInfo`, `ClaudonyApiException` from claudony module; `ClaudonyAppNodeSpec`, `ClaudonyPoolNodeSpec`, `ClaudonyNodeTypes` from Task 1
- Produces: `ClaudonyNodeProvisioner` implementing `NodeProvisioner` with `provision(DesiredNode, ProvisionContext)`, `deprovision(DesiredNode, DeprovisionContext)`, `suspend(DesiredNode, SuspendContext)`, `resume(DesiredNode, ResumeContext)`, `supportsStatefulLifecycle()` returning `true`

- [ ] **Step 1: Write failing tests**

```java
package io.casehub.ops.claudony;

import io.casehub.desiredstate.api.*;
import io.casehub.ops.api.claudony.*;
import io.casehub.platform.api.registry.HealthStatus;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.*;
import static org.assertj.core.api.Assertions.assertThat;

class ClaudonyNodeProvisionerTest {

    @Test
    void handledTypes_returns_app_and_pool() {
        var provisioner = new ClaudonyNodeProvisioner(
            new RecordingFleetClient(), new ClaudonySpecHashStore());
        assertThat(provisioner.handledTypes()).containsExactlyInAnyOrder(
            ClaudonyNodeTypes.APP, ClaudonyNodeTypes.POOL);
    }

    @Test
    void supportsStatefulLifecycle() {
        var provisioner = new ClaudonyNodeProvisioner(
            new RecordingFleetClient(), new ClaudonySpecHashStore());
        assertThat(provisioner.supportsStatefulLifecycle()).isTrue();
    }

    @Test
    void provision_app_calls_registerApp() {
        var client = new RecordingFleetClient();
        var provisioner = new ClaudonyNodeProvisioner(client, new ClaudonySpecHashStore());
        var spec = new ClaudonyAppNodeSpec("my-app", "My App", null, "img:1", Map.of("k", "v"));
        var node = new DesiredNode(NodeId.of(spec.nodeId()), spec, HumanGating.NONE);

        var result = provisioner.provision(node, provisionContext("ns1"));

        assertThat(result).isInstanceOf(ProvisionResult.Success.class);
        assertThat(client.registered).hasSize(1);
        assertThat(client.registered.getFirst().name()).isEqualTo("my-app");
        assertThat(client.registered.getFirst().namespace()).isEqualTo("ns1");
        assertThat(client.registered.getFirst().image()).isEqualTo("img:1");
    }

    @Test
    void provision_app_409_falls_back_to_update() {
        var client = new RecordingFleetClient();
        client.registerThrows409 = true;
        var provisioner = new ClaudonyNodeProvisioner(client, new ClaudonySpecHashStore());
        var spec = new ClaudonyAppNodeSpec("my-app", "My App", null, "img:1", Map.of());
        var node = new DesiredNode(NodeId.of(spec.nodeId()), spec, HumanGating.NONE);

        var result = provisioner.provision(node, provisionContext("ns1"));

        assertThat(result).isInstanceOf(ProvisionResult.Success.class);
        assertThat(client.updated).hasSize(1);
    }

    @Test
    void provision_pool_read_modify_write() {
        var client = new RecordingFleetClient();
        client.apps.put("my-app", appInfo("my-app", List.of()));
        var provisioner = new ClaudonyNodeProvisioner(client, new ClaudonySpecHashStore());
        var spec = new ClaudonyPoolNodeSpec("my-app", "agent-1", 1, 3);
        var node = new DesiredNode(NodeId.of(spec.nodeId()), spec, HumanGating.NONE);

        var result = provisioner.provision(node, provisionContext("ns1"));

        assertThat(result).isInstanceOf(ProvisionResult.Success.class);
        assertThat(client.updated).hasSize(1);
        assertThat(client.updated.getFirst().pools()).hasSize(1);
        assertThat(client.updated.getFirst().pools().getFirst().agentId()).isEqualTo("agent-1");
    }

    @Test
    void provision_pool_fails_when_parent_app_missing() {
        var client = new RecordingFleetClient();
        var provisioner = new ClaudonyNodeProvisioner(client, new ClaudonySpecHashStore());
        var spec = new ClaudonyPoolNodeSpec("missing-app", "agent-1", 1, 3);
        var node = new DesiredNode(NodeId.of(spec.nodeId()), spec, HumanGating.NONE);

        var result = provisioner.provision(node, provisionContext("ns1"));

        assertThat(result).isInstanceOf(ProvisionResult.Failed.class);
    }

    @Test
    void deprovision_app_calls_deregisterApp() {
        var client = new RecordingFleetClient();
        var provisioner = new ClaudonyNodeProvisioner(client, new ClaudonySpecHashStore());
        var spec = new ClaudonyAppNodeSpec("my-app", "My App", null, "img", Map.of());
        var node = new DesiredNode(NodeId.of(spec.nodeId()), spec, HumanGating.NONE);

        var result = provisioner.deprovision(node, deprovisionContext("ns1"));

        assertThat(result).isInstanceOf(DeprovisionResult.Success.class);
        assertThat(client.deregistered).containsExactly("my-app");
    }

    @Test
    void deprovision_pool_removes_from_app() {
        var client = new RecordingFleetClient();
        client.apps.put("my-app", appInfo("my-app",
            List.of(new ClaudonyAppInfo.AppPoolInfo("agent-1", 1, 3),
                     new ClaudonyAppInfo.AppPoolInfo("agent-2", 2, 4))));
        var provisioner = new ClaudonyNodeProvisioner(client, new ClaudonySpecHashStore());
        var spec = new ClaudonyPoolNodeSpec("my-app", "agent-1", 1, 3);
        var node = new DesiredNode(NodeId.of(spec.nodeId()), spec, HumanGating.NONE);

        var result = provisioner.deprovision(node, deprovisionContext("ns1"));

        assertThat(result).isInstanceOf(DeprovisionResult.Success.class);
        assertThat(client.updated).hasSize(1);
        assertThat(client.updated.getFirst().pools()).hasSize(1);
        assertThat(client.updated.getFirst().pools().getFirst().agentId()).isEqualTo("agent-2");
    }

    @Test
    void suspend_app_calls_suspendApp() {
        var client = new RecordingFleetClient();
        var provisioner = new ClaudonyNodeProvisioner(client, new ClaudonySpecHashStore());
        var spec = new ClaudonyAppNodeSpec("my-app", "App", null, "img", Map.of());
        var node = new DesiredNode(NodeId.of(spec.nodeId()), spec, HumanGating.NONE);

        var result = provisioner.suspend(node, suspendContext("ns1"));

        assertThat(result).isInstanceOf(SuspendResult.Success.class);
        assertThat(client.suspended).containsExactly("my-app");
    }

    @Test
    void resume_app_calls_resumeApp() {
        var client = new RecordingFleetClient();
        var provisioner = new ClaudonyNodeProvisioner(client, new ClaudonySpecHashStore());
        var spec = new ClaudonyAppNodeSpec("my-app", "App", null, "img", Map.of());
        var node = new DesiredNode(NodeId.of(spec.nodeId()), spec, HumanGating.NONE);

        var result = provisioner.resume(node, resumeContext("ns1"));

        assertThat(result).isInstanceOf(ResumeResult.Success.class);
        assertThat(client.resumed).containsExactly("my-app");
    }

    @Test
    void suspend_pool_returns_failed() {
        var client = new RecordingFleetClient();
        var provisioner = new ClaudonyNodeProvisioner(client, new ClaudonySpecHashStore());
        var spec = new ClaudonyPoolNodeSpec("my-app", "agent-1", 1, 3);
        var node = new DesiredNode(NodeId.of(spec.nodeId()), spec, HumanGating.NONE);

        var result = provisioner.suspend(node, suspendContext("ns1"));

        assertThat(result).isInstanceOf(SuspendResult.Failed.class);
    }

    // --- helpers ---

    private static ProvisionContext provisionContext(String tenancyId) {
        return new ProvisionContext(tenancyId, null);
    }

    private static DeprovisionContext deprovisionContext(String tenancyId) {
        return new DeprovisionContext(tenancyId, null);
    }

    private static SuspendContext suspendContext(String tenancyId) {
        return new SuspendContext(tenancyId);
    }

    private static ResumeContext resumeContext(String tenancyId) {
        return new ResumeContext(tenancyId);
    }

    private static ClaudonyAppInfo appInfo(String name, List<ClaudonyAppInfo.AppPoolInfo> pools) {
        return new ClaudonyAppInfo(name, "ns1", name, null, "img", Map.of(),
            HealthStatus.HEALTHY, Instant.now(), pools);
    }

    static class RecordingFleetClient implements ClaudonyFleetClient {
        final Map<String, ClaudonyAppInfo> apps = new HashMap<>();
        final List<ClaudonyAppSpec> registered = new ArrayList<>();
        final List<ClaudonyAppSpec> updated = new ArrayList<>();
        final List<String> deregistered = new ArrayList<>();
        final List<String> suspended = new ArrayList<>();
        final List<String> resumed = new ArrayList<>();
        boolean registerThrows409 = false;

        @Override
        public ClaudonyAppInfo registerApp(ClaudonyAppSpec spec) {
            if (registerThrows409) throw new ClaudonyApiException(409, "conflict");
            registered.add(spec);
            return new ClaudonyAppInfo(spec.name(), spec.namespace(), spec.displayName(),
                spec.description(), spec.image(), spec.config(),
                HealthStatus.HEALTHY, Instant.now(), spec.pools() != null ? spec.pools().stream()
                    .map(p -> new ClaudonyAppInfo.AppPoolInfo(p.agentId(), p.minActive(), p.maxActive()))
                    .toList() : List.of());
        }

        @Override
        public ClaudonyAppInfo updateApp(String name, ClaudonyAppSpec spec) {
            updated.add(spec);
            return new ClaudonyAppInfo(name, spec.namespace(), spec.displayName(),
                spec.description(), spec.image(), spec.config(),
                HealthStatus.HEALTHY, Instant.now(), spec.pools() != null ? spec.pools().stream()
                    .map(p -> new ClaudonyAppInfo.AppPoolInfo(p.agentId(), p.minActive(), p.maxActive()))
                    .toList() : List.of());
        }

        @Override public void deregisterApp(String name) { deregistered.add(name); }
        @Override public Optional<ClaudonyAppInfo> getApp(String name) { return Optional.ofNullable(apps.get(name)); }
        @Override public List<ClaudonyAppInfo> listApps(String ns) { return List.copyOf(apps.values()); }
        @Override public void suspendApp(String name) { suspended.add(name); }
        @Override public void resumeApp(String name) { resumed.add(name); }
        @Override public void heartbeat(String name) {}
    }
}
```

- [ ] **Step 2: Run tests — verify compilation failure**

Run: `mvn --batch-mode -o test -pl claudony -Dtest=ClaudonyNodeProvisionerTest`
Expected: Compilation failure — `ClaudonyNodeProvisioner` not found

- [ ] **Step 3: Implement ClaudonyNodeProvisioner**

```java
package io.casehub.ops.claudony;

import io.casehub.desiredstate.api.*;
import io.casehub.ops.api.claudony.*;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.ArrayList;
import java.util.List;
import java.util.Set;

@ApplicationScoped
public class ClaudonyNodeProvisioner implements NodeProvisioner {

    private static final Set<NodeType> HANDLED = Set.of(ClaudonyNodeTypes.APP, ClaudonyNodeTypes.POOL);

    private final ClaudonyFleetClient fleetClient;
    private final ClaudonySpecHashStore specHashStore;

    @Inject
    public ClaudonyNodeProvisioner(ClaudonyFleetClient fleetClient,
                                    ClaudonySpecHashStore specHashStore) {
        this.fleetClient = fleetClient;
        this.specHashStore = specHashStore;
    }

    @Override public Set<NodeType> handledTypes() { return HANDLED; }
    @Override public boolean supportsStatefulLifecycle() { return true; }

    @Override
    public ProvisionResult provision(DesiredNode node, ProvisionContext context) {
        try {
            return switch (node.spec()) {
                case ClaudonyAppNodeSpec spec -> provisionApp(node, spec, context);
                case ClaudonyPoolNodeSpec spec -> provisionPool(node, spec);
                default -> new ProvisionResult.Failed("Unknown spec type: " + node.spec().getClass());
            };
        } catch (Exception e) {
            return new ProvisionResult.Failed(e.getMessage());
        }
    }

    @Override
    public DeprovisionResult deprovision(DesiredNode node, DeprovisionContext context) {
        try {
            return switch (node.spec()) {
                case ClaudonyAppNodeSpec spec -> deprovisionApp(node, spec);
                case ClaudonyPoolNodeSpec spec -> deprovisionPool(node, spec);
                default -> new DeprovisionResult.Failed("Unknown spec type: " + node.spec().getClass());
            };
        } catch (Exception e) {
            return new DeprovisionResult.Failed(e.getMessage());
        }
    }

    @Override
    public SuspendResult suspend(DesiredNode node, SuspendContext context) {
        if (node.spec() instanceof ClaudonyAppNodeSpec spec) {
            try {
                fleetClient.suspendApp(spec.name());
                return new SuspendResult.Success();
            } catch (ClaudonyApiException e) {
                return new SuspendResult.Failed("Suspend failed: " + e.getMessage());
            }
        }
        return new SuspendResult.Failed("suspend not supported for " + node.type().value());
    }

    @Override
    public ResumeResult resume(DesiredNode node, ResumeContext context) {
        if (node.spec() instanceof ClaudonyAppNodeSpec spec) {
            try {
                fleetClient.resumeApp(spec.name());
                return new ResumeResult.Success();
            } catch (ClaudonyApiException e) {
                return new ResumeResult.Failed("Resume failed: " + e.getMessage());
            }
        }
        return new ResumeResult.Failed("resume not supported for " + node.type().value());
    }

    private ProvisionResult provisionApp(DesiredNode node, ClaudonyAppNodeSpec spec,
                                          ProvisionContext context) {
        var appSpec = new ClaudonyAppSpec(
            spec.name(), context.tenancyId(), spec.displayName(),
            spec.description(), spec.image(), spec.config(), null);
        try {
            fleetClient.registerApp(appSpec);
        } catch (ClaudonyApiException e) {
            if (e.statusCode() == 409) {
                fleetClient.updateApp(spec.name(), appSpec);
            } else {
                return new ProvisionResult.Failed("Register failed (" + e.statusCode() + "): " + e.responseBody());
            }
        }
        specHashStore.record(node.id(), spec);
        return new ProvisionResult.Success();
    }

    private ProvisionResult provisionPool(DesiredNode node, ClaudonyPoolNodeSpec spec) {
        var appInfo = fleetClient.getApp(spec.appName());
        if (appInfo.isEmpty()) {
            return new ProvisionResult.Failed("parent app not registered: " + spec.appName());
        }
        var info = appInfo.get();
        var pools = new ArrayList<>(info.pools().stream()
            .filter(p -> !spec.agentId().equals(p.agentId()))
            .map(p -> new ClaudonyAppSpec.PoolSpec(p.agentId(), p.minActive(), p.maxActive()))
            .toList());
        pools.add(new ClaudonyAppSpec.PoolSpec(spec.agentId(), spec.minActive(), spec.maxActive()));

        var updated = new ClaudonyAppSpec(
            info.name(), info.namespace(), info.displayName(),
            info.description(), info.image(), info.config(), pools);
        fleetClient.updateApp(spec.appName(), updated);
        specHashStore.record(node.id(), spec);
        return new ProvisionResult.Success();
    }

    private DeprovisionResult deprovisionApp(DesiredNode node, ClaudonyAppNodeSpec spec) {
        try {
            fleetClient.deregisterApp(spec.name());
        } catch (ClaudonyApiException e) {
            if (e.statusCode() != 404) {
                return new DeprovisionResult.Failed("Deregister failed: " + e.getMessage());
            }
        }
        specHashStore.remove(node.id());
        return new DeprovisionResult.Success();
    }

    private DeprovisionResult deprovisionPool(DesiredNode node, ClaudonyPoolNodeSpec spec) {
        var appInfo = fleetClient.getApp(spec.appName());
        if (appInfo.isEmpty()) {
            specHashStore.remove(node.id());
            return new DeprovisionResult.Success();
        }
        var info = appInfo.get();
        var pools = info.pools().stream()
            .filter(p -> !spec.agentId().equals(p.agentId()))
            .map(p -> new ClaudonyAppSpec.PoolSpec(p.agentId(), p.minActive(), p.maxActive()))
            .toList();

        var updated = new ClaudonyAppSpec(
            info.name(), info.namespace(), info.displayName(),
            info.description(), info.image(), info.config(), pools);
        fleetClient.updateApp(spec.appName(), updated);
        specHashStore.remove(node.id());
        return new DeprovisionResult.Success();
    }
}
```

- [ ] **Step 4: Run tests — verify they pass**

Run: `mvn --batch-mode -o test -pl claudony -Dtest=ClaudonyNodeProvisionerTest`
Expected: All 11 tests pass

- [ ] **Step 5: Run all claudony tests to verify no regressions**

Run: `mvn --batch-mode -o test -pl claudony`
Expected: All tests pass

- [ ] **Step 6: Commit**

```bash
git add claudony/src/main/java/io/casehub/ops/claudony/ClaudonyNodeProvisioner.java claudony/src/test/java/io/casehub/ops/claudony/ClaudonyNodeProvisionerTest.java
git commit -m "feat(#140): ClaudonyNodeProvisioner — provision/deprovision/suspend/resume via fleet API"
```

---

## Batch 4: Supporting SPIs — FaultPolicy, EventSource, NodeSpecFactoryProvider

### Task 5: FaultPolicy, EventSource, and NodeSpecFactoryProvider

**Files:**
- Create: `claudony/src/main/java/io/casehub/ops/claudony/ClaudonyFaultPolicy.java`
- Create: `claudony/src/main/java/io/casehub/ops/claudony/ClaudonyEventSource.java`
- Create: `claudony/src/main/java/io/casehub/ops/claudony/ClaudonyNodeSpecFactoryProvider.java`
- Test: `claudony/src/test/java/io/casehub/ops/claudony/ClaudonyFaultPolicyTest.java`
- Test: `claudony/src/test/java/io/casehub/ops/claudony/ClaudonyEventSourceTest.java`
- Test: `claudony/src/test/java/io/casehub/ops/claudony/ClaudonyNodeSpecFactoryProviderTest.java`

**Interfaces:**
- Consumes: `ClaudonyNodeTypes`, `ClaudonyReviewSpec`, `ClaudonyNodeSpec`, `ClaudonyAppNodeSpec`, `ClaudonyPoolNodeSpec` from Task 1; `ThresholdFaultPolicy`, `FaultPolicy`, `EventSource`, `NodeSpecFactoryProvider` from `casehub-desiredstate-api`
- Produces: `ClaudonyFaultPolicy` implementing `FaultPolicy`, `ClaudonyEventSource` implementing `EventSource` with `emit(StateEvent)` and `emitDrift(NodeId)`, `ClaudonyNodeSpecFactoryProvider` implementing `NodeSpecFactoryProvider`

- [ ] **Step 1: Write failing tests for FaultPolicy**

```java
package io.casehub.ops.claudony;

import io.casehub.desiredstate.api.*;
import io.casehub.ops.api.claudony.*;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import static org.assertj.core.api.Assertions.assertThat;

class ClaudonyFaultPolicyTest {

    private final ClaudonyFaultPolicy policy = new ClaudonyFaultPolicy();

    @Test
    void belowThreshold_noMutations() {
        var graph = graphWith(new ClaudonyAppNodeSpec("app", "App", null, "img", Map.of()));
        var event = new FaultEvent(NodeId.of("claudony_app:app"), FaultType.PROVISION_FAILED,
            "connection refused", 1);
        var actual = new ActualState(Map.of(NodeId.of("claudony_app:app"), NodeStatus.ABSENT));

        var mutations = policy.onFault("ns1", event, graph, actual);
        assertThat(mutations).isEmpty();
    }

    @Test
    void atThreshold_addsReviewNode() {
        var graph = graphWith(new ClaudonyAppNodeSpec("app", "App", null, "img", Map.of()));
        var event = new FaultEvent(NodeId.of("claudony_app:app"), FaultType.PROVISION_FAILED,
            "connection refused", 3);
        var actual = new ActualState(Map.of(NodeId.of("claudony_app:app"), NodeStatus.ABSENT));

        var mutations = policy.onFault("ns1", event, graph, actual);
        assertThat(mutations).isNotEmpty();
    }

    @Test
    void reviewNodeFault_ignored() {
        var reviewSpec = new ClaudonyReviewSpec(NodeId.of("claudony_app:app"), "failed");
        var graph = graphWith(reviewSpec);
        var event = new FaultEvent(NodeId.of("claudony_review:app"), FaultType.PROVISION_FAILED,
            "review failed", 3);
        var actual = new ActualState(Map.of());

        var mutations = policy.onFault("ns1", event, graph, actual);
        assertThat(mutations).isEmpty();
    }

    private DesiredStateGraph graphWith(NodeSpec spec) {
        return new DefaultDesiredStateGraphFactory().of(
            List.of(new DesiredNode(NodeId.of("test"), spec, HumanGating.NONE)), List.of());
    }
}
```

- [ ] **Step 2: Write failing tests for EventSource**

```java
package io.casehub.ops.claudony;

import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.api.NodeStatus;
import io.casehub.desiredstate.api.StateEvent;
import org.junit.jupiter.api.Test;
import java.time.Duration;
import java.util.ArrayList;
import static org.assertj.core.api.Assertions.assertThat;

class ClaudonyEventSourceTest {

    @Test
    void emit_delivers_to_subscriber() {
        var source = new ClaudonyEventSource();
        var received = new ArrayList<StateEvent>();
        source.stream().subscribe().with(received::add);

        var event = new StateEvent(NodeId.of("claudony_app:x"), NodeStatus.DRIFTED, "drift");
        source.emit(event);

        assertThat(received).containsExactly(event);
    }

    @Test
    void emitDrift_delivers_drift_event() {
        var source = new ClaudonyEventSource();
        var received = new ArrayList<StateEvent>();
        source.stream().subscribe().with(received::add);

        source.emitDrift(NodeId.of("claudony_app:x"));

        assertThat(received).hasSize(1);
        assertThat(received.getFirst().nodeId()).isEqualTo(NodeId.of("claudony_app:x"));
        assertThat(received.getFirst().status()).isEqualTo(NodeStatus.DRIFTED);
    }
}
```

- [ ] **Step 3: Write failing tests for NodeSpecFactoryProvider**

```java
package io.casehub.ops.claudony;

import io.casehub.ops.api.claudony.ClaudonyAppNodeSpec;
import io.casehub.ops.api.claudony.ClaudonyPoolNodeSpec;
import org.junit.jupiter.api.Test;
import java.util.Map;
import static org.assertj.core.api.Assertions.assertThat;

class ClaudonyNodeSpecFactoryProviderTest {

    private final ClaudonyNodeSpecFactoryProvider provider = new ClaudonyNodeSpecFactoryProvider();

    @Test
    void provides_factories_for_app_and_pool() {
        var factories = provider.provide();
        assertThat(factories).containsKeys("claudony_app", "claudony_pool");
    }

    @Test
    void app_factory_creates_ClaudonyAppNodeSpec() {
        var factories = provider.provide();
        var spec = factories.get("claudony_app").create(Map.of(
            "name", "my-app", "image", "img:1", "config", Map.of()));
        assertThat(spec).isInstanceOf(ClaudonyAppNodeSpec.class);
        assertThat(((ClaudonyAppNodeSpec) spec).name()).isEqualTo("my-app");
    }

    @Test
    void pool_factory_creates_ClaudonyPoolNodeSpec() {
        var factories = provider.provide();
        var spec = factories.get("claudony_pool").create(Map.of(
            "appName", "my-app", "agentId", "agent-1", "minActive", 1, "maxActive", 3));
        assertThat(spec).isInstanceOf(ClaudonyPoolNodeSpec.class);
        assertThat(((ClaudonyPoolNodeSpec) spec).agentId()).isEqualTo("agent-1");
    }
}
```

- [ ] **Step 4: Run tests — verify compilation failure**

Run: `mvn --batch-mode -o test -pl claudony -Dtest="ClaudonyFaultPolicyTest,ClaudonyEventSourceTest,ClaudonyNodeSpecFactoryProviderTest"`
Expected: Compilation failure

- [ ] **Step 5: Implement all three classes**

Create `ClaudonyFaultPolicy.java`:
```java
package io.casehub.ops.claudony;

import io.casehub.desiredstate.api.*;
import io.casehub.ops.api.claudony.ClaudonyNodeTypes;
import io.casehub.ops.api.claudony.ClaudonyReviewSpec;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.List;
import java.util.Set;

@ApplicationScoped
public class ClaudonyFaultPolicy implements FaultPolicy {

    private static final NodeType CLAUDONY_REVIEW = NodeType.of("claudony_review");

    private final ThresholdFaultPolicy delegate = ThresholdFaultPolicy.builder()
        .faultTypes(Set.of(FaultType.PROVISION_FAILED))
        .nodeTypes(Set.of(ClaudonyNodeTypes.APP, ClaudonyNodeTypes.POOL))
        .ignoreTypes(Set.of(CLAUDONY_REVIEW))
        .tier(3, FaultPolicy.addReviewNode(
            (event, current) -> new ClaudonyReviewSpec(event.node(), event.detail())))
        .build();

    @Override
    public List<GraphMutation<DesiredNode>> onFault(String tenancyId, FaultEvent event,
                                                     DesiredStateGraph current, ActualState actual) {
        return delegate.onFault(tenancyId, event, current, actual);
    }
}
```

Create `ClaudonyEventSource.java`:
```java
package io.casehub.ops.claudony;

import io.casehub.desiredstate.api.EventSource;
import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.api.NodeStatus;
import io.casehub.desiredstate.api.StateEvent;
import io.smallrye.mutiny.Multi;
import io.smallrye.mutiny.subscription.BackPressureStrategy;
import io.smallrye.mutiny.subscription.MultiEmitter;
import jakarta.enterprise.context.ApplicationScoped;

@ApplicationScoped
public class ClaudonyEventSource implements EventSource {

    private volatile MultiEmitter<? super StateEvent> emitter;
    private final Multi<StateEvent> stream;

    public ClaudonyEventSource() {
        this.stream = Multi.createFrom().<StateEvent>emitter(
            e -> this.emitter = e, BackPressureStrategy.BUFFER)
            .broadcast().toAllSubscribers();
    }

    @Override
    public Multi<StateEvent> stream() { return stream; }

    public void emit(StateEvent event) {
        var e = this.emitter;
        if (e != null) e.emit(event);
    }

    public void emitDrift(NodeId nodeId) {
        emit(new StateEvent(nodeId, NodeStatus.DRIFTED, "drift detected"));
    }
}
```

Create `ClaudonyNodeSpecFactoryProvider.java`:
```java
package io.casehub.ops.claudony;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.desiredstate.api.NodeSpecFactory;
import io.casehub.desiredstate.api.NodeSpecFactoryProvider;
import io.casehub.desiredstate.api.annotations.NodeTypeId;
import io.casehub.ops.api.claudony.ClaudonyNodeSpec;
import java.util.HashMap;
import java.util.Map;

public class ClaudonyNodeSpecFactoryProvider implements NodeSpecFactoryProvider {

    private static final ObjectMapper MAPPER = new ObjectMapper();

    @Override
    public Map<String, NodeSpecFactory> provide() {
        var factories = new HashMap<String, NodeSpecFactory>();
        for (var subclass : ClaudonyNodeSpec.class.getPermittedSubclasses()) {
            var annotation = subclass.getAnnotation(NodeTypeId.class);
            if (annotation != null) {
                var specClass = subclass;
                factories.put(annotation.value(),
                    specMap -> MAPPER.convertValue(specMap, specClass));
            }
        }
        return factories;
    }
}
```

- [ ] **Step 6: Run tests — verify they pass**

Run: `mvn --batch-mode -o test -pl claudony -Dtest="ClaudonyFaultPolicyTest,ClaudonyEventSourceTest,ClaudonyNodeSpecFactoryProviderTest"`
Expected: All tests pass

- [ ] **Step 7: Run all claudony tests**

Run: `mvn --batch-mode -o test -pl claudony`
Expected: All tests pass

- [ ] **Step 8: Commit**

```bash
git add claudony/src/main/java/io/casehub/ops/claudony/ClaudonyFaultPolicy.java claudony/src/main/java/io/casehub/ops/claudony/ClaudonyEventSource.java claudony/src/main/java/io/casehub/ops/claudony/ClaudonyNodeSpecFactoryProvider.java claudony/src/test/java/io/casehub/ops/claudony/ClaudonyFaultPolicyTest.java claudony/src/test/java/io/casehub/ops/claudony/ClaudonyEventSourceTest.java claudony/src/test/java/io/casehub/ops/claudony/ClaudonyNodeSpecFactoryProviderTest.java
git commit -m "feat(#140): ClaudonyFaultPolicy, EventSource, and NodeSpecFactoryProvider"
```

---

## Batch 5: Integration — GoalLoader, DomainRegistration, reconciliation test

### Task 6: ClaudonyGoalLoader and DomainRegistration

**Files:**
- Create: `claudony/src/main/java/io/casehub/ops/claudony/ClaudonyGoalLoader.java`
- Modify: `service/src/main/java/io/casehub/ops/service/bootstrap/BootstrapDomainRegistrar.java` — add `registerClaudonyDomain()`
- Test: `claudony/src/test/java/io/casehub/ops/claudony/ClaudonyGoalLoaderTest.java`
- Create: `claudony/src/test/resources/claudony-fleet-test.yaml` (test fixture)

**Interfaces:**
- Consumes: `ClaudonyGoals`, `ClaudonyGoalEntry`, `ClaudonyAppNodeSpec`, `ClaudonyPoolNodeSpec` from Task 1; `CrossDomainCompositionEngine`, `DomainRegistration`, `DomainId` from `casehub-desiredstate-api`
- Produces: `ClaudonyGoalLoader` with `load(String path)` returning `ClaudonyGoals` and `loadDirectory(String directoryPath)` returning `ClaudonyGoals`

- [ ] **Step 1: Create test fixture YAML**

Create `claudony/src/test/resources/claudony-fleet-test.yaml`:
```yaml
apps:
  - spec:
      name: "test-app"
      displayName: "Test App"
      image: "test:1.0"
      config:
        key1: "value1"
    dependsOn: []
pools:
  - spec:
      appName: "test-app"
      agentId: "agent-1"
      minActive: 1
      maxActive: 3
    dependsOn: []
```

- [ ] **Step 2: Write failing tests**

```java
package io.casehub.ops.claudony;

import io.casehub.ops.api.claudony.ClaudonyAppNodeSpec;
import io.casehub.ops.api.claudony.ClaudonyPoolNodeSpec;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class ClaudonyGoalLoaderTest {

    private final ClaudonyGoalLoader loader = new ClaudonyGoalLoader();

    @Test
    void loads_goals_from_classpath() {
        var goals = loader.load("claudony-fleet-test.yaml");

        assertThat(goals.apps()).hasSize(1);
        assertThat(goals.apps().getFirst().spec()).isInstanceOf(ClaudonyAppNodeSpec.class);
        assertThat(goals.apps().getFirst().spec().name()).isEqualTo("test-app");
        assertThat(goals.apps().getFirst().spec().image()).isEqualTo("test:1.0");

        assertThat(goals.pools()).hasSize(1);
        assertThat(goals.pools().getFirst().spec()).isInstanceOf(ClaudonyPoolNodeSpec.class);
        assertThat(goals.pools().getFirst().spec().appName()).isEqualTo("test-app");
    }

    @Test
    void empty_file_returns_empty_goals() {
        var goals = loader.load("nonexistent.yaml");
        assertThat(goals.apps()).isEmpty();
        assertThat(goals.pools()).isEmpty();
    }
}
```

- [ ] **Step 3: Implement ClaudonyGoalLoader**

```java
package io.casehub.ops.claudony;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import io.casehub.ops.api.claudony.ClaudonyGoals;
import jakarta.enterprise.context.ApplicationScoped;
import java.io.*;
import java.nio.file.*;
import java.util.List;
import java.util.stream.Stream;

@ApplicationScoped
public class ClaudonyGoalLoader {

    private final ObjectMapper yamlMapper = new ObjectMapper(new YAMLFactory())
        .registerModule(new JavaTimeModule());

    public ClaudonyGoals load(String path) {
        try (var stream = resolveStream(path)) {
            if (stream == null) return new ClaudonyGoals(List.of(), List.of());
            return yamlMapper.readValue(stream, ClaudonyGoals.class);
        } catch (IOException e) {
            throw new UncheckedIOException(e);
        }
    }

    public ClaudonyGoals loadDirectory(String directoryPath) {
        var dir = Path.of(directoryPath);
        if (!Files.isDirectory(dir)) return new ClaudonyGoals(List.of(), List.of());

        try (Stream<Path> files = Files.list(dir)) {
            var yamlFiles = files
                .filter(p -> p.toString().endsWith(".yaml") || p.toString().endsWith(".yml"))
                .sorted()
                .toList();

            var allApps = new java.util.ArrayList<io.casehub.ops.api.claudony.ClaudonyGoalEntry<io.casehub.ops.api.claudony.ClaudonyAppNodeSpec>>();
            var allPools = new java.util.ArrayList<io.casehub.ops.api.claudony.ClaudonyGoalEntry<io.casehub.ops.api.claudony.ClaudonyPoolNodeSpec>>();

            for (var file : yamlFiles) {
                var goals = yamlMapper.readValue(file.toFile(), ClaudonyGoals.class);
                allApps.addAll(goals.apps());
                allPools.addAll(goals.pools());
            }
            return new ClaudonyGoals(allApps, allPools);
        } catch (IOException e) {
            throw new UncheckedIOException(e);
        }
    }

    private InputStream resolveStream(String path) {
        var classpath = getClass().getClassLoader().getResourceAsStream(path);
        if (classpath != null) return classpath;
        var file = Path.of(path);
        if (Files.exists(file)) {
            try { return Files.newInputStream(file); }
            catch (IOException e) { throw new UncheckedIOException(e); }
        }
        return null;
    }
}
```

- [ ] **Step 4: Update BootstrapDomainRegistrar**

Add a `registerClaudonyDomain()` method following the existing pattern. Use `ide_find_class` to locate `BootstrapDomainRegistrar`, then `ide_insert_member` to add the method. Also update `onStartup()` to call it.

```java
private void registerClaudonyDomain() {
    var emptyGraph = new DefaultDesiredStateGraphFactory().of(List.of(), List.of());
    engine.registerDomain(DomainRegistration.builder(
        DomainId.of("claudony"), CompilationResult.single(emptyGraph))
        .provides(Set.of(ClaudonyNodeTypes.APP, ClaudonyNodeTypes.POOL))
        .build());
}
```

Add import: `import io.casehub.ops.api.claudony.ClaudonyNodeTypes;`

Add call in `onStartup()`: `registerClaudonyDomain();`

- [ ] **Step 5: Run tests — verify they pass**

Run: `mvn --batch-mode -o test -pl claudony -Dtest=ClaudonyGoalLoaderTest`
Expected: All tests pass

- [ ] **Step 6: Run full build to verify service module compiles**

Run: `mvn --batch-mode install -DskipTests`
Expected: All modules compile successfully

- [ ] **Step 7: Commit**

```bash
git add claudony/src/main/java/io/casehub/ops/claudony/ClaudonyGoalLoader.java claudony/src/test/java/io/casehub/ops/claudony/ClaudonyGoalLoaderTest.java claudony/src/test/resources/claudony-fleet-test.yaml service/src/main/java/io/casehub/ops/service/bootstrap/BootstrapDomainRegistrar.java
git commit -m "feat(#140): ClaudonyGoalLoader and domain registration in BootstrapDomainRegistrar"
```

### Task 7: Full reconciliation integration test

**Files:**
- Create: `claudony/src/test/java/io/casehub/ops/claudony/ClaudonyReconciliationIntegrationTest.java`

**Interfaces:**
- Consumes: All SPI implementations from Tasks 2-6, `DefaultDesiredStateGraphFactory` and `TransitionPlanner` from `casehub-desiredstate` runtime (test scope)

- [ ] **Step 1: Write integration test with SimulatingFleetClient**

```java
package io.casehub.ops.claudony;

import io.casehub.desiredstate.api.*;
import io.casehub.ops.api.claudony.*;
import io.casehub.platform.api.registry.HealthStatus;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import static org.assertj.core.api.Assertions.assertThat;

class ClaudonyReconciliationIntegrationTest {

    private SimulatingFleetClient client;
    private ClaudonySpecHashStore specHashStore;
    private ClaudonyGoalCompiler compiler;
    private ClaudonyActualStateAdapter adapter;
    private ClaudonyNodeProvisioner provisioner;
    private DesiredStateGraphFactory graphFactory;

    @BeforeEach
    void setUp() {
        client = new SimulatingFleetClient();
        specHashStore = new ClaudonySpecHashStore();
        compiler = new ClaudonyGoalCompiler();
        adapter = new ClaudonyActualStateAdapter(client, specHashStore);
        provisioner = new ClaudonyNodeProvisioner(client, specHashStore);
        graphFactory = new DefaultDesiredStateGraphFactory();
    }

    @Test
    void greenField_compileProvisionConverge() {
        var goals = new ClaudonyGoals(
            List.of(new ClaudonyGoalEntry<>(
                new ClaudonyAppNodeSpec("app-a", "App A", null, "img:1", Map.of()), List.of())),
            List.of(new ClaudonyGoalEntry<>(
                new ClaudonyPoolNodeSpec("app-a", "agent-1", 1, 3), List.of())));

        var result = compiler.compile(goals, graphFactory);
        var graph = result.graphs().getFirst();

        // First read — all absent
        var actual1 = adapter.readActual(graph, "ns1");
        assertThat(actual1.statuses().values()).allMatch(s -> s == NodeStatus.ABSENT);

        // Provision app first (dependency ordering)
        var appNode = graph.nodes().get(NodeId.of("claudony_app:app-a"));
        var appResult = provisioner.provision(appNode, new ProvisionContext("ns1", null));
        assertThat(appResult).isInstanceOf(ProvisionResult.Success.class);

        // Provision pool
        var poolNode = graph.nodes().get(NodeId.of("claudony_pool:app-a:agent-1"));
        var poolResult = provisioner.provision(poolNode, new ProvisionContext("ns1", null));
        assertThat(poolResult).isInstanceOf(ProvisionResult.Success.class);

        // Second read — all present
        var actual2 = adapter.readActual(graph, "ns1");
        assertThat(actual2.statuses().values()).allMatch(s -> s == NodeStatus.PRESENT);
    }

    @Test
    void driftDetection_poolConfigChanged_reconciles() {
        // Set up initial state
        client.apps.put("app-a", new SimApp("app-a", "ns1", "img:1", HealthStatus.HEALTHY,
            List.of(new ClaudonyAppSpec.PoolSpec("agent-1", 2, 5))));

        var poolSpec = new ClaudonyPoolNodeSpec("app-a", "agent-1", 1, 3);
        specHashStore.record(NodeId.of(poolSpec.nodeId()), poolSpec);

        var graph = graphFactory.of(
            List.of(new DesiredNode(NodeId.of(poolSpec.nodeId()), poolSpec, HumanGating.NONE)),
            List.of());

        // Read — drifted (minActive 2 vs desired 1)
        var actual = adapter.readActual(graph, "ns1");
        assertThat(actual.statuses().get(NodeId.of("claudony_pool:app-a:agent-1")))
            .isEqualTo(NodeStatus.DRIFTED);

        // Re-provision
        var poolNode = graph.nodes().get(NodeId.of("claudony_pool:app-a:agent-1"));
        var result = provisioner.provision(poolNode, new ProvisionContext("ns1", null));
        assertThat(result).isInstanceOf(ProvisionResult.Success.class);

        // Verify pool updated
        var updatedApp = client.apps.get("app-a");
        assertThat(updatedApp.pools).hasSize(1);
        assertThat(updatedApp.pools.getFirst().minActive()).isEqualTo(1);
    }

    @Test
    void suspendAndResume_lifecycle() {
        client.apps.put("app-a", new SimApp("app-a", "ns1", "img:1", HealthStatus.HEALTHY, List.of()));
        var spec = new ClaudonyAppNodeSpec("app-a", "App", null, "img:1", Map.of());
        var node = new DesiredNode(NodeId.of(spec.nodeId()), spec, HumanGating.NONE);

        // Suspend
        var suspendResult = provisioner.suspend(node, new SuspendContext("ns1"));
        assertThat(suspendResult).isInstanceOf(SuspendResult.Success.class);
        assertThat(client.apps.get("app-a").health).isEqualTo(HealthStatus.SUSPENDED);

        // Read actual — suspended
        var graph = graphFactory.of(List.of(node), List.of());
        var actual = adapter.readActual(graph, "ns1");
        assertThat(actual.statuses().get(NodeId.of("claudony_app:app-a")))
            .isEqualTo(NodeStatus.SUSPENDED);

        // Resume
        var resumeResult = provisioner.resume(node, new ResumeContext("ns1"));
        assertThat(resumeResult).isInstanceOf(ResumeResult.Success.class);
        assertThat(client.apps.get("app-a").health).isEqualTo(HealthStatus.HEALTHY);
    }

    @Test
    void deprovision_removesAppAndPools() {
        client.apps.put("app-a", new SimApp("app-a", "ns1", "img:1", HealthStatus.HEALTHY,
            List.of(new ClaudonyAppSpec.PoolSpec("agent-1", 1, 3))));

        var appSpec = new ClaudonyAppNodeSpec("app-a", "App", null, "img:1", Map.of());
        var appNode = new DesiredNode(NodeId.of(appSpec.nodeId()), appSpec, HumanGating.NONE);

        var result = provisioner.deprovision(appNode, new DeprovisionContext("ns1", null));
        assertThat(result).isInstanceOf(DeprovisionResult.Success.class);
        assertThat(client.apps).doesNotContainKey("app-a");
    }

    // --- Simulating fleet client ---

    static class SimApp {
        String name, namespace, image;
        HealthStatus health;
        List<ClaudonyAppSpec.PoolSpec> pools;
        SimApp(String name, String ns, String img, HealthStatus h, List<ClaudonyAppSpec.PoolSpec> pools) {
            this.name = name; this.namespace = ns; this.image = img;
            this.health = h; this.pools = new ArrayList<>(pools);
        }
    }

    static class SimulatingFleetClient implements ClaudonyFleetClient {
        final ConcurrentHashMap<String, SimApp> apps = new ConcurrentHashMap<>();

        @Override
        public ClaudonyAppInfo registerApp(ClaudonyAppSpec spec) {
            if (apps.containsKey(spec.name())) throw new ClaudonyApiException(409, "exists");
            var app = new SimApp(spec.name(), spec.namespace(), spec.image(), HealthStatus.HEALTHY,
                spec.pools() != null ? spec.pools() : List.of());
            apps.put(spec.name(), app);
            return toInfo(app);
        }

        @Override
        public ClaudonyAppInfo updateApp(String name, ClaudonyAppSpec spec) {
            var app = apps.get(name);
            if (app == null) throw new ClaudonyApiException(404, "not found");
            app.image = spec.image();
            app.pools = spec.pools() != null ? new ArrayList<>(spec.pools()) : new ArrayList<>();
            return toInfo(app);
        }

        @Override
        public void deregisterApp(String name) {
            if (apps.remove(name) == null) throw new ClaudonyApiException(404, "not found");
        }

        @Override
        public Optional<ClaudonyAppInfo> getApp(String name) {
            var app = apps.get(name);
            return app != null ? Optional.of(toInfo(app)) : Optional.empty();
        }

        @Override
        public List<ClaudonyAppInfo> listApps(String namespace) {
            return apps.values().stream()
                .filter(a -> namespace == null || namespace.equals(a.namespace))
                .map(this::toInfo).toList();
        }

        @Override
        public void suspendApp(String name) {
            var app = apps.get(name);
            if (app == null) throw new ClaudonyApiException(404, "not found");
            app.health = HealthStatus.SUSPENDED;
        }

        @Override
        public void resumeApp(String name) {
            var app = apps.get(name);
            if (app == null) throw new ClaudonyApiException(404, "not found");
            app.health = HealthStatus.HEALTHY;
        }

        @Override public void heartbeat(String name) {}

        private ClaudonyAppInfo toInfo(SimApp app) {
            return new ClaudonyAppInfo(app.name, app.namespace, app.name, null,
                app.image, Map.of(), app.health, Instant.now(),
                app.pools.stream()
                    .map(p -> new ClaudonyAppInfo.AppPoolInfo(p.agentId(), p.minActive(), p.maxActive()))
                    .toList());
        }
    }
}
```

- [ ] **Step 2: Run integration test**

Run: `mvn --batch-mode -o test -pl claudony -Dtest=ClaudonyReconciliationIntegrationTest`
Expected: All 4 integration tests pass

- [ ] **Step 3: Run full module test suite**

Run: `mvn --batch-mode -o test -pl claudony`
Expected: All tests pass (unit + integration)

- [ ] **Step 4: Commit**

```bash
git add claudony/src/test/java/io/casehub/ops/claudony/ClaudonyReconciliationIntegrationTest.java
git commit -m "feat(#140): claudony reconciliation integration test — full loop verification"
```

---

## Batch 6: Documentation — ADR, ARC42STORIES, consumer guide, issues

### Task 8: ADR-0003, ARC42STORIES update, consumer guide, and issue filing

**Files:**
- Create: `docs/adr/0003-claudony-desiredstate-domain.md`
- Modify: `docs/adr/INDEX.md` — add ADR-0003 entry
- Modify: `ARC42STORIES.MD` — update §4 and §5
- Modify: `docs/guides/consumer-guide.md` — add claudony fleet section

**Interfaces:**
- Consumes: All previous tasks — the documentation describes what was built
- Produces: ADR-0003, updated architecture record, consumer-facing documentation

- [ ] **Step 1: Create ADR-0003**

Create `docs/adr/0003-claudony-desiredstate-domain.md`:
```markdown
# 0003 — Claudony standalone desiredstate domain for fleet management

Date: 2026-10-18
Status: Accepted

## Context and Problem Statement

Issue #140 requires declarative fleet management for claudony — describing what apps and pools should exist and having the reconciliation loop enforce that state. ADR-0002 established `ClaudonyDeploymentTarget implements ApplicationDeploymentTarget` for deploying individual CaseHub applications to claudony. Fleet management is a different concern requiring its own GoalCompiler, ActualStateAdapter, and NodeProvisioner.

## Decision Drivers

* Fleet management needs its own YAML declaration format (`claudony-fleet.yaml`) distinct from deployment topology
* Node types `claudony_app` and `claudony_pool` are disjoint from deployment's `casehub_application` and `pool` — no `handledTypes()` collision
* Pools have different provisioning semantics from apps — sealed hierarchy required
* ADR-0002's `ApplicationDeploymentTarget` pattern remains valid for ad-hoc app deployment

## Decision Outcome

Claudony becomes a standalone desiredstate domain with full SPI Quad alongside the existing `ClaudonyDeploymentTarget`. The two patterns are complementary:

| Concern | Pattern | Node types |
|---------|---------|-----------|
| Deploy a CaseHub app to claudony | ADR-0002: `ApplicationDeploymentTarget` | `casehub_application` |
| Manage claudony fleet declaratively | ADR-0003: standalone SPI Quad | `claudony_app`, `claudony_pool` |

### Positive Consequences

* Declarative fleet management with reconciliation loop enforcement
* Full SPI Quad including fault escalation and event-driven drift detection
* Clean domain boundary — claudony fleet is self-contained

### Negative Consequences

* Dual paths to `ClaudonyFleetClient` — operational documentation must prevent declaring the same app in both patterns
* Pool provisioning uses read-modify-write until claudony adds standalone pool CRUD endpoints

## Relationship to ADR-0002

ADR-0002 remains accepted and valid. This ADR adds a complementary pattern for a different use case, not a replacement.

## Links

* [ops#140](https://github.com/casehubio/casehub-ops/issues/140) — Claudony NodeProvisioner
* [ADR-0002](0002-claudony-integration-via-deployment-target.md) — Complementary pattern
```

- [ ] **Step 2: Update ADR INDEX.md**

Add entry for ADR-0003 following the existing format.

- [ ] **Step 3: Update ARC42STORIES.MD §4 and §5**

In §4 (Solution Strategy), add claudony to the type strategy table:
```markdown
| Sealed hierarchy | Variants have different fields and provisioning semantics | `ClaudonyNodeSpec` → `ClaudonyAppNodeSpec` (image, config) vs `ClaudonyPoolNodeSpec` (agentId, minActive, maxActive) |
```

In §5 (Building Block View), add the claudony domain:
```markdown
Container_Boundary(claudony, "Layer: L7 Claudony") {
    Container(claudony_mod, "claudony module", "CDI beans", "ClaudonyGoalCompiler, ClaudonyNodeProvisioner, fleet client")
}
```

Update the module structure table to include the claudony domain role.

- [ ] **Step 4: Update consumer guide with claudony fleet section**

Add a section documenting:
- Claudony fleet YAML format (`claudony-fleet.yaml`)
- The boundary with deployment module (don't declare same app in both)
- Node types (`claudony_app`, `claudony_pool`)
- Pool-to-app dependency (automatic)

- [ ] **Step 5: File claudony issues**

Using `gh issue create` on the claudony repo:
1. Standalone pool CRUD endpoints
2. ETag/version support on app specs

File ops issues:
1. Backfill `@DesiredStateQualifier` on existing GoalCompilers
2. Promote `SpecHashStore` to shared utility

- [ ] **Step 6: Run full build**

Run: `mvn --batch-mode install`
Expected: All modules build and all tests pass

- [ ] **Step 7: Commit**

```bash
git add docs/adr/0003-claudony-desiredstate-domain.md docs/adr/INDEX.md ARC42STORIES.MD docs/guides/consumer-guide.md
git commit -m "feat(#140): ADR-0003, ARC42STORIES update, and consumer guide for claudony domain"
```

---

## References

- `specs/issue-140-claudony-nodeprovisioner/2026-10-18-claudony-desiredstate-spi-design.md` — design spec this plan implements
- `container/src/main/java/io/casehub/ops/container/ContainerGoalCompiler.java` — GoalCompiler pattern
- `container/src/main/java/io/casehub/ops/container/PodmanActualStateAdapter.java` — ActualStateAdapter pattern
- `container/src/main/java/io/casehub/ops/container/PodmanNodeProvisioner.java` — NodeProvisioner pattern
- `container/src/main/java/io/casehub/ops/container/ContainerFaultPolicy.java` — FaultPolicy pattern
- `container/src/main/java/io/casehub/ops/container/ContainerEventSource.java` — EventSource pattern
- `container/src/main/java/io/casehub/ops/container/ContainerNodeSpecFactoryProvider.java` — factory provider pattern
- `deployment/src/main/java/io/casehub/ops/deployment/SpecHashStore.java` — spec hash drift detection
- `service/src/main/java/io/casehub/ops/service/bootstrap/BootstrapDomainRegistrar.java` — domain registration
- `claudony/src/main/java/io/casehub/ops/claudony/ClaudonyFleetClient.java` — existing fleet client
- `claudony/src/test/java/io/casehub/ops/claudony/ClaudonyDeploymentTargetTest.java` — stub fleet client pattern
- `docs/adr/0002-claudony-integration-via-deployment-target.md` — companion ADR
- GitHub #140 — focal issue
