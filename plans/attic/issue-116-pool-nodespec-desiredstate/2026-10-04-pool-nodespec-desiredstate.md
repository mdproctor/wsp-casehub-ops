# PoolNodeSpec + Pool Lifecycle Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #116 — feat: PoolNodeSpec + PoolProvisionHandler — pool lifecycle via desiredstate reconciliation
**Issue group:** #116

**Goal:** Add pool as the 7th node type in the deployment domain, with provision/deprovision handlers, drift detection, and SPI quad wiring.

**Architecture:** Follow the existing AgentNodeSpec + AgentProvisionHandler pattern exactly. New types in `api/`, handler and drift checker in `deployment/`, wiring changes to the existing SPI quad (sealed permits, goals record, compiler, adapter, provisioner). Pool operations cross service boundary via `PoolOperations` interface with mcpDomain implementation deferred until claudony endpoints exist.

**Tech Stack:** Java 21+, Quarkus, JUnit 5, AssertJ

## Global Constraints

- Each module is a standalone Jandex library — no config required for activation
- `DeploymentNodeSpec` is a sealed interface — compiler enforces exhaustive switch
- Tests use stubs/fakes, not mocks
- All records use `@JsonIgnoreProperties(ignoreUnknown = true)` for forward compatibility
- tenancyId propagated through all calls — bind in repository/adapter layer only

---

## Batch 1: API types — PoolNodeSpec and supporting enums

### Task 1: PoolNodeSpec record, PoolScalingSpec, WorkingDirPolicy, EvictionStrategy, and DeploymentNodeSpec/DeploymentGoals wiring

**Files:**
- Create: `api/src/main/java/io/casehub/ops/api/deployment/PoolScalingSpec.java`
- Create: `api/src/main/java/io/casehub/ops/api/deployment/WorkingDirPolicy.java`
- Create: `api/src/main/java/io/casehub/ops/api/deployment/EvictionStrategy.java`
- Create: `api/src/main/java/io/casehub/ops/api/deployment/PoolNodeSpec.java`
- Modify: `api/src/main/java/io/casehub/ops/api/deployment/DeploymentNodeSpec.java:7` — add `PoolNodeSpec` to sealed permits
- Modify: `api/src/main/java/io/casehub/ops/api/deployment/DeploymentGoals.java:7-25` — add `pools` field
- Create: `api/src/test/java/io/casehub/ops/api/deployment/PoolNodeSpecTest.java`
- Test: `api/src/test/java/io/casehub/ops/api/deployment/PoolNodeSpecTest.java`

**Interfaces:**
- Produces: `PoolNodeSpec(String agentId, String backend, int minActive, int maxActive, String workingDir, WorkingDirPolicy workingDirPolicy, PoolScalingSpec scaling, EvictionStrategy eviction)` — used by Tasks 2, 3, 4
- Produces: `PoolScalingSpec(String type, Double targetFillRatio, String cooldown, String scaleInCooldown)`
- Produces: `DeploymentGoals.pools()` returning `List<GoalEntry<PoolNodeSpec>>` — used by Task 4

- [ ] **Step 1: Write PoolNodeSpec tests**

```java
package io.casehub.ops.api.deployment;

import io.casehub.desiredstate.api.NodeType;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class PoolNodeSpecTest {

    @Test
    void nodeIdAppendsDashPool() {
        var spec = testPool("code-reviewer");
        assertThat(spec.nodeId()).isEqualTo("code-reviewer-pool");
    }

    @Test
    void nodeTypeIsPool() {
        var spec = testPool("code-reviewer");
        assertThat(spec.nodeType()).isEqualTo(NodeType.of("pool"));
    }

    @Test
    void agentIdRequired() {
        assertThatThrownBy(() -> new PoolNodeSpec(
                null, "claudony", 2, 8, null, null, null, null))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("agentId");
    }

    @Test
    void blankAgentIdRejected() {
        assertThatThrownBy(() -> new PoolNodeSpec(
                "  ", "claudony", 2, 8, null, null, null, null))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("agentId");
    }

    @Test
    void backendRequired() {
        assertThatThrownBy(() -> new PoolNodeSpec(
                "agent-1", null, 2, 8, null, null, null, null))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("backend");
    }

    @Test
    void maxActiveMustBeAtLeastOne() {
        assertThatThrownBy(() -> new PoolNodeSpec(
                "agent-1", "claudony", 0, 0, null, null, null, null))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("maxActive must be >= 1");
    }

    @Test
    void maxActiveMustBeGreaterOrEqualMinActive() {
        assertThatThrownBy(() -> new PoolNodeSpec(
                "agent-1", "claudony", 5, 3, null, null, null, null))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("maxActive must be >= minActive");
    }

    @Test
    void evictionDefaultsToMemoryWeighted() {
        var spec = new PoolNodeSpec("agent-1", "claudony", 2, 8, null, null, null, null);
        assertThat(spec.eviction()).isEqualTo(EvictionStrategy.MEMORY_WEIGHTED);
    }

    @Test
    void scalingSpecDefaultsTypeToNone() {
        var scaling = new PoolScalingSpec(null, null, null, null);
        assertThat(scaling.type()).isEqualTo("none");
    }

    @Test
    void scalingSpecPreservesExplicitType() {
        var scaling = new PoolScalingSpec("target-tracking", 0.7, "30s", null);
        assertThat(scaling.type()).isEqualTo("target-tracking");
        assertThat(scaling.targetFillRatio()).isEqualTo(0.7);
        assertThat(scaling.cooldown()).isEqualTo("30s");
    }

    @Test
    void fullConstruction() {
        var scaling = new PoolScalingSpec("target-tracking", 0.7, "30s", "60s");
        var spec = new PoolNodeSpec(
                "code-reviewer", "claudony", 2, 8,
                "~/workspace/reviews", WorkingDirPolicy.SHARED_READ,
                scaling, EvictionStrategy.LRU);

        assertThat(spec.agentId()).isEqualTo("code-reviewer");
        assertThat(spec.backend()).isEqualTo("claudony");
        assertThat(spec.minActive()).isEqualTo(2);
        assertThat(spec.maxActive()).isEqualTo(8);
        assertThat(spec.workingDir()).isEqualTo("~/workspace/reviews");
        assertThat(spec.workingDirPolicy()).isEqualTo(WorkingDirPolicy.SHARED_READ);
        assertThat(spec.scaling().type()).isEqualTo("target-tracking");
        assertThat(spec.eviction()).isEqualTo(EvictionStrategy.LRU);
    }

    static PoolNodeSpec testPool(String agentId) {
        return new PoolNodeSpec(agentId, "claudony", 2, 8,
                "~/workspace", WorkingDirPolicy.SHARED_READ,
                new PoolScalingSpec("target-tracking", 0.7, "30s", null),
                EvictionStrategy.MEMORY_WEIGHTED);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn --batch-mode -o test -pl api -Dtest=PoolNodeSpecTest`
Expected: FAIL — classes do not exist yet

- [ ] **Step 3: Create EvictionStrategy enum**

```java
package io.casehub.ops.api.deployment;

public enum EvictionStrategy {
    MEMORY_WEIGHTED,
    LRU
}
```

- [ ] **Step 4: Create WorkingDirPolicy enum**

```java
package io.casehub.ops.api.deployment;

public enum WorkingDirPolicy {
    EXCLUSIVE,
    SHARED_READ,
    BRANCH_ISOLATED
}
```

- [ ] **Step 5: Create PoolScalingSpec record**

```java
package io.casehub.ops.api.deployment;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;

@JsonIgnoreProperties(ignoreUnknown = true)
public record PoolScalingSpec(
        String type,
        Double targetFillRatio,
        String cooldown,
        String scaleInCooldown
) {
    public PoolScalingSpec {
        if (type == null || type.isBlank()) type = "none";
    }
}
```

- [ ] **Step 6: Create PoolNodeSpec record**

```java
package io.casehub.ops.api.deployment;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
import io.casehub.desiredstate.api.NodeType;

@JsonIgnoreProperties(ignoreUnknown = true)
public record PoolNodeSpec(
        String agentId,
        String backend,
        int minActive,
        int maxActive,
        String workingDir,
        WorkingDirPolicy workingDirPolicy,
        PoolScalingSpec scaling,
        EvictionStrategy eviction
) implements DeploymentNodeSpec {

    public PoolNodeSpec {
        if (agentId == null || agentId.isBlank())
            throw new IllegalArgumentException("agentId is required");
        if (backend == null || backend.isBlank())
            throw new IllegalArgumentException("backend is required");
        if (maxActive < 1)
            throw new IllegalArgumentException("maxActive must be >= 1");
        if (maxActive < minActive)
            throw new IllegalArgumentException("maxActive must be >= minActive");
        if (eviction == null) eviction = EvictionStrategy.MEMORY_WEIGHTED;
    }

    @Override
    public String nodeId() {
        return agentId + "-pool";
    }

    @Override
    public NodeType nodeType() {
        return NodeType.of("pool");
    }
}
```

- [ ] **Step 7: Add PoolNodeSpec to DeploymentNodeSpec sealed permits**

In `DeploymentNodeSpec.java`, change the `permits` clause from:

```java
public sealed interface DeploymentNodeSpec extends NodeSpec permits
                                                            AgentNodeSpec, ChannelNodeSpec, CaseTypeNodeSpec, TrustPolicyNodeSpec, EndpointNodeSpec, DetectionNodeSpec {
```

to:

```java
public sealed interface DeploymentNodeSpec extends NodeSpec permits
                                                            AgentNodeSpec, ChannelNodeSpec, CaseTypeNodeSpec, TrustPolicyNodeSpec, EndpointNodeSpec, DetectionNodeSpec, PoolNodeSpec {
```

- [ ] **Step 8: Add pools field to DeploymentGoals**

In `DeploymentGoals.java`, add `List<GoalEntry<PoolNodeSpec>> pools` as the 7th field (before `adaptations`):

```java
@JsonIgnoreProperties(ignoreUnknown = true)
public record DeploymentGoals(
        List<GoalEntry<AgentNodeSpec>> agents,
        List<GoalEntry<ChannelNodeSpec>> channels,
        List<GoalEntry<CaseTypeNodeSpec>> caseTypes,
        List<GoalEntry<TrustPolicyNodeSpec>> trust,
        List<GoalEntry<EndpointNodeSpec>> endpoints,
        List<GoalEntry<DetectionNodeSpec>> detections,
        List<GoalEntry<PoolNodeSpec>> pools,
        List<AdaptationRuleSpec> adaptations
) {
    public DeploymentGoals {
        agents = agents != null ? List.copyOf(agents) : List.of();
        channels = channels != null ? List.copyOf(channels) : List.of();
        caseTypes = caseTypes != null ? List.copyOf(caseTypes) : List.of();
        trust = trust != null ? List.copyOf(trust) : List.of();
        endpoints = endpoints != null ? List.copyOf(endpoints) : List.of();
        detections = detections != null ? List.copyOf(detections) : List.of();
        pools = pools != null ? List.copyOf(pools) : List.of();
        adaptations = adaptations != null ? List.copyOf(adaptations) : List.of();
    }
}
```

**Important:** Adding a field to the `DeploymentGoals` record changes its canonical constructor. All existing call sites that construct `DeploymentGoals` must be updated to pass the new `pools` parameter. Search with `ide_find_references` for all usages of the `DeploymentGoals` constructor. Each existing call site needs `List.of()` inserted as the 7th argument.

- [ ] **Step 9: Fix all DeploymentGoals constructor call sites**

Use `ide_find_references` on `DeploymentGoals` to find every constructor call. For each one, insert `List.of()` as the 7th positional argument (after `detections`, before `adaptations`). This includes:
- `DeploymentGoalCompilerTest.java` — every `new DeploymentGoals(...)` call
- Any YAML/JSON parser that constructs `DeploymentGoals`
- Any test fixture builders

- [ ] **Step 10: Run tests to verify they pass**

Run: `mvn --batch-mode -o test -pl api`
Expected: PASS — all existing api tests plus new PoolNodeSpecTest

- [ ] **Step 11: Compile deployment module to check sealed switch exhaustiveness**

Run: `mvn --batch-mode -o compile -pl deployment`
Expected: FAIL — `DeploymentNodeProvisioner.doProvision()` and `doDeprovision()` switch expressions are now non-exhaustive (missing `case PoolNodeSpec`). This is expected — Task 3 will fix it.

- [ ] **Step 12: Commit**

```bash
git add api/src/main/java/io/casehub/ops/api/deployment/EvictionStrategy.java
git add api/src/main/java/io/casehub/ops/api/deployment/WorkingDirPolicy.java
git add api/src/main/java/io/casehub/ops/api/deployment/PoolScalingSpec.java
git add api/src/main/java/io/casehub/ops/api/deployment/PoolNodeSpec.java
git add api/src/main/java/io/casehub/ops/api/deployment/DeploymentNodeSpec.java
git add api/src/main/java/io/casehub/ops/api/deployment/DeploymentGoals.java
git add api/src/test/java/io/casehub/ops/api/deployment/PoolNodeSpecTest.java
git commit -m "feat(#116): PoolNodeSpec record + supporting enums + DeploymentGoals wiring"
```

---

## Batch 2: Deployment module — handler, drift checker, SPI quad wiring

### Task 2: PoolProvisionHandler with PoolOperations interface

**Files:**
- Create: `deployment/src/main/java/io/casehub/ops/deployment/handler/PoolProvisionHandler.java`
- Create: `deployment/src/test/java/io/casehub/ops/deployment/handler/PoolProvisionHandlerTest.java`

**Interfaces:**
- Consumes: `PoolNodeSpec` from Task 1
- Produces: `PoolProvisionHandler.provision(PoolNodeSpec, ProvisionContext)` → `ProvisionResult`
- Produces: `PoolProvisionHandler.deprovision(PoolNodeSpec, DeprovisionContext)` → `DeprovisionResult`
- Produces: `PoolProvisionHandler.PoolOperations` interface — used by Task 3 (drift checker) and Task 4 (wiring)
- Produces: `PoolProvisionHandler.PoolInfo` record — used by Task 3
- Produces: `PoolProvisionHandler.PoolCreateRequest` record
- Produces: `PoolProvisionHandler.PoolUpdateRequest` record

- [ ] **Step 1: Write PoolProvisionHandler tests**

```java
package io.casehub.ops.deployment.handler;

import io.casehub.desiredstate.api.DeprovisionContext;
import io.casehub.desiredstate.api.DeprovisionResult;
import io.casehub.desiredstate.api.ProvisionContext;
import io.casehub.desiredstate.api.ProvisionResult;
import io.casehub.desiredstate.runtime.DefaultDesiredStateGraphFactory;
import io.casehub.ops.api.deployment.EvictionStrategy;
import io.casehub.ops.api.deployment.PoolNodeSpec;
import io.casehub.ops.api.deployment.PoolScalingSpec;
import io.casehub.ops.api.deployment.WorkingDirPolicy;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;

import static org.assertj.core.api.Assertions.assertThat;

class PoolProvisionHandlerTest {

    private StubPoolOperations ops;
    private PoolProvisionHandler handler;

    @BeforeEach
    void setUp() {
        ops = new StubPoolOperations();
        handler = new PoolProvisionHandler(ops);
    }

    @Test
    void provisionCreatesPoolWhenAbsent() {
        var spec = testPool("code-reviewer");
        var context = new ProvisionContext("tenant-1",
                new DefaultDesiredStateGraphFactory().empty());

        var result = handler.provision(spec, context);

        assertThat(result).isInstanceOf(ProvisionResult.Success.class);
        assertThat(ops.pools).containsKey("code-reviewer");
        var created = ops.pools.get("code-reviewer");
        assertThat(created.minActive()).isEqualTo(2);
        assertThat(created.maxActive()).isEqualTo(8);
        assertThat(created.scalingType()).isEqualTo("target-tracking");
        assertThat(created.eviction()).isEqualTo("MEMORY_WEIGHTED");
    }

    @Test
    void provisionUpdatesPoolWhenPresent() {
        ops.pools.put("code-reviewer", new PoolProvisionHandler.PoolInfo(
                "code-reviewer", "ACTIVE", 1, 4,
                "none", null, null, null, "LRU"));

        var spec = testPool("code-reviewer");
        var context = new ProvisionContext("tenant-1",
                new DefaultDesiredStateGraphFactory().empty());

        var result = handler.provision(spec, context);

        assertThat(result).isInstanceOf(ProvisionResult.Success.class);
        var updated = ops.pools.get("code-reviewer");
        assertThat(updated.minActive()).isEqualTo(2);
        assertThat(updated.maxActive()).isEqualTo(8);
    }

    @Test
    void provisionSkipsUpdateWhenConfigMatches() {
        ops.pools.put("code-reviewer", new PoolProvisionHandler.PoolInfo(
                "code-reviewer", "ACTIVE", 2, 8,
                "target-tracking", 0.7, "30s", null, "MEMORY_WEIGHTED"));

        var spec = testPool("code-reviewer");
        var context = new ProvisionContext("tenant-1",
                new DefaultDesiredStateGraphFactory().empty());

        var result = handler.provision(spec, context);

        assertThat(result).isInstanceOf(ProvisionResult.Success.class);
        assertThat(ops.updateCount).isZero();
    }

    @Test
    void deprovisionDestroysPoolWhenPresent() {
        ops.pools.put("code-reviewer", new PoolProvisionHandler.PoolInfo(
                "code-reviewer", "ACTIVE", 2, 8,
                "target-tracking", 0.7, "30s", null, "MEMORY_WEIGHTED"));

        var spec = testPool("code-reviewer");
        var context = new DeprovisionContext("tenant-1",
                new DefaultDesiredStateGraphFactory().empty());

        var result = handler.deprovision(spec, context);

        assertThat(result).isInstanceOf(DeprovisionResult.Success.class);
        assertThat(ops.pools).doesNotContainKey("code-reviewer");
    }

    @Test
    void deprovisionSucceedsWhenPoolAlreadyAbsent() {
        var spec = testPool("code-reviewer");
        var context = new DeprovisionContext("tenant-1",
                new DefaultDesiredStateGraphFactory().empty());

        var result = handler.deprovision(spec, context);

        assertThat(result).isInstanceOf(DeprovisionResult.Success.class);
    }

    static PoolNodeSpec testPool(String agentId) {
        return new PoolNodeSpec(agentId, "claudony", 2, 8,
                "~/workspace/reviews", WorkingDirPolicy.SHARED_READ,
                new PoolScalingSpec("target-tracking", 0.7, "30s", null),
                EvictionStrategy.MEMORY_WEIGHTED);
    }

    static class StubPoolOperations implements PoolProvisionHandler.PoolOperations {
        final ConcurrentHashMap<String, PoolProvisionHandler.PoolInfo> pools = new ConcurrentHashMap<>();
        int updateCount = 0;

        @Override
        public Optional<PoolProvisionHandler.PoolInfo> getPool(String name) {
            return Optional.ofNullable(pools.get(name));
        }

        @Override
        public void createPool(PoolProvisionHandler.PoolCreateRequest request) {
            pools.put(request.agentId(), new PoolProvisionHandler.PoolInfo(
                    request.agentId(), "ACTIVE",
                    request.minActive(), request.maxActive(),
                    request.scalingType(), request.targetFillRatio(),
                    request.cooldown(), request.scaleInCooldown(),
                    request.eviction()));
        }

        @Override
        public void updatePool(String name, PoolProvisionHandler.PoolUpdateRequest request) {
            updateCount++;
            var existing = pools.get(name);
            pools.put(name, new PoolProvisionHandler.PoolInfo(
                    name, existing.status(),
                    request.minActive() != null ? request.minActive() : existing.minActive(),
                    request.maxActive() != null ? request.maxActive() : existing.maxActive(),
                    request.scalingType() != null ? request.scalingType() : existing.scalingType(),
                    request.targetFillRatio() != null ? request.targetFillRatio() : existing.targetFillRatio(),
                    request.cooldown() != null ? request.cooldown() : existing.cooldown(),
                    request.scaleInCooldown() != null ? request.scaleInCooldown() : existing.scaleInCooldown(),
                    request.eviction() != null ? request.eviction() : existing.eviction()));
        }

        @Override
        public void destroyPool(String name) {
            pools.remove(name);
        }
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn --batch-mode -o test -pl deployment -Dtest=PoolProvisionHandlerTest`
Expected: FAIL — `PoolProvisionHandler` does not exist

- [ ] **Step 3: Implement PoolProvisionHandler**

```java
package io.casehub.ops.deployment.handler;

import io.casehub.desiredstate.api.DeprovisionContext;
import io.casehub.desiredstate.api.DeprovisionResult;
import io.casehub.desiredstate.api.ProvisionContext;
import io.casehub.desiredstate.api.ProvisionResult;
import io.casehub.ops.api.deployment.PoolNodeSpec;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.util.Objects;
import java.util.Optional;

@ApplicationScoped
public class PoolProvisionHandler {

    public interface PoolOperations {
        Optional<PoolInfo> getPool(String name);
        void createPool(PoolCreateRequest request);
        void updatePool(String name, PoolUpdateRequest request);
        void destroyPool(String name);
    }

    public record PoolInfo(
            String name, String status,
            int minActive, int maxActive,
            String scalingType, Double targetFillRatio,
            String cooldown, String scaleInCooldown,
            String eviction
    ) {}

    public record PoolCreateRequest(
            String agentId, String backend,
            int minActive, int maxActive,
            String workingDir, String workingDirPolicy,
            String scalingType, Double targetFillRatio,
            String cooldown, String scaleInCooldown,
            String eviction
    ) {}

    public record PoolUpdateRequest(
            Integer minActive, Integer maxActive,
            String scalingType, Double targetFillRatio,
            String cooldown, String scaleInCooldown,
            String eviction
    ) {}

    private final PoolOperations ops;

    @Inject
    public PoolProvisionHandler(PoolOperations ops) {
        this.ops = ops;
    }

    public ProvisionResult provision(PoolNodeSpec spec, ProvisionContext context) {
        var existing = ops.getPool(spec.agentId());
        if (existing.isEmpty()) {
            ops.createPool(toCreateRequest(spec));
            return new ProvisionResult.Success();
        }
        if (configDiffers(spec, existing.get())) {
            ops.updatePool(spec.agentId(), toUpdateRequest(spec));
        }
        return new ProvisionResult.Success();
    }

    public DeprovisionResult deprovision(PoolNodeSpec spec, DeprovisionContext context) {
        var existing = ops.getPool(spec.agentId());
        if (existing.isPresent()) {
            ops.destroyPool(spec.agentId());
        }
        return new DeprovisionResult.Success();
    }

    private boolean configDiffers(PoolNodeSpec spec, PoolInfo actual) {
        if (spec.minActive() != actual.minActive()) return true;
        if (spec.maxActive() != actual.maxActive()) return true;
        var scalingType = spec.scaling() != null ? spec.scaling().type() : "none";
        if (!Objects.equals(scalingType, actual.scalingType())) return true;
        var targetFillRatio = spec.scaling() != null ? spec.scaling().targetFillRatio() : null;
        if (!Objects.equals(targetFillRatio, actual.targetFillRatio())) return true;
        var eviction = spec.eviction() != null ? spec.eviction().name() : "MEMORY_WEIGHTED";
        return !Objects.equals(eviction, actual.eviction());
    }

    private PoolCreateRequest toCreateRequest(PoolNodeSpec spec) {
        return new PoolCreateRequest(
                spec.agentId(), spec.backend(),
                spec.minActive(), spec.maxActive(),
                spec.workingDir(),
                spec.workingDirPolicy() != null ? spec.workingDirPolicy().name() : null,
                spec.scaling() != null ? spec.scaling().type() : "none",
                spec.scaling() != null ? spec.scaling().targetFillRatio() : null,
                spec.scaling() != null ? spec.scaling().cooldown() : null,
                spec.scaling() != null ? spec.scaling().scaleInCooldown() : null,
                spec.eviction() != null ? spec.eviction().name() : "MEMORY_WEIGHTED");
    }

    private PoolUpdateRequest toUpdateRequest(PoolNodeSpec spec) {
        return new PoolUpdateRequest(
                spec.minActive(), spec.maxActive(),
                spec.scaling() != null ? spec.scaling().type() : "none",
                spec.scaling() != null ? spec.scaling().targetFillRatio() : null,
                spec.scaling() != null ? spec.scaling().cooldown() : null,
                spec.scaling() != null ? spec.scaling().scaleInCooldown() : null,
                spec.eviction() != null ? spec.eviction().name() : "MEMORY_WEIGHTED");
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn --batch-mode -o test -pl deployment -Dtest=PoolProvisionHandlerTest`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add deployment/src/main/java/io/casehub/ops/deployment/handler/PoolProvisionHandler.java
git add deployment/src/test/java/io/casehub/ops/deployment/handler/PoolProvisionHandlerTest.java
git commit -m "feat(#116): PoolProvisionHandler with PoolOperations interface"
```

### Task 3: PoolDriftChecker

**Files:**
- Create: `deployment/src/main/java/io/casehub/ops/deployment/drift/PoolDriftChecker.java`
- Create: `deployment/src/test/java/io/casehub/ops/deployment/drift/PoolDriftCheckerTest.java`

**Interfaces:**
- Consumes: `PoolNodeSpec` from Task 1
- Consumes: `PoolProvisionHandler.PoolOperations`, `PoolProvisionHandler.PoolInfo` from Task 2
- Produces: `PoolDriftChecker implements NodeDriftChecker` — used by Task 4 (wiring)

- [ ] **Step 1: Write PoolDriftChecker tests**

```java
package io.casehub.ops.deployment.drift;

import io.casehub.desiredstate.api.NodeStatus;
import io.casehub.ops.api.deployment.EvictionStrategy;
import io.casehub.ops.api.deployment.PoolNodeSpec;
import io.casehub.ops.api.deployment.PoolScalingSpec;
import io.casehub.ops.api.deployment.WorkingDirPolicy;
import io.casehub.ops.deployment.handler.PoolProvisionHandler;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;

import static org.junit.jupiter.api.Assertions.assertEquals;

class PoolDriftCheckerTest {

    private PoolDriftChecker checker;
    private StubPoolOperations ops;
    private static final String TENANCY_ID = "tenant-1";

    @BeforeEach
    void setUp() {
        ops = new StubPoolOperations();
        checker = new PoolDriftChecker(ops);
    }

    @Test
    void nodeType() {
        assertEquals("pool", checker.nodeType());
    }

    @Test
    void poolPresent() {
        ops.pools.put("code-reviewer", new PoolProvisionHandler.PoolInfo(
                "code-reviewer", "ACTIVE", 2, 8,
                "target-tracking", 0.7, "30s", null, "MEMORY_WEIGHTED"));

        var spec = testPool("code-reviewer");
        assertEquals(NodeStatus.PRESENT, checker.check(spec, TENANCY_ID));
    }

    @Test
    void poolAbsent() {
        var spec = testPool("code-reviewer");
        assertEquals(NodeStatus.ABSENT, checker.check(spec, TENANCY_ID));
    }

    @Test
    void poolDrifted_capacityMismatch() {
        ops.pools.put("code-reviewer", new PoolProvisionHandler.PoolInfo(
                "code-reviewer", "ACTIVE", 1, 4,
                "target-tracking", 0.7, "30s", null, "MEMORY_WEIGHTED"));

        var spec = testPool("code-reviewer");
        assertEquals(NodeStatus.DRIFTED, checker.check(spec, TENANCY_ID));
    }

    @Test
    void poolDrifted_scalingTypeMismatch() {
        ops.pools.put("code-reviewer", new PoolProvisionHandler.PoolInfo(
                "code-reviewer", "ACTIVE", 2, 8,
                "none", null, null, null, "MEMORY_WEIGHTED"));

        var spec = testPool("code-reviewer");
        assertEquals(NodeStatus.DRIFTED, checker.check(spec, TENANCY_ID));
    }

    @Test
    void poolDrifted_targetFillRatioMismatch() {
        ops.pools.put("code-reviewer", new PoolProvisionHandler.PoolInfo(
                "code-reviewer", "ACTIVE", 2, 8,
                "target-tracking", 0.5, "30s", null, "MEMORY_WEIGHTED"));

        var spec = testPool("code-reviewer");
        assertEquals(NodeStatus.DRIFTED, checker.check(spec, TENANCY_ID));
    }

    @Test
    void poolDrifted_evictionMismatch() {
        ops.pools.put("code-reviewer", new PoolProvisionHandler.PoolInfo(
                "code-reviewer", "ACTIVE", 2, 8,
                "target-tracking", 0.7, "30s", null, "LRU"));

        var spec = testPool("code-reviewer");
        assertEquals(NodeStatus.DRIFTED, checker.check(spec, TENANCY_ID));
    }

    @Test
    void unknownSpecType() {
        var spec = new io.casehub.ops.api.deployment.ChannelNodeSpec(
                "ch1", "desc", io.casehub.qhorus.api.channel.ChannelSemantic.APPEND,
                null, null, null, null, null, null, null,
                null, null, null, null);

        assertEquals(NodeStatus.UNKNOWN, checker.check(spec, TENANCY_ID));
    }

    static PoolNodeSpec testPool(String agentId) {
        return new PoolNodeSpec(agentId, "claudony", 2, 8,
                "~/workspace/reviews", WorkingDirPolicy.SHARED_READ,
                new PoolScalingSpec("target-tracking", 0.7, "30s", null),
                EvictionStrategy.MEMORY_WEIGHTED);
    }

    static class StubPoolOperations implements PoolProvisionHandler.PoolOperations {
        final ConcurrentHashMap<String, PoolProvisionHandler.PoolInfo> pools = new ConcurrentHashMap<>();

        @Override
        public Optional<PoolProvisionHandler.PoolInfo> getPool(String name) {
            return Optional.ofNullable(pools.get(name));
        }

        @Override
        public void createPool(PoolProvisionHandler.PoolCreateRequest request) {
            pools.put(request.agentId(), new PoolProvisionHandler.PoolInfo(
                    request.agentId(), "ACTIVE",
                    request.minActive(), request.maxActive(),
                    request.scalingType(), request.targetFillRatio(),
                    request.cooldown(), request.scaleInCooldown(),
                    request.eviction()));
        }

        @Override
        public void updatePool(String name, PoolProvisionHandler.PoolUpdateRequest request) {}

        @Override
        public void destroyPool(String name) {
            pools.remove(name);
        }
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn --batch-mode -o test -pl deployment -Dtest=PoolDriftCheckerTest`
Expected: FAIL — `PoolDriftChecker` does not exist

- [ ] **Step 3: Implement PoolDriftChecker**

```java
package io.casehub.ops.deployment.drift;

import io.casehub.desiredstate.api.NodeSpec;
import io.casehub.desiredstate.api.NodeStatus;
import io.casehub.ops.api.deployment.NodeDriftChecker;
import io.casehub.ops.api.deployment.PoolNodeSpec;
import io.casehub.ops.deployment.handler.PoolProvisionHandler;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

import java.util.Objects;

@ApplicationScoped
public class PoolDriftChecker implements NodeDriftChecker {

    private static final Logger LOG = Logger.getLogger(PoolDriftChecker.class);

    private final PoolProvisionHandler.PoolOperations ops;

    @Inject
    public PoolDriftChecker(PoolProvisionHandler.PoolOperations ops) {
        this.ops = ops;
    }

    @Override
    public String nodeType() {
        return "pool";
    }

    @Override
    public NodeStatus check(NodeSpec spec, String tenancyId) {
        if (!(spec instanceof PoolNodeSpec poolSpec)) {
            return NodeStatus.UNKNOWN;
        }

        var actual = ops.getPool(poolSpec.agentId());
        if (actual.isEmpty()) {
            return NodeStatus.ABSENT;
        }

        var pool = actual.get();
        if (capacityDrifted(poolSpec, pool)
                || scalingDrifted(poolSpec, pool)
                || evictionDrifted(poolSpec, pool)) {
            return NodeStatus.DRIFTED;
        }
        return NodeStatus.PRESENT;
    }

    private boolean capacityDrifted(PoolNodeSpec spec, PoolProvisionHandler.PoolInfo actual) {
        if (spec.minActive() != actual.minActive()) {
            LOG.debugf("pool %s: minActive drifted [%d → %d]",
                    spec.agentId(), spec.minActive(), actual.minActive());
            return true;
        }
        if (spec.maxActive() != actual.maxActive()) {
            LOG.debugf("pool %s: maxActive drifted [%d → %d]",
                    spec.agentId(), spec.maxActive(), actual.maxActive());
            return true;
        }
        return false;
    }

    private boolean scalingDrifted(PoolNodeSpec spec, PoolProvisionHandler.PoolInfo actual) {
        var desiredType = spec.scaling() != null ? spec.scaling().type() : "none";
        if (!Objects.equals(desiredType, actual.scalingType())) {
            LOG.debugf("pool %s: scalingType drifted [%s → %s]",
                    spec.agentId(), desiredType, actual.scalingType());
            return true;
        }
        var desiredRatio = spec.scaling() != null ? spec.scaling().targetFillRatio() : null;
        if (!Objects.equals(desiredRatio, actual.targetFillRatio())) {
            LOG.debugf("pool %s: targetFillRatio drifted [%s → %s]",
                    spec.agentId(), desiredRatio, actual.targetFillRatio());
            return true;
        }
        return false;
    }

    private boolean evictionDrifted(PoolNodeSpec spec, PoolProvisionHandler.PoolInfo actual) {
        var desiredEviction = spec.eviction() != null ? spec.eviction().name() : "MEMORY_WEIGHTED";
        if (!Objects.equals(desiredEviction, actual.eviction())) {
            LOG.debugf("pool %s: eviction drifted [%s → %s]",
                    spec.agentId(), desiredEviction, actual.eviction());
            return true;
        }
        return false;
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn --batch-mode -o test -pl deployment -Dtest=PoolDriftCheckerTest`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add deployment/src/main/java/io/casehub/ops/deployment/drift/PoolDriftChecker.java
git add deployment/src/test/java/io/casehub/ops/deployment/drift/PoolDriftCheckerTest.java
git commit -m "feat(#116): PoolDriftChecker — full config drift detection"
```

### Task 4: SPI quad wiring — GoalCompiler, ActualStateAdapter, NodeProvisioner

**Files:**
- Modify: `deployment/src/main/java/io/casehub/ops/deployment/DeploymentGoalCompiler.java:36-48` — add `compileEntries(goals.pools(), ...)`
- Modify: `deployment/src/main/java/io/casehub/ops/deployment/DeploymentActualStateAdapter.java:39-46` — add `NodeType.of("pool")` to `handledTypes()`
- Modify: `deployment/src/main/java/io/casehub/ops/deployment/DeploymentNodeProvisioner.java` — inject `PoolProvisionHandler`, add to `handledTypes()`, add `case PoolNodeSpec` to switch expressions
- Modify: `deployment/src/test/java/io/casehub/ops/deployment/DeploymentGoalCompilerTest.java` — add pool compilation test
- Modify: `deployment/src/test/java/io/casehub/ops/deployment/DeploymentNodeProvisionerTest.java` — add pool dispatch test (if exists)
- Modify: `deployment/src/test/java/io/casehub/ops/deployment/DeploymentActualStateAdapterTest.java` — verify pool type handled

**Interfaces:**
- Consumes: `PoolNodeSpec`, `DeploymentGoals.pools()` from Task 1
- Consumes: `PoolProvisionHandler` from Task 2
- Consumes: `PoolDriftChecker` from Task 3

- [ ] **Step 1: Write GoalCompiler pool test**

Add this test to `DeploymentGoalCompilerTest.java`:

```java
@Test
void compilesPoolNode() {
    var pool = new PoolNodeSpec("code-reviewer", "claudony", 2, 8,
            "~/workspace/reviews", WorkingDirPolicy.SHARED_READ,
            new PoolScalingSpec("target-tracking", 0.7, "30s", null),
            EvictionStrategy.MEMORY_WEIGHTED);
    var goals = new DeploymentGoals(
            List.of(),
            List.of(),
            List.of(),
            List.of(),
            List.of(),
            List.of(),
            List.of(new GoalEntry<>(pool, List.of())),
            List.of());

    DesiredStateGraph graph = ((io.casehub.desiredstate.api.CompilationResult.SingleGraph)
            compiler.compile(goals, factory)).graph();

    assertThat(graph.nodes()).hasSize(1);
    DesiredNode node = graph.nodes().get(NodeId.of("code-reviewer-pool"));
    assertThat(node.id()).isEqualTo(NodeId.of("code-reviewer-pool"));
    assertThat(node.type().value()).isEqualTo("pool");
    assertThat(node.spec()).isInstanceOf(PoolNodeSpec.class);
}

@Test
void poolDependsOnAgent() {
    var agent = testAgent("code-reviewer");
    var pool = new PoolNodeSpec("code-reviewer", "claudony", 2, 8,
            null, null, null, null);
    var goals = new DeploymentGoals(
            List.of(new GoalEntry<>(agent, List.of())),
            List.of(),
            List.of(),
            List.of(),
            List.of(),
            List.of(),
            List.of(new GoalEntry<>(pool, List.of("code-reviewer"))),
            List.of());

    DesiredStateGraph graph = ((io.casehub.desiredstate.api.CompilationResult.SingleGraph)
            compiler.compile(goals, factory)).graph();

    assertThat(graph.nodes()).hasSize(2);
    assertThat(graph.dependencies()).hasSize(1);
    var dep = graph.dependencies().iterator().next();
    assertThat(dep.from()).isEqualTo(NodeId.of("code-reviewer-pool"));
    assertThat(dep.to()).isEqualTo(NodeId.of("code-reviewer"));
}
```

Add imports for `PoolNodeSpec`, `PoolScalingSpec`, `WorkingDirPolicy`, `EvictionStrategy`.

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode -o test -pl deployment -Dtest=DeploymentGoalCompilerTest#compilesPoolNode`
Expected: FAIL — `compile()` does not process `goals.pools()`

- [ ] **Step 3: Wire pool entries into DeploymentGoalCompiler**

In `DeploymentGoalCompiler.compile()`, add this line after `compileEntries(goals.detections(), nodes, dependencies);`:

```java
compileEntries(goals.pools(), nodes, dependencies);
```

- [ ] **Step 4: Run GoalCompiler tests to verify they pass**

Run: `mvn --batch-mode -o test -pl deployment -Dtest=DeploymentGoalCompilerTest`
Expected: PASS

- [ ] **Step 5: Wire pool type into DeploymentActualStateAdapter**

In `DeploymentActualStateAdapter.handledTypes()`, add `NodeType.of("pool")` to the set:

```java
public Set<NodeType> handledTypes() {
    return Set.of(
            NodeType.of("agent"),
            NodeType.of("channel"),
            NodeType.of("case_type"),
            NodeType.of("trust_policy"),
            NodeType.of("endpoint"),
            NodeType.of("detection"),
            NodeType.of("pool"));
}
```

- [ ] **Step 6: Wire PoolProvisionHandler into DeploymentNodeProvisioner**

Add `PoolProvisionHandler poolHandler` as a constructor parameter. Add `NodeType.of("pool")` to `handledTypes()`. Add `case PoolNodeSpec s -> poolHandler.provision(s, context);` to the `doProvision()` switch and `case PoolNodeSpec s -> poolHandler.deprovision(s, context);` to the `doDeprovision()` switch.

The constructor becomes:

```java
@Inject
public DeploymentNodeProvisioner(
        AgentRegistry agentRegistry,
        DeploymentProviderConfigStore providerConfigStore,
        ChannelProvisionHandler channelHandler,
        CaseTypeProvisionHandler caseTypeHandler,
        TrustPolicyProvisionHandler trustHandler,
        EndpointProvisionHandler endpointHandler,
        DetectionProvisionHandler detectionHandler,
        PoolProvisionHandler poolHandler,
        SpecHashStore specHashStore,
        ApprovalEvaluator approvalEvaluator,
        PlanStore planStore) {
    this.agentHandler = new AgentProvisionHandler(agentRegistry, providerConfigStore);
    this.channelHandler = channelHandler;
    this.caseTypeHandler = caseTypeHandler;
    this.trustHandler = trustHandler;
    this.endpointHandler = endpointHandler;
    this.detectionHandler = detectionHandler;
    this.poolHandler = poolHandler;
    this.specHashStore = specHashStore;
    this.approvalEvaluator = approvalEvaluator;
    this.planStore = planStore;
}
```

Add `private final PoolProvisionHandler poolHandler;` field.

In `handledTypes()`:

```java
return Set.of(
        NodeType.of("agent"),
        NodeType.of("channel"),
        NodeType.of("case_type"),
        NodeType.of("trust_policy"),
        NodeType.of("endpoint"),
        NodeType.of("detection"),
        NodeType.of("pool"));
```

In `doProvision()` switch:

```java
case PoolNodeSpec s -> poolHandler.provision(s, context);
```

In `doDeprovision()` switch:

```java
case PoolNodeSpec s -> poolHandler.deprovision(s, context);
```

- [ ] **Step 7: Fix DeploymentNodeProvisionerTest constructor calls**

Use `ide_find_references` on the `DeploymentNodeProvisioner` constructor to find all test call sites. Add `poolHandler` parameter to each constructor call. Tests can pass a `PoolProvisionHandler` constructed with the same `StubPoolOperations` pattern used in `PoolProvisionHandlerTest`.

- [ ] **Step 8: Run all deployment tests**

Run: `mvn --batch-mode -o test -pl deployment`
Expected: PASS

- [ ] **Step 9: Run full build**

Run: `mvn --batch-mode install`
Expected: PASS — all modules compile and tests pass

- [ ] **Step 10: Commit**

```bash
git add deployment/src/main/java/io/casehub/ops/deployment/DeploymentGoalCompiler.java
git add deployment/src/main/java/io/casehub/ops/deployment/DeploymentActualStateAdapter.java
git add deployment/src/main/java/io/casehub/ops/deployment/DeploymentNodeProvisioner.java
git add deployment/src/test/java/io/casehub/ops/deployment/DeploymentGoalCompilerTest.java
git add deployment/src/test/java/io/casehub/ops/deployment/DeploymentNodeProvisionerTest.java
git add deployment/src/test/java/io/casehub/ops/deployment/DeploymentActualStateAdapterTest.java
git commit -m "feat(#116): wire PoolNodeSpec into SPI quad — compiler, adapter, provisioner"
```

---

## References

- [2026-10-04-pool-nodespec-desiredstate-design.md] — design spec this plan implements
- [AgentNodeSpec.java] — pattern source for record shape
- [AgentProvisionHandler.java] — pattern source for handler
- [AgentDriftChecker.java] — pattern source for drift checker
- [ChannelProvisionHandler.java] — pattern source for PoolOperations interface
- [DeploymentGoalCompiler.java] — wiring point
- [DeploymentActualStateAdapter.java] — wiring point
- [DeploymentNodeProvisioner.java] — wiring point
- [DeploymentGoalCompilerTest.java] — test pattern source
- [AgentProvisionHandlerTest.java] — test pattern source
- [AgentDriftCheckerTest.java] — test pattern source
- [GitHub #116] — focal issue
