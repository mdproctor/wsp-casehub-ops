---
layout: post
title: "The Audit Gap — Building Test Infrastructure for Step 4"
date: 2026-10-22
entry_type: note
subtype: diary
projects: [casehubio/casehub-ops]
tags: [testing, audit, desired-state, topology]
series: issue-171-audit-assertion-test-infra
---

# The Audit Gap — Building Test Infrastructure for Step 4

The operational assertions principle has four steps: deploy, exercise, assert outcome, assert audit. Every topology level test in casehub-ops covers the first three. None of them touch step 4. The audit trail — ledger entries, RAS detections, CloudEvents — gets produced in production but never verified in tests.

This is the kind of gap that's invisible until you look for it. The tests are green. The code works. But "works" means "nodes provision correctly." It doesn't mean "the system recorded that it provisioned them." In a platform where tamper-evident audit trails are a first-class concern, that's a significant omission.

I spent the design phase working out where the audit boundary actually is for topology tests. The answer isn't obvious. The production audit trail flows through CDI observers — `@ObservesAsync CloudEvent` handlers that write to the ledger, trigger RAS situation detections, and update work items. Topology tests are pure Java. No CDI container. No Quarkus. The `ReconciliationTestBase` wires up a `SimpleTransitionExecutor` directly, bypassing everything downstream of CloudEvent emission.

The decision review caught something I'd missed: the reconciliation tier doesn't even have a `cloudEventSink`. That's a `ReconciliationLoop` feature, and reconciliation tests don't use `ReconciliationLoop` — they call `SimpleTransitionExecutor` directly. So the original plan of "just use cloudEventSink everywhere" was wrong. The two test tiers have fundamentally different execution pipelines, and the audit capture mechanism had to match.

The solution is a `TransitionExecutor` decorator — `CapturingTransitionExecutor` wraps any executor and records the `TransitionResult` into an `AuditCapture` object. For the reconciliation and live tiers, this captures transition outcomes (which nodes succeeded, which failed). For the integration-live tier, the existing `cloudEventSink` callback feeds into the same `AuditCapture`. One query API, two capture mechanisms aligned to two execution pipelines.

The implementation landed in `testing/` as three classes: `AuditCapture`, `CapturingTransitionExecutor`, and `AuditAssertions`. `ReconciliationTestBase` gained an `audit` field and an `executeTransitionWithAudit()` method. Level authors in #172 and #173 can now write assertions like `AuditAssertions.assertAllSucceeded(audit)` or `AuditAssertions.assertCloudEventForNode(audit, NODE_FAULTED, "my-llm")` — verifying that the pipeline not only did the right thing but recorded doing it.

What this doesn't cover is worth naming: end-to-end ledger persistence and RAS detection are downstream of the CloudEvent boundary. Verifying that CDI observers correctly process CloudEvents into ledger entries requires a CDI-wired application context — that's a service-module integration test concern, not a topology test concern. The topology tests verify signal production; someone still needs to verify signal consumption.
