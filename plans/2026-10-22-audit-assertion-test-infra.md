# Audit Assertion Test Infrastructure Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #171 — Define audit assertion test infrastructure — ledger/RAS/CloudEvent verification
**Issue group:** #171

**Goal:** Build shared test infrastructure in the `testing/` module that captures transition outcomes and CloudEvents during topology test execution, with query methods and assertion helpers for level authors.

**Architecture:** Three classes in `io.casehub.ops.testing.audit` — `AuditCapture` (unified query API), `CapturingTransitionExecutor` (TransitionExecutor decorator for reconciliation/live tiers), and `AuditAssertions` (static assertion helpers). `ReconciliationTestBase` gains an `AuditCapture` field and a `executeTransitionWithAudit()` helper method. Integration-live tests use `AuditCapture::recordCloudEvent` as the `cloudEventSink` callback.

**Tech Stack:** Java 21, casehub-desiredstate-api (TransitionExecutor, TransitionResult, StepOutcome, NodeId, TransitionPlan), io.cloudevents:cloudevents-api (CloudEvent), AssertJ

## Global Constraints

- All new classes go in `testing/src/main/java/io/casehub/ops/testing/audit/` — testing/ is test-scope only, never compile or runtime
- `AuditCapture` must be thread-safe — use `CopyOnWriteArrayList` (matches existing pattern in FleetObservabilityIntegrationTest)
- No CDI annotations — testing/ is pure Java, no Quarkus dependency
- No comments unless the WHY is non-obvious
- Existing `executeTransition()` in `ReconciliationTestBase` must remain unchanged for backward compatibility

---

## Batch 1: Audit capture infrastructure

### Task 1: AuditCapture + CapturingTransitionExecutor

**Files:**
- Modify: `testing/pom.xml` — add cloudevents-api and assertj-core dependencies
- Create: `testing/src/main/java/io/casehub/ops/testing/audit/AuditCapture.java`
- Create: `testing/src/main/java/io/casehub/ops/testing/audit/CapturingTransitionExecutor.java`
- Test: `testing/src/test/java/io/casehub/ops/testing/audit/AuditCaptureTest.java`
- Test: `testing/src/test/java/io/casehub/ops/testing/audit/CapturingTransitionExecutorTest.java`

**Interfaces:**
- Consumes: `io.casehub.desiredstate.api.TransitionExecutor` (single-method interface: `TransitionResult execute(TransitionPlan, String)`), `io.casehub.desiredstate.api.TransitionResult` (record: `Map<NodeId, StepOutcome> outcomes()`), `io.casehub.desiredstate.api.StepOutcome` (sealed: Succeeded, Failed, AlreadyConverged, Skipped, Rejected), `io.cloudevents.CloudEvent`
- Produces: `AuditCapture` — used by Task 2 (AuditAssertions) and Task 3 (ReconciliationTestBase). Key methods: `transitionResults()`, `allOutcomes()`, `succeededNodes()`, `failedNodes()`, `cloudEvents()`, `cloudEventsOfType(String)`, `cloudEventCount(String)`, `recordTransition(TransitionResult)`, `recordCloudEvent(CloudEvent)`, `clear()`. `CapturingTransitionExecutor` — used by Task 3 (ReconciliationTestBase). Constructor: `CapturingTransitionExecutor(TransitionExecutor delegate, AuditCapture capture)`.

- [ ] **Step 1: Add dependencies to testing/pom.xml**

Add `cloudevents-api` and `assertj-core` as compile-scope dependencies (testing/ is itself test-scope-only, so compile here means available to consumers' test classpath). Add `junit-jupiter` and `casehub-desiredstate` as test-scope dependencies for testing/'s own tests.

```xml
<!-- After existing </dependency> for casehub-platform-api -->
<dependency>
    <groupId>io.cloudevents</groupId>
    <artifactId>cloudevents-api</artifactId>
</dependency>
<dependency>
    <groupId>org.assertj</groupId>
    <artifactId>assertj-core</artifactId>
</dependency>

<!-- test-scope: for testing module's own unit tests -->
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-junit5-internal</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-desiredstate</artifactId>
    <scope>test</scope>
</dependency>
```

- [ ] **Step 2: Verify dependencies resolve**

Run: `mvn --batch-mode -o dependency:resolve -pl testing`
Expected: BUILD SUCCESS (all dependencies available in local repo)

- [ ] **Step 3: Write AuditCapture failing test**

Create `testing/src/test/java/io/casehub/ops/testing/audit/AuditCaptureTest.java`:

```java
package io.casehub.ops.testing.audit;

import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.api.StepOutcome;
import io.casehub.desiredstate.api.TransitionResult;
import io.cloudevents.CloudEvent;
import io.cloudevents.core.builder.CloudEventBuilder;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.net.URI;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class AuditCaptureTest {

    private AuditCapture capture;

    @BeforeEach
    void setUp() {
        capture = new AuditCapture();
    }

    @Test
    void recordsTransitionResult() {
        TransitionResult result = new TransitionResult(Map.of(
                NodeId.of("node-a"), new StepOutcome.Succeeded(),
                NodeId.of("node-b"), new StepOutcome.Succeeded()));
        capture.recordTransition(result);
        assertThat(capture.transitionResults()).hasSize(1);
        assertThat(capture.allOutcomes()).hasSize(2);
    }

    @Test
    void succeededNodesFiltersCorrectly() {
        TransitionResult result = new TransitionResult(Map.of(
                NodeId.of("ok"), new StepOutcome.Succeeded(),
                NodeId.of("fail"), new StepOutcome.Failed("boom")));
        capture.recordTransition(result);
        assertThat(capture.succeededNodes()).containsExactly(NodeId.of("ok"));
        assertThat(capture.failedNodes()).containsExactly(NodeId.of("fail"));
    }

    @Test
    void recordsCloudEvents() {
        CloudEvent event = CloudEventBuilder.v1()
                .withId("1")
                .withSource(URI.create("/test"))
                .withType("io.casehub.desiredstate.reconciliation.completed")
                .build();
        capture.recordCloudEvent(event);
        assertThat(capture.cloudEvents()).hasSize(1);
        assertThat(capture.cloudEventsOfType("io.casehub.desiredstate.reconciliation.completed")).hasSize(1);
        assertThat(capture.cloudEventsOfType("io.casehub.desiredstate.node.faulted")).isEmpty();
    }

    @Test
    void cloudEventCountReturnsCorrectCount() {
        for (int i = 0; i < 3; i++) {
            capture.recordCloudEvent(CloudEventBuilder.v1()
                    .withId(String.valueOf(i))
                    .withSource(URI.create("/test"))
                    .withType("io.casehub.desiredstate.node.faulted")
                    .build());
        }
        assertThat(capture.cloudEventCount("io.casehub.desiredstate.node.faulted")).isEqualTo(3);
    }

    @Test
    void clearResetsAllState() {
        capture.recordTransition(new TransitionResult(Map.of(
                NodeId.of("n"), new StepOutcome.Succeeded())));
        capture.recordCloudEvent(CloudEventBuilder.v1()
                .withId("1").withSource(URI.create("/test"))
                .withType("test").build());
        capture.clear();
        assertThat(capture.transitionResults()).isEmpty();
        assertThat(capture.cloudEvents()).isEmpty();
    }

    @Test
    void accumulatesMultipleTransitions() {
        capture.recordTransition(new TransitionResult(Map.of(
                NodeId.of("a"), new StepOutcome.Succeeded())));
        capture.recordTransition(new TransitionResult(Map.of(
                NodeId.of("b"), new StepOutcome.Failed("err"))));
        assertThat(capture.transitionResults()).hasSize(2);
        assertThat(capture.allOutcomes()).hasSize(2);
        assertThat(capture.succeededNodes()).containsExactly(NodeId.of("a"));
        assertThat(capture.failedNodes()).containsExactly(NodeId.of("b"));
    }
}
```

- [ ] **Step 4: Run test to verify it fails**

Run: `mvn --batch-mode -o test -pl testing -Dtest=AuditCaptureTest`
Expected: COMPILATION ERROR — `AuditCapture` class does not exist

- [ ] **Step 5: Implement AuditCapture**

Create `testing/src/main/java/io/casehub/ops/testing/audit/AuditCapture.java`:

```java
package io.casehub.ops.testing.audit;

import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.api.StepOutcome;
import io.casehub.desiredstate.api.TransitionResult;
import io.cloudevents.CloudEvent;

import java.util.List;
import java.util.Map;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.stream.Collectors;

public class AuditCapture {

    private final List<TransitionResult> transitionResults = new CopyOnWriteArrayList<>();
    private final List<CloudEvent> cloudEvents = new CopyOnWriteArrayList<>();

    public void recordTransition(TransitionResult result) {
        transitionResults.add(result);
    }

    public void recordCloudEvent(CloudEvent event) {
        cloudEvents.add(event);
    }

    public List<TransitionResult> transitionResults() {
        return List.copyOf(transitionResults);
    }

    public Map<NodeId, StepOutcome> allOutcomes() {
        return transitionResults.stream()
                .flatMap(r -> r.outcomes().entrySet().stream())
                .collect(Collectors.toMap(Map.Entry::getKey, Map.Entry::getValue, (a, b) -> b));
    }

    public List<NodeId> succeededNodes() {
        return allOutcomes().entrySet().stream()
                .filter(e -> e.getValue() instanceof StepOutcome.Succeeded)
                .map(Map.Entry::getKey)
                .toList();
    }

    public List<NodeId> failedNodes() {
        return allOutcomes().entrySet().stream()
                .filter(e -> e.getValue() instanceof StepOutcome.Failed)
                .map(Map.Entry::getKey)
                .toList();
    }

    public List<CloudEvent> cloudEvents() {
        return List.copyOf(cloudEvents);
    }

    public List<CloudEvent> cloudEventsOfType(String eventType) {
        return cloudEvents.stream()
                .filter(e -> e.getType().equals(eventType))
                .toList();
    }

    public int cloudEventCount(String eventType) {
        return (int) cloudEvents.stream()
                .filter(e -> e.getType().equals(eventType))
                .count();
    }

    public void clear() {
        transitionResults.clear();
        cloudEvents.clear();
    }
}
```

- [ ] **Step 6: Run AuditCapture tests**

Run: `mvn --batch-mode -o test -pl testing -Dtest=AuditCaptureTest`
Expected: All 6 tests PASS

- [ ] **Step 7: Write CapturingTransitionExecutor failing test**

Create `testing/src/test/java/io/casehub/ops/testing/audit/CapturingTransitionExecutorTest.java`:

```java
package io.casehub.ops.testing.audit;

import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.api.StepOutcome;
import io.casehub.desiredstate.api.TransitionResult;
import io.casehub.desiredstate.runtime.DefaultNodeProvisionerRouter;
import io.casehub.desiredstate.runtime.NoOpHumanNodeHandler;
import io.casehub.desiredstate.runtime.NoOpPendingApprovalHandler;
import io.casehub.desiredstate.runtime.NodeStepExecutor;
import io.casehub.desiredstate.runtime.SimpleTransitionExecutor;
import io.casehub.desiredstate.testing.MockActualStateAdapter;
import io.casehub.desiredstate.testing.MockNodeProvisioner;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class CapturingTransitionExecutorTest {

    @Test
    void delegatesAndCaptures() {
        TransitionResult expected = new TransitionResult(Map.of(
                NodeId.of("n1"), new StepOutcome.Succeeded()));

        AuditCapture capture = new AuditCapture();
        CapturingTransitionExecutor executor = new CapturingTransitionExecutor(
                (plan, tenancyId) -> expected, capture);

        TransitionResult result = executor.execute(null, "test-tenant");

        assertThat(result).isSameAs(expected);
        assertThat(capture.transitionResults()).containsExactly(expected);
    }

    @Test
    void capturesMultipleExecutions() {
        TransitionResult r1 = new TransitionResult(Map.of(
                NodeId.of("a"), new StepOutcome.Succeeded()));
        TransitionResult r2 = new TransitionResult(Map.of(
                NodeId.of("b"), new StepOutcome.Failed("err")));

        AuditCapture capture = new AuditCapture();
        var results = List.of(r1, r2).iterator();
        CapturingTransitionExecutor executor = new CapturingTransitionExecutor(
                (plan, tid) -> results.next(), capture);

        executor.execute(null, "t1");
        executor.execute(null, "t2");

        assertThat(capture.transitionResults()).containsExactly(r1, r2);
        assertThat(capture.succeededNodes()).containsExactly(NodeId.of("a"));
        assertThat(capture.failedNodes()).containsExactly(NodeId.of("b"));
    }
}
```

- [ ] **Step 8: Run test to verify it fails**

Run: `mvn --batch-mode -o test -pl testing -Dtest=CapturingTransitionExecutorTest`
Expected: COMPILATION ERROR — `CapturingTransitionExecutor` does not exist

- [ ] **Step 9: Implement CapturingTransitionExecutor**

Create `testing/src/main/java/io/casehub/ops/testing/audit/CapturingTransitionExecutor.java`:

```java
package io.casehub.ops.testing.audit;

import io.casehub.desiredstate.api.TransitionExecutor;
import io.casehub.desiredstate.api.TransitionPlan;
import io.casehub.desiredstate.api.TransitionResult;

public class CapturingTransitionExecutor implements TransitionExecutor {

    private final TransitionExecutor delegate;
    private final AuditCapture capture;

    public CapturingTransitionExecutor(TransitionExecutor delegate, AuditCapture capture) {
        this.delegate = delegate;
        this.capture = capture;
    }

    @Override
    public TransitionResult execute(TransitionPlan plan, String tenancyId) {
        TransitionResult result = delegate.execute(plan, tenancyId);
        capture.recordTransition(result);
        return result;
    }
}
```

- [ ] **Step 10: Run all Task 1 tests**

Run: `mvn --batch-mode -o test -pl testing`
Expected: All 8 tests PASS

- [ ] **Step 11: Commit**

```bash
git add testing/pom.xml testing/src/
git commit -m "feat(#171): AuditCapture + CapturingTransitionExecutor — audit capture infrastructure

Add AuditCapture for collecting transition outcomes and CloudEvents during
topology test execution. CapturingTransitionExecutor decorates any
TransitionExecutor to record results into AuditCapture.

Refs #171

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 2: AuditAssertions

**Files:**
- Create: `testing/src/main/java/io/casehub/ops/testing/audit/AuditAssertions.java`
- Test: `testing/src/test/java/io/casehub/ops/testing/audit/AuditAssertionsTest.java`

**Interfaces:**
- Consumes: `AuditCapture` (from Task 1) — `succeededNodes()`, `failedNodes()`, `allOutcomes()`, `cloudEventsOfType(String)`, `cloudEventCount(String)`, `cloudEvents()`
- Produces: Static assertion methods used by level authors in #172 and #173: `assertAllSucceeded(AuditCapture)`, `assertNodeSucceeded(AuditCapture, String)`, `assertNodeFailed(AuditCapture, String)`, `assertNoFailures(AuditCapture)`, `assertCloudEventEmitted(AuditCapture, String)`, `assertCloudEventEmitted(AuditCapture, String, int)`, `assertCloudEventForNode(AuditCapture, String, String)`, `assertNoCloudEventsOfType(AuditCapture, String)`

- [ ] **Step 1: Write AuditAssertions failing test**

Create `testing/src/test/java/io/casehub/ops/testing/audit/AuditAssertionsTest.java`:

```java
package io.casehub.ops.testing.audit;

import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.api.StepOutcome;
import io.casehub.desiredstate.api.TransitionResult;
import io.cloudevents.core.builder.CloudEventBuilder;
import org.junit.jupiter.api.Test;

import java.net.URI;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThatCode;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class AuditAssertionsTest {

    @Test
    void assertAllSucceeded_passes_whenAllSucceeded() {
        AuditCapture capture = new AuditCapture();
        capture.recordTransition(new TransitionResult(Map.of(
                NodeId.of("a"), new StepOutcome.Succeeded(),
                NodeId.of("b"), new StepOutcome.Succeeded())));
        assertThatCode(() -> AuditAssertions.assertAllSucceeded(capture))
                .doesNotThrowAnyException();
    }

    @Test
    void assertAllSucceeded_fails_whenAnyFailed() {
        AuditCapture capture = new AuditCapture();
        capture.recordTransition(new TransitionResult(Map.of(
                NodeId.of("a"), new StepOutcome.Succeeded(),
                NodeId.of("b"), new StepOutcome.Failed("err"))));
        assertThatThrownBy(() -> AuditAssertions.assertAllSucceeded(capture))
                .isInstanceOf(AssertionError.class);
    }

    @Test
    void assertNodeSucceeded_passes_whenNodeSucceeded() {
        AuditCapture capture = new AuditCapture();
        capture.recordTransition(new TransitionResult(Map.of(
                NodeId.of("my-node"), new StepOutcome.Succeeded())));
        assertThatCode(() -> AuditAssertions.assertNodeSucceeded(capture, "my-node"))
                .doesNotThrowAnyException();
    }

    @Test
    void assertNodeSucceeded_fails_whenNodeFailed() {
        AuditCapture capture = new AuditCapture();
        capture.recordTransition(new TransitionResult(Map.of(
                NodeId.of("my-node"), new StepOutcome.Failed("err"))));
        assertThatThrownBy(() -> AuditAssertions.assertNodeSucceeded(capture, "my-node"))
                .isInstanceOf(AssertionError.class);
    }

    @Test
    void assertNodeFailed_passes_whenNodeFailed() {
        AuditCapture capture = new AuditCapture();
        capture.recordTransition(new TransitionResult(Map.of(
                NodeId.of("bad"), new StepOutcome.Failed("err"))));
        assertThatCode(() -> AuditAssertions.assertNodeFailed(capture, "bad"))
                .doesNotThrowAnyException();
    }

    @Test
    void assertNoFailures_passes_whenNoFailures() {
        AuditCapture capture = new AuditCapture();
        capture.recordTransition(new TransitionResult(Map.of(
                NodeId.of("a"), new StepOutcome.Succeeded())));
        assertThatCode(() -> AuditAssertions.assertNoFailures(capture))
                .doesNotThrowAnyException();
    }

    @Test
    void assertNoFailures_fails_whenFailuresExist() {
        AuditCapture capture = new AuditCapture();
        capture.recordTransition(new TransitionResult(Map.of(
                NodeId.of("a"), new StepOutcome.Failed("err"))));
        assertThatThrownBy(() -> AuditAssertions.assertNoFailures(capture))
                .isInstanceOf(AssertionError.class);
    }

    @Test
    void assertCloudEventEmitted_passes_whenEventPresent() {
        AuditCapture capture = new AuditCapture();
        capture.recordCloudEvent(CloudEventBuilder.v1()
                .withId("1").withSource(URI.create("/test"))
                .withType("io.casehub.desiredstate.reconciliation.completed").build());
        assertThatCode(() -> AuditAssertions.assertCloudEventEmitted(
                capture, "io.casehub.desiredstate.reconciliation.completed"))
                .doesNotThrowAnyException();
    }

    @Test
    void assertCloudEventEmitted_fails_whenEventAbsent() {
        AuditCapture capture = new AuditCapture();
        assertThatThrownBy(() -> AuditAssertions.assertCloudEventEmitted(
                capture, "io.casehub.desiredstate.reconciliation.completed"))
                .isInstanceOf(AssertionError.class);
    }

    @Test
    void assertCloudEventEmitted_withMinCount() {
        AuditCapture capture = new AuditCapture();
        capture.recordCloudEvent(CloudEventBuilder.v1()
                .withId("1").withSource(URI.create("/test"))
                .withType("test.type").build());
        capture.recordCloudEvent(CloudEventBuilder.v1()
                .withId("2").withSource(URI.create("/test"))
                .withType("test.type").build());
        assertThatCode(() -> AuditAssertions.assertCloudEventEmitted(capture, "test.type", 2))
                .doesNotThrowAnyException();
        assertThatThrownBy(() -> AuditAssertions.assertCloudEventEmitted(capture, "test.type", 3))
                .isInstanceOf(AssertionError.class);
    }

    @Test
    void assertNoCloudEventsOfType_passes_whenNonePresent() {
        AuditCapture capture = new AuditCapture();
        assertThatCode(() -> AuditAssertions.assertNoCloudEventsOfType(
                capture, "io.casehub.desiredstate.node.faulted"))
                .doesNotThrowAnyException();
    }

    @Test
    void assertNoCloudEventsOfType_fails_whenPresent() {
        AuditCapture capture = new AuditCapture();
        capture.recordCloudEvent(CloudEventBuilder.v1()
                .withId("1").withSource(URI.create("/test"))
                .withType("io.casehub.desiredstate.node.faulted").build());
        assertThatThrownBy(() -> AuditAssertions.assertNoCloudEventsOfType(
                capture, "io.casehub.desiredstate.node.faulted"))
                .isInstanceOf(AssertionError.class);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode -o test -pl testing -Dtest=AuditAssertionsTest`
Expected: COMPILATION ERROR — `AuditAssertions` does not exist

- [ ] **Step 3: Implement AuditAssertions**

Create `testing/src/main/java/io/casehub/ops/testing/audit/AuditAssertions.java`:

```java
package io.casehub.ops.testing.audit;

import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.api.StepOutcome;

import static org.assertj.core.api.Assertions.assertThat;

public final class AuditAssertions {

    private AuditAssertions() {}

    public static void assertAllSucceeded(AuditCapture capture) {
        assertThat(capture.allOutcomes().values())
                .as("All transition outcomes should be Succeeded")
                .allMatch(o -> o instanceof StepOutcome.Succeeded);
    }

    public static void assertNodeSucceeded(AuditCapture capture, String nodeId) {
        StepOutcome outcome = capture.allOutcomes().get(NodeId.of(nodeId));
        assertThat(outcome)
                .as("Outcome for node '%s'", nodeId)
                .isNotNull()
                .isInstanceOf(StepOutcome.Succeeded.class);
    }

    public static void assertNodeFailed(AuditCapture capture, String nodeId) {
        StepOutcome outcome = capture.allOutcomes().get(NodeId.of(nodeId));
        assertThat(outcome)
                .as("Outcome for node '%s'", nodeId)
                .isNotNull()
                .isInstanceOf(StepOutcome.Failed.class);
    }

    public static void assertNoFailures(AuditCapture capture) {
        assertThat(capture.failedNodes())
                .as("No nodes should have failed")
                .isEmpty();
    }

    public static void assertCloudEventEmitted(AuditCapture capture, String eventType) {
        assertThat(capture.cloudEventsOfType(eventType))
                .as("Expected at least one CloudEvent of type '%s'", eventType)
                .isNotEmpty();
    }

    public static void assertCloudEventEmitted(AuditCapture capture, String eventType, int minCount) {
        assertThat(capture.cloudEventCount(eventType))
                .as("Expected at least %d CloudEvent(s) of type '%s'", minCount, eventType)
                .isGreaterThanOrEqualTo(minCount);
    }

    public static void assertCloudEventForNode(AuditCapture capture, String eventType, String nodeId) {
        boolean found = capture.cloudEventsOfType(eventType).stream()
                .anyMatch(e -> {
                    if (e.getData() == null) return false;
                    String json = new String(e.getData().toBytes());
                    return json.contains("\"" + nodeId + "\"");
                });
        assertThat(found)
                .as("Expected CloudEvent of type '%s' for node '%s'", eventType, nodeId)
                .isTrue();
    }

    public static void assertNoCloudEventsOfType(AuditCapture capture, String eventType) {
        assertThat(capture.cloudEventsOfType(eventType))
                .as("Expected no CloudEvents of type '%s'", eventType)
                .isEmpty();
    }
}
```

- [ ] **Step 4: Run all testing/ tests**

Run: `mvn --batch-mode -o test -pl testing`
Expected: All tests PASS (AuditCaptureTest + CapturingTransitionExecutorTest + AuditAssertionsTest)

- [ ] **Step 5: Commit**

```bash
git add testing/src/
git commit -m "feat(#171): AuditAssertions — static assertion helpers for audit verification

Provides assertAllSucceeded, assertNodeSucceeded, assertNodeFailed,
assertNoFailures, assertCloudEventEmitted, assertCloudEventForNode, and
assertNoCloudEventsOfType for use by level authors in #172 and #173.

Refs #171

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

## Batch 2: ReconciliationTestBase integration

### Task 3: Wire AuditCapture into ReconciliationTestBase

**Files:**
- Modify: `topology-tests/pom.xml` — add casehub-ops-testing dependency
- Modify: `topology-tests/src/test/java/io/casehub/ops/topology/reconciliation/ReconciliationTestBase.java` — add `AuditCapture` field and `executeTransitionWithAudit()` helper
- Test: `topology-tests/src/test/java/io/casehub/ops/topology/reconciliation/ReconciliationTestBaseVerificationTest.java` — verify audit capture works through the base class

**Interfaces:**
- Consumes: `AuditCapture` (from Task 1), `CapturingTransitionExecutor` (from Task 1), `AuditAssertions` (from Task 2)
- Produces: `ReconciliationTestBase.audit` field (protected), `ReconciliationTestBase.executeTransitionWithAudit(TransitionPlan, String)` method — used by level authors in #172 and #173

- [ ] **Step 1: Add casehub-ops-testing dependency to topology-tests/pom.xml**

```xml
<!-- After existing casehub-desiredstate-testing dependency -->
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-ops-testing</artifactId>
    <scope>test</scope>
</dependency>
```

- [ ] **Step 2: Verify dependency resolves**

Run: `mvn --batch-mode -o dependency:resolve -pl topology-tests`
Expected: BUILD SUCCESS

- [ ] **Step 3: Read current ReconciliationTestBaseVerificationTest**

Read `topology-tests/src/test/java/io/casehub/ops/topology/reconciliation/ReconciliationTestBaseVerificationTest.java` to understand what it already tests. The new audit verification test will extend it.

- [ ] **Step 4: Write failing test for audit capture integration**

Add a new test method to `ReconciliationTestBaseVerificationTest.java` (or create a new test class if the existing one is unrelated):

```java
@Test
void executeTransitionWithAudit_capturesOutcomes() throws IOException {
    DesiredStateGraph graph = compileSingleGraph("topologies/infra/single-service/ghost-blog.yaml");
    TransitionPlan plan = planFromEmpty(graph);
    TransitionResult result = executeTransitionWithAudit(plan, "audit-test");

    assertThat(result.outcomes().values())
            .allMatch(o -> o instanceof StepOutcome.Succeeded);

    AuditAssertions.assertAllSucceeded(audit);
    AuditAssertions.assertNoFailures(audit);

    assertThat(audit.transitionResults()).hasSize(1);
    assertThat(audit.succeededNodes()).isNotEmpty();
}
```

Imports to add:

```java
import io.casehub.ops.testing.audit.AuditAssertions;
```

- [ ] **Step 5: Run test to verify it fails**

Run: `mvn --batch-mode -o test -pl topology-tests -Dtest=ReconciliationTestBaseVerificationTest#executeTransitionWithAudit_capturesOutcomes`
Expected: COMPILATION ERROR — `executeTransitionWithAudit` method does not exist, `audit` field does not exist

- [ ] **Step 6: Add AuditCapture field and executeTransitionWithAudit to ReconciliationTestBase**

Add to `ReconciliationTestBase.java`:

```java
import io.casehub.ops.testing.audit.AuditCapture;
import io.casehub.ops.testing.audit.CapturingTransitionExecutor;
```

Add field:

```java
protected AuditCapture audit;
```

Add initialization in `setUpReconciliation()`:

```java
audit = new AuditCapture();
```

Add method after `executeTransition()`:

```java
protected TransitionResult executeTransitionWithAudit(TransitionPlan plan, String tenancyId) {
    NodeStepExecutor nodeStepExecutor = new NodeStepExecutor(
            new DefaultNodeProvisionerRouter(List.of(provisioner)),
            new NoOpHumanNodeHandler(),
            new NoOpPendingApprovalHandler(),
            (step, tid) -> new StepOutcome.Succeeded());
    SimpleTransitionExecutor delegate = new SimpleTransitionExecutor(nodeStepExecutor);
    CapturingTransitionExecutor executor = new CapturingTransitionExecutor(delegate, audit);
    return executor.execute(plan, tenancyId);
}
```

- [ ] **Step 7: Run the verification test**

Run: `mvn --batch-mode -o test -pl topology-tests -Dtest=ReconciliationTestBaseVerificationTest#executeTransitionWithAudit_capturesOutcomes`
Expected: PASS

- [ ] **Step 8: Run all reconciliation tests to verify no regressions**

Run: `mvn --batch-mode -o test -pl topology-tests -Preconciliation`
Expected: All existing reconciliation tests PASS (they use `executeTransition()` which is unchanged)

- [ ] **Step 9: Run full build**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS across all modules

- [ ] **Step 10: Commit**

```bash
git add topology-tests/pom.xml topology-tests/src/
git commit -m "feat(#171): integrate AuditCapture into ReconciliationTestBase

Add audit field and executeTransitionWithAudit() to ReconciliationTestBase.
Level authors can now capture transition outcomes for audit assertions.
Existing executeTransition() remains unchanged for backward compatibility.

Refs #171

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

## References

- [2026-10-22-audit-assertion-test-infra-design.md] — design spec this plan implements
- [topology-tests/.../reconciliation/ReconciliationTestBase.java] — base class being extended
- [topology-tests/.../integration/FleetObservabilityIntegrationTest.java] — existing CloudEvent capture pattern
- [testing/src/main/java/io/casehub/ops/testing/] — existing stub implementations
- [testing/pom.xml] — testing module dependencies
- [topology-tests/pom.xml] — topology-tests module dependencies
- [GitHub #171] — focal issue
- [GitHub #163] — parent epic
- [GitHub #172] — downstream: add audit assertions to L05-L08
- [GitHub #173] — downstream: add audit assertions to L01-L04
