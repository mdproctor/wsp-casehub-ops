---
layout: post
title: "Proving the System Did Something"
date: 2026-10-29
entry_type: note
subtype: diary
projects: [casehubio/casehub-ops]
tags: [testing, desired-state, accountability, cloudevents, assertions]
series: issue-213-use-case-state-assertions
---

# Proving the System Did Something

There's a testing failure mode I've been staring at for weeks: tests that pass because the system responded, not because the system *worked*. The topology tests in casehub-ops deploy agent fleets across LOCAL, K8S, and PODMAN environments, exercise the LLM agents, and then check that the responses look plausible. Keyword matching, response length, content not blank. The test passes. But what did the system actually *do*?

The question sounds pedantic until you think about what desired-state management is for. A reconciliation loop converges a graph of nodes — provisioning, deprovisioning, recovering from faults. Every action is supposed to produce an accountability record: who did what, when, to which node, and whether it succeeded. If the test doesn't verify that record, it verifies the engine ran without verifying the engine is trustworthy.

## What "the system works" actually means

I'd been thinking about this at the wrong level. The structural assertions — `assertResponseWellFormed`, `assertRouteConsistent` — check that the fleet exerciser got a response with the right shape. That's necessary but not sufficient. It's like checking that a cash register prints a receipt without checking whether the receipt matches what was sold.

For the accountability level (Level 02 in the topology test hierarchy), "the system works" means three specific things:

**Completeness.** Every node in the desired-state graph must have a corresponding audit outcome. If the graph has six nodes and the audit trail has five entries, something was silently skipped. The reconciliation loop ran, but one node went unrecorded — which means someone looking at the audit trail would believe only five things happened. That's not an accountability system, it's a liability.

**Attribution.** Every CloudEvent in the audit trail must carry a source, a type, a timestamp, and the tenancy it belongs to. These are the accountability fields — they answer "who initiated this action, what kind of action was it, when did it happen, and for which tenant." A CloudEvent without a timestamp is an event that can't be sequenced. A CloudEvent without a tenancy ID is an event that can't be attributed to any customer's data.

**Integrity.** The reconciliation summary — the `RECONCILIATION_COMPLETED` CloudEvent — must be internally consistent. Its `nodeCount` must match the graph. Its `faultCount` must be zero for a clean deployment. If the summary says six nodes and the graph has seven, the summary is lying. If the summary reports zero faults but a fault occurred, the system is concealing failures.

## Three assertions, not three categories

We built these as three concrete methods in `AuditAssertions`:

```java
AuditAssertions.assertAuditTrailCoversGraph(auditCapture, graph);
AuditAssertions.assertCloudEventsHaveAttribution(auditCapture, tenancyId);
AuditAssertions.assertReconciliationSummaryConsistent(auditCapture, graph);
```

Each takes the `AuditCapture` (the test's CloudEvent recorder) and the `DesiredStateGraph` (the topology being deployed). The graph is the source of truth — the assertion compares what the audit trail recorded against what was supposed to happen.

`assertAuditTrailCoversGraph` extracts every `NodeId` from the graph, then checks that `auditCapture.allOutcomes()` has a corresponding `StepOutcome` for each one. If any graph node is missing from the audit, the assertion fails with a message naming the missing nodes. The difference from `assertAllSucceeded` is significant: `assertAllSucceeded` checks that recorded outcomes are all successes. It doesn't notice if a node was never recorded at all.

`assertCloudEventsHaveAttribution` iterates every CloudEvent in the capture. For each: source must be non-null, type must be non-blank, time must be present. If the event carries a `tenancyid` extension, it must match the expected tenant. The `tenancyid` check is deliberately lenient — some events (like `nodeDriftExempted`) don't carry tenancy. The assertion catches mismatched tenancy, not missing tenancy.

`assertReconciliationSummaryConsistent` finds the latest `RECONCILIATION_COMPLETED` event, parses its JSON data payload, and checks `nodeCount` against `graph.nodes().size()` and `faultCount` against zero. I chose string-level JSON checks rather than deserialising `ReconciliationCompletedData` — the test module shouldn't need a Jackson dependency on the runtime's data records, and the JSON structure is stable enough that `contains("\"nodeCount\":6")` is both readable and reliable.

## Where this landed

The three assertions were applied to every topology test that has audit infrastructure — 14 test classes covering levels 01 through 14. The pattern is the same everywhere: add the three calls immediately after the initial `assertAllSucceeded`, before any fault injection, scaling, or mutation that changes the graph state. This is important — after a fault injection, the graph's expected state changes, so `assertReconciliationSummaryConsistent` would fail for the wrong reason.

The levels that already had strong use-case assertions (08-fleet-observability, 10-pool-deployment, 12-ad-hoc-deployer, 14-fault-recovery) gained the accountability backbone on top of what they already verify. Level 08 already proves that faults are detected and recovered. Now it also proves that the pre-fault state was fully recorded with correct attribution. Level 10 already proves that pool scaling changes are reflected in the fleet API. Now it also proves the reconciliation summary is honest.

Levels 02 and 03 got the most transformation. The accountability acceptance test used to have a verbose per-node loop checking each `StepOutcome` individually and then asserting the succeeded-nodes count was "greater than or equal to" the graph size. That count-based assertion was actually weaker than it looked — if an extra phantom outcome appeared from a previous run, the test would still pass. `assertAuditTrailCoversGraph` does the right thing: it checks set membership, not magnitude.

## What this doesn't do yet

The elephant in the room: these assertions verify the *system's* effect (the audit trail was complete, attributed, and consistent), not the *agent's* effect (the LLM created a case, dispatched work, triggered a state transition). That second layer requires the HTTP agent invocation endpoint — invoking agents through the deployed stack so their side effects land in the real system state. That infrastructure is still being built (platform#666, eidos#208).

When it arrives, the pattern is already clear. `assertAuditTrailCoversGraph` becomes the foundation — it guarantees the audit trail is complete. On top of that, a level-specific assertion like `assertCaseCreated(ledgerClient, caseId)` or `assertWorkItemDispatched(workClient, expectedAttributes)` verifies the agent's actual effect. The audit backbone holds everywhere; the effect assertions are use-case-specific.

The structural assertions (`assertResponseWellFormed`, `assertRouteConsistent`) remain as the base layer. They verify the fleet exerciser works. The accountability assertions verify the reconciliation loop is trustworthy. The effect assertions — when they arrive — will verify the agents are functional. Three layers, each independent, each testable.

I've been building infrastructure testing for years, and the recurring trap is always the same: the test passes because *something happened*, not because *the right thing happened*. The three accountability assertions are a direct response to that trap. They're small — 60 lines of production code — but they close a gap that would otherwise grow silently as the topology test suite expands.
