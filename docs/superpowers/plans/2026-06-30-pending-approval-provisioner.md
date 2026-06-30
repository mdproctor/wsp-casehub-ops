# PendingApproval Provisioner Support — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Enable ops provisioners to return `PendingApproval` for risk-gated changes, with shared approval types, a `PendingApprovalHandler` implementation, domain-specific risk evaluators, and plan storage.

**Architecture:** Shared approval types in `api/` (`io.casehub.ops.api.approval`). Each domain provides an `ApprovalEvaluator` that classifies risk and generates plans. The `OpsPendingApprovalHandler` manages pending/approved/rejected state. Provisioners follow a uniform flow: evaluate → if high-risk, store plan and return `PendingApproval` → on re-entry with approval, validate spec hasn't changed, then provision.

**Tech Stack:** Java 21 records/sealed interfaces, Quarkus CDI (`@ApplicationScoped`), JUnit 5, AssertJ

## Global Constraints

- All new types use Java records where immutable, sealed interfaces where polymorphic
- `@ApplicationScoped` beans in `api/` displace `@DefaultBean` no-ops in desiredstate runtime — this is the documented convention per `engine-spi-noops-defaultbean` protocol
- No casehub-work dependency — approval trigger mechanism is ops#32
- TDD: write failing test → implement → verify pass → commit
- Build must pass after every task: `mvn --batch-mode install`
- Use IntelliJ MCP for all renames, moves, and reference lookups — never bash grep/find for semantic operations

**Spec:** `specs/issue-13-pending-approval-provisioner/2026-06-30-pending-approval-provisioner-design.md`

---

### Task 1: Shared approval types and PlanStore

Create the foundational types in `io.casehub.ops.api.approval`. All subsequent tasks depend on this.

**Files:**
- Create: `api/src/main/java/io/casehub/ops/api/approval/RiskClassification.java`
- Create: `api/src/main/java/io/casehub/ops/api/approval/ApprovalThresholds.java`
- Create: `api/src/main/java/io/casehub/ops/api/approval/PlanDetail.java`
- Create: `api/src/main/java/io/casehub/ops/api/approval/ApprovalPlan.java`
- Create: `api/src/main/java/io/casehub/ops/api/approval/ApprovalDecision.java`
- Create: `api/src/main/java/io/casehub/ops/api/approval/ApprovalEvaluator.java`
- Create: `api/src/main/java/io/casehub/ops/api/approval/PlanStore.java`
- Create: `api/src/main/java/io/casehub/ops/api/approval/InMemoryPlanStore.java`
- Create: `api/src/test/java/io/casehub/ops/api/approval/ApprovalThresholdsTest.java`
- Create: `api/src/test/java/io/casehub/ops/api/approval/InMemoryPlanStoreTest.java`

**Interfaces:**
- Produces: `RiskClassification` enum (LOW, MEDIUM, HIGH, CRITICAL)
- Produces: `ApprovalThresholds(RiskClassification autoApproveBelow)` with `boolean requiresApproval(RiskClassification risk)`
- Produces: `PlanDetail` marker interface
- Produces: `ApprovalPlan(NodeId, StepAction, RiskClassification, String summary, String tenancyId, NodeSpec originalSpec, PlanDetail detail)`
- Produces: `ApprovalDecision` sealed interface: `AutoApproved()`, `RequiresApproval(ApprovalPlan plan)`
- Produces: `ApprovalEvaluator` interface: `ApprovalDecision evaluate(DesiredNode, StepAction, String tenancyId)`
- Produces: `PlanStore` interface: `String store(ApprovalPlan)`, `Optional<ApprovalPlan> retrieve(String)`, `void remove(String)`
- Produces: `InMemoryPlanStore` `@ApplicationScoped` impl

- [ ] **Step 1: Write ApprovalThresholds test**

```java
// api/src/test/java/io/casehub/ops/api/approval/ApprovalThresholdsTest.java
package io.casehub.ops.api.approval;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class ApprovalThresholdsTest {

    @Test
    void requiresApprovalAtThreshold() {
        var thresholds = new ApprovalThresholds(RiskClassification.HIGH);
        assertThat(thresholds.requiresApproval(RiskClassification.HIGH)).isTrue();
    }

    @Test
    void requiresApprovalAboveThreshold() {
        var thresholds = new ApprovalThresholds(RiskClassification.HIGH);
        assertThat(thresholds.requiresApproval(RiskClassification.CRITICAL)).isTrue();
    }

    @Test
    void doesNotRequireApprovalBelowThreshold() {
        var thresholds = new ApprovalThresholds(RiskClassification.HIGH);
        assertThat(thresholds.requiresApproval(RiskClassification.MEDIUM)).isFalse();
        assertThat(thresholds.requiresApproval(RiskClassification.LOW)).isFalse();
    }

    @Test
    void lowThresholdRequiresApprovalForEverything() {
        var thresholds = new ApprovalThresholds(RiskClassification.LOW);
        assertThat(thresholds.requiresApproval(RiskClassification.LOW)).isTrue();
        assertThat(thresholds.requiresApproval(RiskClassification.MEDIUM)).isTrue();
        assertThat(thresholds.requiresApproval(RiskClassification.HIGH)).isTrue();
        assertThat(thresholds.requiresApproval(RiskClassification.CRITICAL)).isTrue();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode -f api/pom.xml test -pl . -Dtest=ApprovalThresholdsTest -q`
Expected: compilation failure — `RiskClassification` and `ApprovalThresholds` do not exist

- [ ] **Step 3: Create all shared types**

Create these files in `api/src/main/java/io/casehub/ops/api/approval/`:

```java
// RiskClassification.java
package io.casehub.ops.api.approval;

public enum RiskClassification { LOW, MEDIUM, HIGH, CRITICAL }
```

```java
// ApprovalThresholds.java
package io.casehub.ops.api.approval;

public record ApprovalThresholds(RiskClassification autoApproveBelow) {
    public boolean requiresApproval(RiskClassification risk) {
        return risk.compareTo(autoApproveBelow) >= 0;
    }
}
```

```java
// PlanDetail.java
package io.casehub.ops.api.approval;

public interface PlanDetail {}
```

```java
// ApprovalPlan.java
package io.casehub.ops.api.approval;

import java.util.Objects;
import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.api.NodeSpec;
import io.casehub.desiredstate.api.StepAction;

public record ApprovalPlan(
        NodeId nodeId,
        StepAction action,
        RiskClassification risk,
        String summary,
        String tenancyId,
        NodeSpec originalSpec,
        PlanDetail detail) {

    public ApprovalPlan {
        Objects.requireNonNull(nodeId, "nodeId");
        Objects.requireNonNull(action, "action");
        Objects.requireNonNull(risk, "risk");
        Objects.requireNonNull(summary, "summary");
        Objects.requireNonNull(tenancyId, "tenancyId");
        Objects.requireNonNull(originalSpec, "originalSpec");
        // detail nullable
    }
}
```

```java
// ApprovalDecision.java
package io.casehub.ops.api.approval;

public sealed interface ApprovalDecision {
    record AutoApproved() implements ApprovalDecision {}
    record RequiresApproval(ApprovalPlan plan) implements ApprovalDecision {}
}
```

```java
// ApprovalEvaluator.java
package io.casehub.ops.api.approval;

import io.casehub.desiredstate.api.DesiredNode;
import io.casehub.desiredstate.api.StepAction;

public interface ApprovalEvaluator {
    ApprovalDecision evaluate(DesiredNode node, StepAction action, String tenancyId);
}
```

```java
// PlanStore.java
package io.casehub.ops.api.approval;

import java.util.Optional;

public interface PlanStore {
    String store(ApprovalPlan plan);
    Optional<ApprovalPlan> retrieve(String planReference);
    void remove(String planReference);
}
```

```java
// InMemoryPlanStore.java
package io.casehub.ops.api.approval;

import java.util.Optional;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;

import jakarta.enterprise.context.ApplicationScoped;

@ApplicationScoped
public class InMemoryPlanStore implements PlanStore {

    private final ConcurrentHashMap<String, ApprovalPlan> plans = new ConcurrentHashMap<>();

    @Override
    public String store(ApprovalPlan plan) {
        String ref = UUID.randomUUID().toString();
        plans.put(ref, plan);
        return ref;
    }

    @Override
    public Optional<ApprovalPlan> retrieve(String planReference) {
        return Optional.ofNullable(plans.get(planReference));
    }

    @Override
    public void remove(String planReference) {
        plans.remove(planReference);
    }
}
```

- [ ] **Step 4: Run ApprovalThresholds test to verify it passes**

Run: `mvn --batch-mode -f api/pom.xml test -Dtest=ApprovalThresholdsTest -q`
Expected: PASS

- [ ] **Step 5: Write InMemoryPlanStore test**

```java
// api/src/test/java/io/casehub/ops/api/approval/InMemoryPlanStoreTest.java
package io.casehub.ops.api.approval;

import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.api.NodeSpec;
import io.casehub.desiredstate.api.StepAction;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class InMemoryPlanStoreTest {

    private InMemoryPlanStore store;

    @BeforeEach
    void setUp() {
        store = new InMemoryPlanStore();
    }

    private ApprovalPlan testPlan() {
        return new ApprovalPlan(
                NodeId.of("node-1"), StepAction.PROVISION, RiskClassification.HIGH,
                "Test plan", "tenant-1", new StubSpec(), null);
    }

    @Test
    void storeAndRetrieve() {
        var plan = testPlan();
        String ref = store.store(plan);
        assertThat(ref).isNotNull().isNotBlank();
        assertThat(store.retrieve(ref)).contains(plan);
    }

    @Test
    void retrieveMissingKeyReturnsEmpty() {
        assertThat(store.retrieve("nonexistent")).isEmpty();
    }

    @Test
    void removeDeletesPlan() {
        String ref = store.store(testPlan());
        store.remove(ref);
        assertThat(store.retrieve(ref)).isEmpty();
    }

    @Test
    void removeNonexistentKeyIsNoOp() {
        store.remove("nonexistent"); // should not throw
    }

    @Test
    void eachStoreGeneratesUniqueReference() {
        String ref1 = store.store(testPlan());
        String ref2 = store.store(testPlan());
        assertThat(ref1).isNotEqualTo(ref2);
    }

    private record StubSpec() implements NodeSpec {}
}
```

- [ ] **Step 6: Run InMemoryPlanStore test to verify it passes**

Run: `mvn --batch-mode -f api/pom.xml test -Dtest=InMemoryPlanStoreTest -q`
Expected: PASS

- [ ] **Step 7: Full build verification**

Run: `mvn --batch-mode install -q`
Expected: BUILD SUCCESS (existing tests still pass — no callers of new types yet)

- [ ] **Step 8: Commit**

```
git add api/src/main/java/io/casehub/ops/api/approval/ api/src/test/java/io/casehub/ops/api/approval/
git commit -m "feat(#13): shared approval types — RiskClassification, ApprovalThresholds, PlanStore, ApprovalEvaluator"
```

---

### Task 2: Infra type migration

Move `RiskClassification` and `RiskThresholds` from infra-specific packages to the shared `approval` package. Delete originals, update all callers.

**Files:**
- Delete: `api/src/main/java/io/casehub/ops/api/infra/context/RiskClassification.java`
- Delete: `api/src/main/java/io/casehub/ops/api/infra/context/RiskThresholds.java`
- Modify: `api/src/main/java/io/casehub/ops/api/infra/context/InfraProvisionContext.java` — field type `RiskThresholds` → `ApprovalThresholds`
- Modify: `api/src/main/java/io/casehub/ops/api/infra/plan/ProvisionPlan.java` — add `implements PlanDetail`, rename `humanReadableSummary` → `summary`
- Modify: `infra/src/main/java/io/casehub/ops/infra/InfraNodeProvisioner.java` — update imports and constant type
- Modify: `infra/src/test/java/io/casehub/ops/infra/InfraNodeProvisionerTest.java` — update imports
- Modify: `infra/src/test/java/io/casehub/ops/infra/standalone/StandaloneBackendTest.java` — update imports and constant
- Modify: `api/src/test/java/io/casehub/ops/api/infra/spi/InfraBackendContractTest.java` — update imports and all `RiskThresholds` → `ApprovalThresholds`

**Interfaces:**
- Consumes: `RiskClassification` from `io.casehub.ops.api.approval` (Task 1)
- Consumes: `ApprovalThresholds` from `io.casehub.ops.api.approval` (Task 1)
- Produces: `ProvisionPlan implements PlanDetail` with `summary()` field (renamed from `humanReadableSummary()`)

- [ ] **Step 1: Use IntelliJ to rename `humanReadableSummary` → `summary` in ProvisionPlan**

Use `ide_refactor_rename` on `ProvisionPlan.humanReadableSummary` field (line 13, column 16):

```
ide_refactor_rename(file="api/src/main/java/io/casehub/ops/api/infra/plan/ProvisionPlan.java", line=13, column=16, newName="summary")
```

This renames the field, the accessor method, and the constructor parameter in one operation. Also update the compact constructor's `Objects.requireNonNull` call: change the string literal `"humanReadableSummary"` to `"summary"`.

- [ ] **Step 2: Add `implements PlanDetail` to ProvisionPlan**

Change the record declaration from:
```java
public record ProvisionPlan(
```
to:
```java
import io.casehub.ops.api.approval.PlanDetail;

public record ProvisionPlan(
        NodeId nodeId,
        List<PlannedChange> changes,
        RiskClassification risk,
        String summary,
        ToolPlanDetail toolDetail) implements PlanDetail {
```

Update the import of `RiskClassification` from `io.casehub.ops.api.infra.context.RiskClassification` to `io.casehub.ops.api.approval.RiskClassification`.

- [ ] **Step 3: Update InfraProvisionContext**

Change `RiskThresholds` → `ApprovalThresholds` in `InfraProvisionContext.java`:
- Replace import `io.casehub.ops.api.infra.context.RiskThresholds` with `io.casehub.ops.api.approval.ApprovalThresholds`
- Replace import `io.casehub.ops.api.infra.context.RiskClassification` with `io.casehub.ops.api.approval.RiskClassification` (if present)
- Change field type: `RiskThresholds thresholds` → `ApprovalThresholds thresholds`
- Update compact constructor: `Objects.requireNonNull(thresholds, "thresholds")` stays the same

- [ ] **Step 4: Delete old type files**

Delete `api/src/main/java/io/casehub/ops/api/infra/context/RiskClassification.java` and `api/src/main/java/io/casehub/ops/api/infra/context/RiskThresholds.java`.

- [ ] **Step 5: Update InfraNodeProvisioner imports and constant**

In `infra/src/main/java/io/casehub/ops/infra/InfraNodeProvisioner.java`:
- Replace `import io.casehub.ops.api.infra.context.RiskClassification` with `import io.casehub.ops.api.approval.RiskClassification`
- Replace `import io.casehub.ops.api.infra.context.RiskThresholds` with `import io.casehub.ops.api.approval.ApprovalThresholds`
- Change constant: `private static final RiskThresholds DEFAULT_THRESHOLDS = new RiskThresholds(RiskClassification.LOW, false)` → `private static final ApprovalThresholds DEFAULT_THRESHOLDS = new ApprovalThresholds(RiskClassification.LOW)`

Note: `RiskThresholds` had `(autoApproveBelow, requireSecondReviewer)`. `ApprovalThresholds` has only `(autoApproveBelow)` — drop the boolean argument.

- [ ] **Step 6: Update test files**

In `infra/src/test/java/io/casehub/ops/infra/standalone/StandaloneBackendTest.java`:
- Replace imports: `RiskClassification` → `io.casehub.ops.api.approval.RiskClassification`, `RiskThresholds` → `io.casehub.ops.api.approval.ApprovalThresholds`
- Change constant: `new RiskThresholds(RiskClassification.LOW, false)` → `new ApprovalThresholds(RiskClassification.LOW)`

In `api/src/test/java/io/casehub/ops/api/infra/spi/InfraBackendContractTest.java`:
- Replace imports: same as above
- Change all `new RiskThresholds(RiskClassification.X, false)` → `new ApprovalThresholds(RiskClassification.X)` (3 occurrences: lines 292, 301, 535)

- [ ] **Step 7: Full build verification**

Run: `mvn --batch-mode install -q`
Expected: BUILD SUCCESS — all existing tests pass with updated types

- [ ] **Step 8: Commit**

```
git add -A
git commit -m "refactor(#13): migrate RiskClassification and RiskThresholds to shared approval package"
```

---

### Task 3: OpsPendingApprovalHandler

The `PendingApprovalHandler` implementation with full state machine — the core approval mechanism.

**Files:**
- Create: `api/src/main/java/io/casehub/ops/api/approval/OpsPendingApprovalHandler.java`
- Create: `api/src/test/java/io/casehub/ops/api/approval/OpsPendingApprovalHandlerTest.java`

**Interfaces:**
- Consumes: `PlanStore` (Task 1)
- Produces: `OpsPendingApprovalHandler` implementing `io.casehub.desiredstate.api.PendingApprovalHandler`
- Produces: `approve(NodeId, StepAction, String tenancyId, String approvedBy) → boolean`
- Produces: `reject(NodeId, StepAction, String tenancyId, String reason) → boolean`

- [ ] **Step 1: Write OpsPendingApprovalHandler test**

```java
// api/src/test/java/io/casehub/ops/api/approval/OpsPendingApprovalHandlerTest.java
package io.casehub.ops.api.approval;

import io.casehub.desiredstate.api.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class OpsPendingApprovalHandlerTest {

    private OpsPendingApprovalHandler handler;
    private InMemoryPlanStore planStore;

    private static final NodeId NODE_1 = NodeId.of("node-1");
    private static final String TENANT = "tenant-1";

    @BeforeEach
    void setUp() {
        planStore = new InMemoryPlanStore();
        handler = new OpsPendingApprovalHandler(planStore);
    }

    private DesiredNode testNode() {
        return new DesiredNode(NODE_1, NodeType.of("agent"), new StubSpec(), false);
    }

    // --- check() returns None when nothing recorded ---

    @Test
    void checkReturnsNoneWhenEmpty() {
        var result = handler.check(testNode(), StepAction.PROVISION, TENANT);
        assertThat(result).isInstanceOf(ApprovalCheckResult.None.class);
    }

    // --- recordPending → check returns Pending ---

    @Test
    void recordPendingThenCheckReturnsPending() {
        handler.recordPending(testNode(), StepAction.PROVISION, TENANT, "plan-ref-1");
        var result = handler.check(testNode(), StepAction.PROVISION, TENANT);
        assertThat(result).isInstanceOf(ApprovalCheckResult.Pending.class);
        assertThat(((ApprovalCheckResult.Pending) result).planReference()).isEqualTo("plan-ref-1");
    }

    @Test
    void recordPendingReturnsSkipped() {
        var outcome = handler.recordPending(testNode(), StepAction.PROVISION, TENANT, "plan-ref-1");
        assertThat(outcome).isInstanceOf(StepOutcome.Skipped.class);
        assertThat(((StepOutcome.Skipped) outcome).reason()).contains("plan-ref-1");
    }

    // --- approve → check returns Approved ---

    @Test
    void approveTransitionsPendingToApproved() {
        handler.recordPending(testNode(), StepAction.PROVISION, TENANT, "plan-ref-1");
        boolean result = handler.approve(NODE_1, StepAction.PROVISION, TENANT, "admin");
        assertThat(result).isTrue();

        var check = handler.check(testNode(), StepAction.PROVISION, TENANT);
        assertThat(check).isInstanceOf(ApprovalCheckResult.Approved.class);
        var approved = (ApprovalCheckResult.Approved) check;
        assertThat(approved.approval().planReference()).isEqualTo("plan-ref-1");
        assertThat(approved.approval().approvedBy()).isEqualTo("admin");
        assertThat(approved.approval().approvedAt()).isNotNull();
    }

    @Test
    void approvedNotConsumedOnRead() {
        handler.recordPending(testNode(), StepAction.PROVISION, TENANT, "plan-ref-1");
        handler.approve(NODE_1, StepAction.PROVISION, TENANT, "admin");

        var check1 = handler.check(testNode(), StepAction.PROVISION, TENANT);
        var check2 = handler.check(testNode(), StepAction.PROVISION, TENANT);
        assertThat(check1).isInstanceOf(ApprovalCheckResult.Approved.class);
        assertThat(check2).isInstanceOf(ApprovalCheckResult.Approved.class);
    }

    // --- reject → check returns Rejected ---

    @Test
    void rejectTransitionsPendingToRejected() {
        handler.recordPending(testNode(), StepAction.PROVISION, TENANT, "plan-ref-1");
        boolean result = handler.reject(NODE_1, StepAction.PROVISION, TENANT, "too risky");
        assertThat(result).isTrue();

        var check = handler.check(testNode(), StepAction.PROVISION, TENANT);
        assertThat(check).isInstanceOf(ApprovalCheckResult.Rejected.class);
        var rejected = (ApprovalCheckResult.Rejected) check;
        assertThat(rejected.reason()).isEqualTo("too risky");
    }

    // --- acknowledgeRejection removes entry and cleans up plan ---

    @Test
    void acknowledgeRejectionRemovesEntry() {
        String planRef = planStore.store(new ApprovalPlan(
                NODE_1, StepAction.PROVISION, RiskClassification.HIGH,
                "test", TENANT, new StubSpec(), null));
        handler.recordPending(testNode(), StepAction.PROVISION, TENANT, planRef);
        handler.reject(NODE_1, StepAction.PROVISION, TENANT, "too risky");

        handler.acknowledgeRejection(testNode(), StepAction.PROVISION, TENANT);

        assertThat(handler.check(testNode(), StepAction.PROVISION, TENANT))
                .isInstanceOf(ApprovalCheckResult.None.class);
        assertThat(planStore.retrieve(planRef)).isEmpty();
    }

    // --- precondition failures ---

    @Test
    void approveReturnsFalseWhenNoEntry() {
        boolean result = handler.approve(NODE_1, StepAction.PROVISION, TENANT, "admin");
        assertThat(result).isFalse();
    }

    @Test
    void approveReturnsFalseWhenAlreadyApproved() {
        handler.recordPending(testNode(), StepAction.PROVISION, TENANT, "plan-ref-1");
        handler.approve(NODE_1, StepAction.PROVISION, TENANT, "admin");
        boolean result = handler.approve(NODE_1, StepAction.PROVISION, TENANT, "admin2");
        assertThat(result).isFalse();
    }

    @Test
    void rejectReturnsFalseWhenNoEntry() {
        boolean result = handler.reject(NODE_1, StepAction.PROVISION, TENANT, "reason");
        assertThat(result).isFalse();
    }

    // --- recordPending supersedes stale entry ---

    @Test
    void recordPendingSupersedes() {
        handler.recordPending(testNode(), StepAction.PROVISION, TENANT, "old-ref");
        handler.approve(NODE_1, StepAction.PROVISION, TENANT, "admin");
        handler.recordPending(testNode(), StepAction.PROVISION, TENANT, "new-ref");

        var check = handler.check(testNode(), StepAction.PROVISION, TENANT);
        assertThat(check).isInstanceOf(ApprovalCheckResult.Pending.class);
        assertThat(((ApprovalCheckResult.Pending) check).planReference()).isEqualTo("new-ref");
    }

    // --- action isolation ---

    @Test
    void provisionAndDeprovisionAreIndependent() {
        handler.recordPending(testNode(), StepAction.PROVISION, TENANT, "prov-ref");
        handler.recordPending(testNode(), StepAction.DEPROVISION, TENANT, "deprov-ref");

        var provCheck = handler.check(testNode(), StepAction.PROVISION, TENANT);
        var deprovCheck = handler.check(testNode(), StepAction.DEPROVISION, TENANT);
        assertThat(((ApprovalCheckResult.Pending) provCheck).planReference()).isEqualTo("prov-ref");
        assertThat(((ApprovalCheckResult.Pending) deprovCheck).planReference()).isEqualTo("deprov-ref");
    }

    private record StubSpec() implements NodeSpec {}
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode -f api/pom.xml test -Dtest=OpsPendingApprovalHandlerTest -q`
Expected: compilation failure — `OpsPendingApprovalHandler` does not exist

- [ ] **Step 3: Implement OpsPendingApprovalHandler**

```java
// api/src/main/java/io/casehub/ops/api/approval/OpsPendingApprovalHandler.java
package io.casehub.ops.api.approval;

import java.time.Instant;
import java.util.concurrent.ConcurrentHashMap;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import io.casehub.desiredstate.api.ApprovalCheckResult;
import io.casehub.desiredstate.api.DesiredNode;
import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.api.PendingApprovalHandler;
import io.casehub.desiredstate.api.PlanApproval;
import io.casehub.desiredstate.api.StepAction;
import io.casehub.desiredstate.api.StepOutcome;

@ApplicationScoped
public class OpsPendingApprovalHandler implements PendingApprovalHandler {

    private final PlanStore planStore;
    private final ConcurrentHashMap<String, PendingEntry> entries = new ConcurrentHashMap<>();

    @Inject
    public OpsPendingApprovalHandler(PlanStore planStore) {
        this.planStore = planStore;
    }

    @Override
    public ApprovalCheckResult check(DesiredNode node, StepAction action, String tenancyId) {
        var entry = entries.get(key(node.id(), action, tenancyId));
        if (entry == null) {
            return new ApprovalCheckResult.None();
        }
        return switch (entry.status) {
            case PENDING -> new ApprovalCheckResult.Pending(entry.planReference);
            case APPROVED -> new ApprovalCheckResult.Approved(entry.approval);
            case REJECTED -> new ApprovalCheckResult.Rejected(entry.planReference, entry.rejectionReason);
        };
    }

    @Override
    public StepOutcome recordPending(DesiredNode node, StepAction action, String tenancyId, String planReference) {
        entries.put(key(node.id(), action, tenancyId),
                new PendingEntry(planReference, Status.PENDING, null, null));
        return new StepOutcome.Skipped("pending approval: " + planReference);
    }

    @Override
    public void acknowledgeRejection(DesiredNode node, StepAction action, String tenancyId) {
        var removed = entries.remove(key(node.id(), action, tenancyId));
        if (removed != null) {
            planStore.remove(removed.planReference);
        }
    }

    public boolean approve(NodeId nodeId, StepAction action, String tenancyId, String approvedBy) {
        String k = key(nodeId, action, tenancyId);
        var entry = entries.get(k);
        if (entry == null || entry.status != Status.PENDING) {
            return false;
        }
        var approval = new PlanApproval(entry.planReference, approvedBy, Instant.now());
        entries.put(k, new PendingEntry(entry.planReference, Status.APPROVED, approval, null));
        return true;
    }

    public boolean reject(NodeId nodeId, StepAction action, String tenancyId, String reason) {
        String k = key(nodeId, action, tenancyId);
        var entry = entries.get(k);
        if (entry == null || entry.status != Status.PENDING) {
            return false;
        }
        entries.put(k, new PendingEntry(entry.planReference, Status.REJECTED, null, reason));
        return true;
    }

    private String key(NodeId nodeId, StepAction action, String tenancyId) {
        return nodeId.value() + ":" + action.name() + ":" + tenancyId;
    }

    private enum Status { PENDING, APPROVED, REJECTED }

    private record PendingEntry(String planReference, Status status, PlanApproval approval, String rejectionReason) {}
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode -f api/pom.xml test -Dtest=OpsPendingApprovalHandlerTest -q`
Expected: PASS

- [ ] **Step 5: Full build verification**

Run: `mvn --batch-mode install -q`
Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```
git add api/src/main/java/io/casehub/ops/api/approval/OpsPendingApprovalHandler.java api/src/test/java/io/casehub/ops/api/approval/OpsPendingApprovalHandlerTest.java
git commit -m "feat(#13): OpsPendingApprovalHandler — PendingApprovalHandler impl with state machine"
```

---

### Task 4: Deployment domain — evaluator and provisioner approval flow

Create `DeploymentApprovalEvaluator` and modify `DeploymentNodeProvisioner` to support the approval lifecycle.

**Files:**
- Create: `deployment/src/main/java/io/casehub/ops/deployment/DeploymentApprovalEvaluator.java`
- Create: `deployment/src/test/java/io/casehub/ops/deployment/DeploymentApprovalEvaluatorTest.java`
- Modify: `deployment/src/main/java/io/casehub/ops/deployment/DeploymentNodeProvisioner.java`
- Modify: `deployment/src/test/java/io/casehub/ops/deployment/DeploymentNodeProvisionerTest.java`

**Interfaces:**
- Consumes: `ApprovalEvaluator`, `ApprovalDecision`, `ApprovalPlan`, `PlanStore` (Task 1)
- Consumes: `RiskClassification` from `io.casehub.ops.api.approval` (Task 1)
- Produces: `DeploymentApprovalEvaluator` — classifies deployment specs by risk
- Produces: Modified `DeploymentNodeProvisioner` with approval flow for both provision and deprovision

- [ ] **Step 1: Write DeploymentApprovalEvaluator test**

```java
// deployment/src/test/java/io/casehub/ops/deployment/DeploymentApprovalEvaluatorTest.java
package io.casehub.ops.deployment;

import io.casehub.desiredstate.api.*;
import io.casehub.ops.api.approval.*;
import io.casehub.ops.api.deployment.*;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class DeploymentApprovalEvaluatorTest {

    private final DeploymentApprovalEvaluator evaluator = new DeploymentApprovalEvaluator();

    @Test
    void trustPolicyRequiresApproval() {
        var spec = new TrustPolicyNodeSpec("claims-routing", 0.85, 10, 0.1, 0.3, Map.of(), false);
        var node = new DesiredNode(NodeId.of("tp-1"), NodeType.of("trust"), spec, false);

        var decision = evaluator.evaluate(node, StepAction.PROVISION, "tenant-1");

        assertThat(decision).isInstanceOf(ApprovalDecision.RequiresApproval.class);
        var req = (ApprovalDecision.RequiresApproval) decision;
        assertThat(req.plan().risk()).isEqualTo(RiskClassification.HIGH);
        assertThat(req.plan().summary()).contains("claims-routing");
        assertThat(req.plan().originalSpec()).isEqualTo(spec);
    }

    @Test
    void channelAutoApproves() {
        var spec = new ChannelNodeSpec("dev/work", "desc", null,
                null, null, null, null, null, null, null, null, null, null, null);
        var node = new DesiredNode(NodeId.of("ch-1"), NodeType.of("channel"), spec, false);

        var decision = evaluator.evaluate(node, StepAction.PROVISION, "tenant-1");
        assertThat(decision).isInstanceOf(ApprovalDecision.AutoApproved.class);
    }

    @Test
    void agentAutoApprovesWithDefaultThresholds() {
        var spec = new AgentNodeSpec("agent-1", "Agent", "worker", "anthropic", "claude", "4.6",
                "1.0", null, null, null, null, null, List.of(), null, null, null, null, List.of());
        var node = new DesiredNode(NodeId.of("a-1"), NodeType.of("agent"), spec, false);

        var decision = evaluator.evaluate(node, StepAction.PROVISION, "tenant-1");
        assertThat(decision).isInstanceOf(ApprovalDecision.AutoApproved.class);
    }

    @Test
    void caseTypeAutoApproves() {
        var spec = new CaseTypeNodeSpec("ns", "name", "1.0", "Title", "Summary", null, null);
        var node = new DesiredNode(NodeId.of("ct-1"), NodeType.of("casetype"), spec, false);

        var decision = evaluator.evaluate(node, StepAction.PROVISION, "tenant-1");
        assertThat(decision).isInstanceOf(ApprovalDecision.AutoApproved.class);
    }

    @Test
    void endpointAutoApproves() {
        var spec = new EndpointNodeSpec("/api/v1/claims", "Claims API", "tenant-1",
                "claims-handler", List.of("GET", "POST"));
        var node = new DesiredNode(NodeId.of("ep-1"), NodeType.of("endpoint"), spec, false);

        var decision = evaluator.evaluate(node, StepAction.PROVISION, "tenant-1");
        assertThat(decision).isInstanceOf(ApprovalDecision.AutoApproved.class);
    }

    @Test
    void nonDeploymentSpecAutoApproves() {
        NodeSpec unknownSpec = new NodeSpec() {};
        var node = new DesiredNode(NodeId.of("x-1"), NodeType.of("unknown"), unknownSpec, false);

        var decision = evaluator.evaluate(node, StepAction.PROVISION, "tenant-1");
        assertThat(decision).isInstanceOf(ApprovalDecision.AutoApproved.class);
    }

    @Test
    void deprovisionTrustPolicyAlsoRequiresApproval() {
        var spec = new TrustPolicyNodeSpec("claims-routing", 0.85, 10, 0.1, 0.3, Map.of(), false);
        var node = new DesiredNode(NodeId.of("tp-1"), NodeType.of("trust"), spec, false);

        var decision = evaluator.evaluate(node, StepAction.DEPROVISION, "tenant-1");
        assertThat(decision).isInstanceOf(ApprovalDecision.RequiresApproval.class);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode -f deployment/pom.xml test -Dtest=DeploymentApprovalEvaluatorTest -q`
Expected: compilation failure — `DeploymentApprovalEvaluator` does not exist

- [ ] **Step 3: Implement DeploymentApprovalEvaluator**

```java
// deployment/src/main/java/io/casehub/ops/deployment/DeploymentApprovalEvaluator.java
package io.casehub.ops.deployment;

import jakarta.enterprise.context.ApplicationScoped;

import io.casehub.desiredstate.api.DesiredNode;
import io.casehub.desiredstate.api.StepAction;
import io.casehub.ops.api.approval.*;
import io.casehub.ops.api.deployment.*;

@ApplicationScoped
public class DeploymentApprovalEvaluator implements ApprovalEvaluator {

    private static final ApprovalThresholds THRESHOLDS =
            new ApprovalThresholds(RiskClassification.HIGH);

    @Override
    public ApprovalDecision evaluate(DesiredNode node, StepAction action, String tenancyId) {
        if (!(node.spec() instanceof DeploymentNodeSpec spec)) {
            return new ApprovalDecision.AutoApproved();
        }

        RiskClassification risk = classifyRisk(spec);
        if (!THRESHOLDS.requiresApproval(risk)) {
            return new ApprovalDecision.AutoApproved();
        }

        var plan = new ApprovalPlan(
                node.id(), action, risk,
                generateSummary(spec, action),
                tenancyId, node.spec(), null);
        return new ApprovalDecision.RequiresApproval(plan);
    }

    private RiskClassification classifyRisk(DeploymentNodeSpec spec) {
        return switch (spec) {
            case TrustPolicyNodeSpec s -> RiskClassification.HIGH;
            case AgentNodeSpec s -> RiskClassification.MEDIUM;
            case ChannelNodeSpec s -> RiskClassification.LOW;
            case CaseTypeNodeSpec s -> RiskClassification.LOW;
            case EndpointNodeSpec s -> RiskClassification.LOW;
        };
    }

    private String generateSummary(DeploymentNodeSpec spec, StepAction action) {
        String verb = action == StepAction.PROVISION ? "Provision" : "Deprovision";
        return switch (spec) {
            case TrustPolicyNodeSpec s -> verb + " trust policy '" + s.capability()
                    + "': threshold=" + s.threshold() + ", blendFactor=" + s.blendFactor();
            case AgentNodeSpec s -> verb + " agent '" + s.agentId()
                    + "' (" + s.modelProvider() + "/" + s.modelName() + ")";
            case ChannelNodeSpec s -> verb + " channel '" + s.name() + "'";
            case CaseTypeNodeSpec s -> verb + " case type '" + s.namespace() + "/" + s.name() + "'";
            case EndpointNodeSpec s -> verb + " endpoint '" + s.path() + "'";
        };
    }
}
```

- [ ] **Step 4: Run evaluator test to verify it passes**

Run: `mvn --batch-mode -f deployment/pom.xml test -Dtest=DeploymentApprovalEvaluatorTest -q`
Expected: PASS

- [ ] **Step 5: Write DeploymentNodeProvisioner approval flow test**

Add new test methods to the existing `DeploymentNodeProvisionerTest.java`. The test needs updated setup to inject the evaluator and plan store.

Read the current test file first with `ide_read_file`, then add:
- Updated `setUp()` with evaluator and plan store injection
- Test: high-risk node (TrustPolicy) returns PendingApproval
- Test: low-risk node (Channel) provisions directly
- Test: re-entry with valid approval provisions successfully
- Test: re-entry with stale spec re-evaluates
- Test: deprovision approval flow

The `DeploymentNodeProvisioner` constructor gains two new parameters: `ApprovalEvaluator` and `PlanStore`. Existing tests continue to pass because the evaluator auto-approves channels, agents, case types, and endpoints.

- [ ] **Step 6: Implement DeploymentNodeProvisioner approval flow**

Modify `DeploymentNodeProvisioner.java`:
- Add `ApprovalEvaluator` and `PlanStore` constructor parameters (injected via CDI)
- In `provision()`: check `context.hasApproval()` → re-entry flow. Otherwise evaluate → if `RequiresApproval`, store plan and return `PendingApproval`. If `AutoApproved`, call `doProvision()`.
- Extract existing handler dispatch into private `doProvision(DeploymentNodeSpec, ProvisionContext)` and `doDeprovision(DeploymentNodeSpec, DeprovisionContext)`.
- Re-entry: retrieve plan, compare `originalSpec` via `equals()`, if match → `doProvision()` + `planStore.remove()` on success. If mismatch → `planStore.remove()`, strip approval, re-evaluate.
- Same pattern for `deprovision()`.

- [ ] **Step 7: Run all deployment tests**

Run: `mvn --batch-mode -f deployment/pom.xml test -q`
Expected: ALL PASS

- [ ] **Step 8: Full build verification**

Run: `mvn --batch-mode install -q`
Expected: BUILD SUCCESS

- [ ] **Step 9: Commit**

```
git add deployment/src/main/java/io/casehub/ops/deployment/DeploymentApprovalEvaluator.java deployment/src/test/java/io/casehub/ops/deployment/DeploymentApprovalEvaluatorTest.java deployment/src/main/java/io/casehub/ops/deployment/DeploymentNodeProvisioner.java deployment/src/test/java/io/casehub/ops/deployment/DeploymentNodeProvisionerTest.java
git commit -m "feat(#13): deployment approval flow — evaluator + provisioner re-entry"
```

---

### Task 5: Infra domain — evaluator and provisioner approval flow

Create `InfraApprovalEvaluator` (delegates to `InfraBackend.plan()`) and modify `InfraNodeProvisioner` for the plan/apply approval lifecycle.

**Files:**
- Create: `infra/src/main/java/io/casehub/ops/infra/InfraApprovalEvaluator.java`
- Create: `infra/src/test/java/io/casehub/ops/infra/InfraApprovalEvaluatorTest.java`
- Modify: `infra/src/main/java/io/casehub/ops/infra/InfraNodeProvisioner.java`
- Modify: `infra/src/test/java/io/casehub/ops/infra/InfraNodeProvisionerTest.java`

**Interfaces:**
- Consumes: `ApprovalEvaluator`, `PlanStore`, `ApprovalPlan`, `PlanDetail` (Task 1)
- Consumes: `ProvisionPlan implements PlanDetail` (Task 2)
- Consumes: `InfraBackend.plan()` — existing SPI
- Produces: `InfraApprovalEvaluator` — delegates to backend's `plan()` method for risk
- Produces: Modified `InfraNodeProvisioner` with plan/apply lifecycle wired up

- [ ] **Step 1: Write InfraApprovalEvaluator test**

Tests use the existing `TrackingBackend` pattern from `InfraNodeProvisionerTest`. The evaluator calls `backend.plan()` to get a `ProvisionPlan`, checks risk against thresholds. If the backend returns empty (no changes), auto-approve. If risk is below threshold, auto-approve. If above, return `RequiresApproval` with `ProvisionPlan` as `PlanDetail`.

Key tests:
- Backend returns empty plan → AutoApproved
- Backend returns LOW risk plan → AutoApproved (default threshold is HIGH)
- Backend returns HIGH risk plan → RequiresApproval with ProvisionPlan as detail
- Non-InfraDesiredNodeSpec → AutoApproved

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Implement InfraApprovalEvaluator**

The evaluator needs the same backend map as `InfraNodeProvisioner`. It resolves the backend from the `InfraDesiredNodeSpec.backendId()`, calls `backend.plan()` (which returns `Uni<Optional<ProvisionPlan>>`), awaits the result, and evaluates risk.

Constructor injects `@Any Instance<InfraBackend>` (same as InfraNodeProvisioner) and `ApprovalThresholds`.

- [ ] **Step 4: Run evaluator test to verify it passes**

- [ ] **Step 5: Write InfraNodeProvisioner approval flow tests**

Add to existing `InfraNodeProvisionerTest`:
- Updated constructor with evaluator and plan store
- Test: evaluator returns RequiresApproval → provisioner returns PendingApproval
- Test: evaluator returns AutoApproved → provisioner provisions directly (existing behavior)
- Test: re-entry with approval → builds InfraProvisionContext with APPLY phase and approvedPlan from stored ProvisionPlan
- Test: re-entry with stale spec → re-evaluates

- [ ] **Step 6: Implement InfraNodeProvisioner approval flow**

Modify `InfraNodeProvisioner.java`:
- Remove the stale TODO comment
- Add `ApprovalEvaluator` and `PlanStore` to constructor
- In `provision()`: check `context.hasApproval()` → retrieve ProvisionPlan from stored ApprovalPlan via pattern matching (`plan.detail() instanceof ProvisionPlan infraPlan`) → build InfraProvisionContext with `ProvisionPhase.APPLY` and `approvedPlan = infraPlan` → call backend. On success, remove plan.
- First call: delegate to evaluator. If RequiresApproval, store plan, return PendingApproval. If AutoApproved, provision as before.

- [ ] **Step 7: Run all infra tests**

Run: `mvn --batch-mode -f infra/pom.xml test -q`
Expected: ALL PASS

- [ ] **Step 8: Full build verification**

Run: `mvn --batch-mode install -q`
Expected: BUILD SUCCESS

- [ ] **Step 9: Commit**

```
git add infra/src/
git commit -m "feat(#13): infra approval flow — evaluator delegates to InfraBackend.plan(), provisioner wires plan/apply lifecycle"
```

---

### Task 6: Compliance and IoT stub evaluators

Add stub evaluators returning `AutoApproved` and wire the approval flow into compliance and IoT provisioners.

**Files:**
- Create: `compliance/src/main/java/io/casehub/ops/compliance/ComplianceApprovalEvaluator.java`
- Create: `compliance/src/test/java/io/casehub/ops/compliance/ComplianceApprovalEvaluatorTest.java`
- Create: `iot/src/main/java/io/casehub/ops/iot/IoTApprovalEvaluator.java`
- Create: `iot/src/test/java/io/casehub/ops/iot/IoTApprovalEvaluatorTest.java`
- Modify: `compliance/src/main/java/io/casehub/ops/compliance/ComplianceNodeProvisioner.java`
- Modify: `compliance/src/test/java/io/casehub/ops/compliance/ComplianceNodeProvisionerTest.java`
- Modify: `iot/src/main/java/io/casehub/ops/iot/IoTNodeProvisioner.java`
- Modify: `iot/src/test/java/io/casehub/ops/iot/IoTNodeProvisionerTest.java`

**Interfaces:**
- Consumes: `ApprovalEvaluator`, `ApprovalDecision`, `PlanStore` (Task 1)
- Produces: `ComplianceApprovalEvaluator` and `IoTApprovalEvaluator` — both return `AutoApproved` for all nodes
- Produces: Modified provisioners with structural approval flow (evaluator injected, `hasApproval()` check present, never triggers)

- [ ] **Step 1: Write and implement ComplianceApprovalEvaluator**

Stub evaluator: `evaluate()` always returns `new ApprovalDecision.AutoApproved()`. Test verifies it returns AutoApproved for a `ComplianceControlSpec` node.

- [ ] **Step 2: Write and implement IoTApprovalEvaluator**

Same pattern. Test verifies AutoApproved for both `PhysicalDeviceSpec` and `DeviceConfigSpec` nodes.

- [ ] **Step 3: Modify ComplianceNodeProvisioner**

Add `ApprovalEvaluator` and `PlanStore` to constructor. Add the standard approval flow (evaluate → if RequiresApproval store plan and return PendingApproval → re-entry with stale-spec guard). Since the evaluator always auto-approves, the flow is structurally present but never activates.

Update `ComplianceNodeProvisionerTest` with new constructor parameters (pass the stub evaluator and an InMemoryPlanStore).

- [ ] **Step 4: Modify IoTNodeProvisioner**

Same pattern as compliance. Update `IoTNodeProvisionerTest`.

- [ ] **Step 5: Run all compliance and IoT tests**

Run: `mvn --batch-mode -f compliance/pom.xml test -q && mvn --batch-mode -f iot/pom.xml test -q`
Expected: ALL PASS

- [ ] **Step 6: Full build verification**

Run: `mvn --batch-mode install -q`
Expected: BUILD SUCCESS

- [ ] **Step 7: Commit**

```
git add compliance/src/ iot/src/
git commit -m "feat(#13): compliance and IoT stub evaluators — approval flow structurally present, auto-approves all"
```

---

### Task 7: Integration test — full approval lifecycle

End-to-end test exercising the complete approval flow through a real provisioner with real evaluator, handler, and plan store.

**Files:**
- Create: `deployment/src/test/java/io/casehub/ops/deployment/ApprovalLifecycleIntegrationTest.java`

**Interfaces:**
- Consumes: All types from Tasks 1–4
- Produces: Integration test covering happy path, auto-approve, rejection, stale-approval, and deprovision flows

- [ ] **Step 1: Write integration test**

```java
// deployment/src/test/java/io/casehub/ops/deployment/ApprovalLifecycleIntegrationTest.java
package io.casehub.ops.deployment;

import io.casehub.desiredstate.api.*;
import io.casehub.desiredstate.runtime.DefaultDesiredStateGraphFactory;
import io.casehub.eidos.api.AgentRegistry;
import io.casehub.ops.api.approval.*;
import io.casehub.ops.api.deployment.*;
import io.casehub.ops.deployment.handler.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

import static org.assertj.core.api.Assertions.assertThat;

class ApprovalLifecycleIntegrationTest {

    private DeploymentNodeProvisioner provisioner;
    private OpsPendingApprovalHandler handler;
    private InMemoryPlanStore planStore;
    private DesiredStateGraph emptyGraph;

    @BeforeEach
    void setUp() {
        planStore = new InMemoryPlanStore();
        handler = new OpsPendingApprovalHandler(planStore);
        var evaluator = new DeploymentApprovalEvaluator();
        // Use same stub registries as DeploymentNodeProvisionerTest
        var agentRegistry = new StubAgentRegistry();
        var providerConfigStore = new DeploymentProviderConfigStore();
        var trustProvider = new DeploymentTrustRoutingPolicyProvider();
        emptyGraph = new DefaultDesiredStateGraphFactory().empty();

        provisioner = new DeploymentNodeProvisioner(
                agentRegistry, providerConfigStore,
                new ChannelProvisionHandler(new StubChannelOperations()),
                new CaseTypeProvisionHandler(),
                new TrustPolicyProvisionHandler(trustProvider),
                new EndpointProvisionHandler(new StubEndpointRegistry()),
                new SpecHashStore(),
                evaluator, planStore);
    }

    @Test
    void happyPath_highRiskNodeRequiresApproval_thenProvisions() {
        var spec = new TrustPolicyNodeSpec("claims-routing", 0.85, 10, 0.1, 0.3, Map.of(), false);
        var node = new DesiredNode(NodeId.of("tp-1"), NodeType.of("trust"), spec, false);
        var context = new ProvisionContext("tenant-1", emptyGraph);

        // Cycle 1: provisioner returns PendingApproval
        var result1 = provisioner.provision(node, context);
        assertThat(result1).isInstanceOf(ProvisionResult.PendingApproval.class);
        var pa = (ProvisionResult.PendingApproval) result1;

        // Simulate executor calling recordPending
        handler.recordPending(node, StepAction.PROVISION, "tenant-1", pa.planReference());

        // Human approves
        handler.approve(NodeId.of("tp-1"), StepAction.PROVISION, "tenant-1", "admin");

        // Cycle 2: executor sees Approved, enriches context
        var check = handler.check(node, StepAction.PROVISION, "tenant-1");
        assertThat(check).isInstanceOf(ApprovalCheckResult.Approved.class);
        var approved = (ApprovalCheckResult.Approved) check;
        var contextWithApproval = context.withApproval(approved.approval());

        // Provisioner re-entry: provisions successfully
        var result2 = provisioner.provision(node, contextWithApproval);
        assertThat(result2).isInstanceOf(ProvisionResult.Success.class);

        // Plan cleaned up
        assertThat(planStore.retrieve(pa.planReference())).isEmpty();
    }

    @Test
    void autoApprove_lowRiskNodeProvisionsDirect() {
        var spec = new ChannelNodeSpec("dev/work", "desc", null,
                null, null, null, null, null, null, null, null, null, null, null);
        var node = new DesiredNode(NodeId.of("ch-1"), NodeType.of("channel"), spec, false);
        var context = new ProvisionContext("tenant-1", emptyGraph);

        var result = provisioner.provision(node, context);
        assertThat(result).isInstanceOf(ProvisionResult.Success.class);
    }

    @Test
    void rejection_cleansUpPlanOnAcknowledge() {
        var spec = new TrustPolicyNodeSpec("claims-routing", 0.85, 10, 0.1, 0.3, Map.of(), false);
        var node = new DesiredNode(NodeId.of("tp-1"), NodeType.of("trust"), spec, false);
        var context = new ProvisionContext("tenant-1", emptyGraph);

        var result = provisioner.provision(node, context);
        var pa = (ProvisionResult.PendingApproval) result;
        handler.recordPending(node, StepAction.PROVISION, "tenant-1", pa.planReference());

        // Reject
        handler.reject(NodeId.of("tp-1"), StepAction.PROVISION, "tenant-1", "too risky");

        // Acknowledge
        handler.acknowledgeRejection(node, StepAction.PROVISION, "tenant-1");

        // State is clean
        assertThat(handler.check(node, StepAction.PROVISION, "tenant-1"))
                .isInstanceOf(ApprovalCheckResult.None.class);
        assertThat(planStore.retrieve(pa.planReference())).isEmpty();
    }

    @Test
    void staleApproval_specChangedSinceApproval_reEvaluates() {
        var spec1 = new TrustPolicyNodeSpec("claims-routing", 0.85, 10, 0.1, 0.3, Map.of(), false);
        var node1 = new DesiredNode(NodeId.of("tp-1"), NodeType.of("trust"), spec1, false);
        var context = new ProvisionContext("tenant-1", emptyGraph);

        // Cycle 1: PendingApproval for spec1
        var result1 = provisioner.provision(node1, context);
        var pa1 = (ProvisionResult.PendingApproval) result1;
        handler.recordPending(node1, StepAction.PROVISION, "tenant-1", pa1.planReference());
        handler.approve(NodeId.of("tp-1"), StepAction.PROVISION, "tenant-1", "admin");

        // Spec changes before re-entry
        var spec2 = new TrustPolicyNodeSpec("claims-routing", 0.95, 10, 0.1, 0.3, Map.of(), false);
        var node2 = new DesiredNode(NodeId.of("tp-1"), NodeType.of("trust"), spec2, false);

        var check = handler.check(node2, StepAction.PROVISION, "tenant-1");
        var contextWithApproval = context.withApproval(((ApprovalCheckResult.Approved) check).approval());

        // Provisioner detects stale spec → returns new PendingApproval
        var result2 = provisioner.provision(node2, contextWithApproval);
        assertThat(result2).isInstanceOf(ProvisionResult.PendingApproval.class);

        // Old plan cleaned up
        assertThat(planStore.retrieve(pa1.planReference())).isEmpty();
    }

    // Stub classes (reuse patterns from DeploymentNodeProvisionerTest)
    // ... StubAgentRegistry, StubChannelOperations, StubEndpointRegistry
    // Copy from DeploymentNodeProvisionerTest — they are inner classes there
}
```

Note: The stub inner classes (`StubAgentRegistry`, `StubChannelOperations`, `StubEndpointRegistry`) should be copied from `DeploymentNodeProvisionerTest` or extracted to a shared test fixture. Extraction to `testing/` is a follow-up cleanup.

- [ ] **Step 2: Run integration test**

Run: `mvn --batch-mode -f deployment/pom.xml test -Dtest=ApprovalLifecycleIntegrationTest -q`
Expected: ALL PASS

- [ ] **Step 3: Full build verification**

Run: `mvn --batch-mode install -q`
Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```
git add deployment/src/test/java/io/casehub/ops/deployment/ApprovalLifecycleIntegrationTest.java
git commit -m "test(#13): approval lifecycle integration test — happy path, auto-approve, rejection, stale-spec"
```
