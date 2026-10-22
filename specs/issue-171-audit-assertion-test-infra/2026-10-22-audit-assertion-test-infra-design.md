# Audit Assertion Test Infrastructure Design

**Issue:** casehubio/casehub-ops#171
**Epic:** casehubio/casehub-ops#163 (Add audit trail assertions to all topology levels)
**Date:** 2026-10-22

## Problem

The operational assertions principle defines four steps: deploy, exercise, assert outcome, assert audit. Step 4 is missing everywhere — no topology integration test verifies that operations produce audit records. Before adding audit assertions to levels (#172, #173), we need shared test infrastructure that captures audit-relevant signals and provides query/assertion utilities for level authors.

## Scope

This issue delivers **test infrastructure** — the capture mechanisms, query APIs, and assertion helpers. It does NOT add assertions to individual levels (that's #172 and #173) and does NOT verify end-to-end ledger/RAS/work persistence (that's a service-module concern).

**What "audit" means at the topology test level:**
- **Reconciliation/live tiers:** Transition outcomes — StepOutcome types (Succeeded, Failed, AlreadyConverged) per node, confirming the execution pipeline recognized and handled each node correctly.
- **Integration-live tier:** CloudEvent emission — event types (RECONCILIATION_COMPLETED, NODE_FAULTED, NODE_RECOVERED, NODE_DRIFTED) with correct data fields (nodeId, tenancyId, faultType), confirming the runtime emits the signals that downstream audit consumers (ledger, RAS, work) rely on.

## Architecture

### Components

Three classes in `testing/` module, package `io.casehub.ops.testing.audit`:

1. **`AuditCapture`** — unified query API over captured audit signals
2. **`CapturingTransitionExecutor`** — `TransitionExecutor` decorator for reconciliation/live tiers
3. **`AuditAssertions`** — static assertion helper methods

### AuditCapture

Central test utility. Collects audit signals from two sources and provides a unified query API.

```java
public class AuditCapture {

    private final List<TransitionResult> transitionResults = new CopyOnWriteArrayList<>();
    private final List<CloudEvent> cloudEvents = new CopyOnWriteArrayList<>();

    // -- Capture methods (called by infrastructure, not by tests) --

    void recordTransition(TransitionResult result) { ... }
    void recordCloudEvent(CloudEvent event) { ... }

    // -- Query methods (called by tests) --

    // Transition outcomes (reconciliation/live tiers)
    List<TransitionResult> transitionResults();
    Map<NodeId, StepOutcome> allOutcomes();
    Map<NodeId, StepOutcome> outcomesOfType(Class<? extends StepOutcome> type);
    List<NodeId> succeededNodes();
    List<NodeId> failedNodes();

    // CloudEvents (integration-live tier)
    List<CloudEvent> cloudEvents();
    List<CloudEvent> cloudEventsOfType(String eventType);
    int cloudEventCount(String eventType);

    // Lifecycle
    void clear();
}
```

Thread-safe via `CopyOnWriteArrayList` — matches the pattern already established in `FleetObservabilityIntegrationTest`.

### CapturingTransitionExecutor

Decorator wrapping any `TransitionExecutor`. Records each `TransitionResult` into `AuditCapture`.

```java
public class CapturingTransitionExecutor implements TransitionExecutor {

    private final TransitionExecutor delegate;
    private final AuditCapture capture;

    public CapturingTransitionExecutor(TransitionExecutor delegate, AuditCapture capture) { ... }

    @Override
    public TransitionResult execute(TransitionPlan plan, String tenancyId) {
        TransitionResult result = delegate.execute(plan, tenancyId);
        capture.recordTransition(result);
        return result;
    }
}
```

### AuditAssertions

Static assertion helpers using AssertJ under the hood.

```java
public final class AuditAssertions {

    // Transition outcome assertions
    public static void assertAllSucceeded(AuditCapture capture);
    public static void assertNodeSucceeded(AuditCapture capture, String nodeId);
    public static void assertNodeFailed(AuditCapture capture, String nodeId);
    public static void assertNoFailures(AuditCapture capture);

    // CloudEvent assertions
    public static void assertCloudEventEmitted(AuditCapture capture, String eventType);
    public static void assertCloudEventEmitted(AuditCapture capture, String eventType, int count);
    public static void assertCloudEventForNode(AuditCapture capture, String eventType, String nodeId);
    public static void assertNoCloudEventsOfType(AuditCapture capture, String eventType);
}
```

## Integration Patterns

### Reconciliation tier (ReconciliationTestBase)

Level authors wrap `SimpleTransitionExecutor` with `CapturingTransitionExecutor`:

```java
// In test setup
AuditCapture audit = new AuditCapture();

// In executeTransition (override or helper)
SimpleTransitionExecutor delegate = new SimpleTransitionExecutor(nodeStepExecutor);
CapturingTransitionExecutor executor = new CapturingTransitionExecutor(delegate, audit);
TransitionResult result = executor.execute(plan, tenancyId);

// In assertions
AuditAssertions.assertAllSucceeded(audit);
AuditAssertions.assertNodeSucceeded(audit, "my-llm");
```

`ReconciliationTestBase` gains an `AuditCapture audit` field and an overloaded `executeTransition` that uses the capturing executor. Existing tests continue to work unchanged — the capture is additive.

### Integration-live tier (ReconciliationLoop tests)

Level authors pass `audit::recordCloudEvent` as the cloudEventSink:

```java
// In test setup
AuditCapture audit = new AuditCapture();

ReconciliationLoop loop = ReconciliationLoop.builder(planner, executor, actualRouter, faultEngine, emptySource)
    .cloudEventSink(audit::recordCloudEvent)
    .build();

// In assertions
AuditAssertions.assertCloudEventEmitted(audit, DesiredStateEventTypes.RECONCILIATION_COMPLETED);
AuditAssertions.assertCloudEventForNode(audit, DesiredStateEventTypes.NODE_FAULTED, "my-llm.analyst");
```

This replaces the ad-hoc `CopyOnWriteArrayList<CloudEvent> capturedEvents` + local `eventsOfType()` helper in `FleetObservabilityIntegrationTest`.

### Live tier (K3sTestBase)

Same pattern as reconciliation tier — `CapturingTransitionExecutor` wrapping `SimpleTransitionExecutor`. The live tier constructs its executor the same way.

## Test Fidelity Gradient

| Tier | What's captured | What's asserted | Fidelity |
|------|----------------|-----------------|----------|
| Reconciliation | TransitionResult (Map<NodeId, StepOutcome>) | Outcome types per node — Succeeded, Failed, AlreadyConverged | Signal production — did the pipeline handle each node correctly? |
| Live (K3s) | TransitionResult (same) | Same — outcome types per node | Signal production against real infrastructure |
| Integration-live | CloudEvents via cloudEventSink | Event types + data fields (nodeId, tenancyId, faultType) | Signal emission — are the right events emitted for downstream consumers? |

## Assertion Semantics

Level authors assert:
- **Event types** — RECONCILIATION_COMPLETED, NODE_FAULTED, NODE_RECOVERED confirm the pipeline recognized the condition
- **Key data fields** — nodeId, tenancyId confirm the event applies to the right entity
- **Outcome types** — Succeeded, Failed, AlreadyConverged confirm the pipeline handled the node correctly

Level authors do NOT assert:
- **Event ordering** — controlled by ReconciliationLoop, not the topology
- **Exact payload structure** — brittle, breaks on field additions
- **Exact event counts** — fault recovery may produce variable counts depending on timing

## Module Changes

### testing/ (casehub-ops-testing)

Add package `io.casehub.ops.testing.audit` with three classes:
- `AuditCapture`
- `CapturingTransitionExecutor`
- `AuditAssertions`

New dependencies in testing/pom.xml:
- `io.casehub:casehub-desiredstate-api` (for TransitionExecutor, TransitionResult, StepOutcome, NodeId)
- `io.cloudevents:cloudevents-api` (for CloudEvent)
- `org.assertj:assertj-core` (for assertion helpers)

All test scope — testing/ is already a test-scope-only module.

### topology-tests/

Add `AuditCapture audit` field and `executeTransitionWithAudit()` helper to `ReconciliationTestBase`. Existing `executeTransition()` remains unchanged for backward compatibility.

### Assumptions

- **A1:** The reconciliation tier needs its own distinct capture mechanism (TransitionExecutor decorator) separate from cloudEventSink, because it doesn't use ReconciliationLoop.
- **A2:** The `TransitionExecutor` interface is stable — it's a single-method interface with three implementations (Simple, Parallel, Mock). A decorator is safe to build on.

### Deferred

Service-module audit consumption tests — verifying that CDI interceptors correctly process CloudEvents into ledger entries, RAS events, and work audit records — are a separate concern not tracked by #171–#173. The parent epic's "verify ledger/work/RAS" requires both this infrastructure (signal production) and a future service-module counterpart (signal consumption).

## References

- `topology-tests/.../reconciliation/ReconciliationTestBase.java` — pure Java reconciliation test base, executeTransition pattern
- `topology-tests/.../integration/FleetObservabilityIntegrationTest.java` — L08 CloudEvent capture via cloudEventSink
- `topology-tests/.../integration/ServiceCluster.java` — ServiceCluster test infrastructure
- `topology-tests/.../reconciliation/FailableNodeProvisioner.java` — existing test double pattern (decorator-like)
- `testing/src/main/java/io/casehub/ops/testing/` — 6 existing stub implementations
- `io.casehub.desiredstate.api.TransitionExecutor` — interface being decorated
- `io.casehub.desiredstate.api.TransitionResult` — record with Map<NodeId, StepOutcome>
- `io.casehub.desiredstate.api.DesiredStateEventTypes` — CloudEvent type constants
- `io.casehub.ops.api.lifecycle.OpsCloudEventTypes` — 28 ops-specific CloudEvent types
- casehubio/casehub-ops#163 — parent epic
- casehubio/casehub-ops#172 — add audit assertions to L05-L08
- casehubio/casehub-ops#173 — add audit assertions to L01-L04
