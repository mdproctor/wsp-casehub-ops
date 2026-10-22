---
layout: post
title: "Three Dead Ends and a Bridge"
date: 2026-10-22
entry_type: note
subtype: diary
projects: [casehubio/casehub-ops]
tags: [case-engine, event-driven, judgment, scaffold, approval]
---

# Three Dead Ends and a Bridge

Ops has seven case definitions. Drift remediation, compliance, CVE response, incident, scaling, service upgrade — and an umbrella `application-lifecycle` case that spawns them as sub-cases when context entries change. The choreography pattern works: context change triggers binding, binding launches sub-case, sub-case runs workers, workers write context, case completes.

One problem. Nothing writes those context entries.

## The gap that made everything decorative

`ApplicationEventBroadcaster` publishes domain events — drift detected, compliance violation, CVE discovered. The case system watches for context changes like `.driftDetected` and `.complianceViolation`. These two systems sit side by side, unconnected. Domain events fire into the void. Cases wait for context changes that never arrive.

I'd been aware of this for a while, but it became urgent when scaffold#68 needed to compose ops cases into higher-level workflows. Scaffold can't consume what doesn't activate.

The fix is `CaseContextEventObserver` — a CDI observer that subscribes to the broadcaster on startup and maps six event types to their corresponding context keys. When `PROBLEMS_DRIFT` fires, it signals `.driftDetected` on the umbrella case. The engine's choreography evaluator picks up the context change and fires the drift-remediation sub-case. The bridge is six lines of mapping and a `CaseHubRuntime.signal()` call. The gap was conceptual, not technical.

## Approval was the only workflow that bypassed cases

Every operational workflow in ops routes through the case system — except deployment approval. Approval used a flat `ApprovalEvaluator` → `PendingApprovalHandler` → WorkItem pattern. Fine for simple approve/reject decisions, but high-risk deployments need investigation, evidence collection, precedent lookup, and multi-reviewer escalation. That's exactly what cases provide.

`DeploymentApprovalCaseDescriptor` brings approval under case management. The flow: `evaluate-risk` classifies by action and namespace (DELETE in production is CRITICAL, UPDATE in a dev namespace is LOW), `collect-evidence` gathers deployment context, a `JudgmentTarget` binding presents the evidence to a reviewer, and `record-decision` writes the approval outcome back to case context. The umbrella case gets a new `on-approval-required` binding that spawns this sub-case.

## Escalation was informational — nobody got paged

The third gap was subtler. All existing case descriptors have escalation workers — `escalate-drift`, `escalate-compliance`. They write escalation data to case context. Severity, affected nodes, risk classification. Thorough documentation of a problem that nobody is asked to act on. No WorkItem, no task assignment, no SLA timer.

The fix uses `JudgmentTarget` instead of capability dispatch on the escalation bindings. Drift escalation gets a 4-hour SLA with outcomes `remediate`, `accept-drift`, `escalate-further`. Compliance escalation gets 24 hours with `remediate-manually`, `accept-risk`, `escalate`. The escalation workers still run — they prepare the context that the reviewer sees — but the binding target is now a judgment yield, not a fire-and-forget capability call.

## Why JudgmentTarget and not HumanTaskTarget

`HumanTaskTarget` is deprecated. `JudgmentTarget` is its replacement, and the difference matters. A human task assumes a human will respond. A judgment yield is caller-agnostic — a human, an LLM, a webhook, or an A2A agent can provide the judgment. The engine's `JudgmentScheduler` publishes the request; any caller type can respond.

For ops, this is the right abstraction. Low-risk deployments could be auto-approved by an LLM that reviews the evidence. High-risk deployments route to a human reviewer. The case definition doesn't need to know which — it defines the prompt, the outcomes, the SLA, and the routing config. The scheduler layer handles dispatch.

Claude caught a correctness issue during review: the original `DeploymentApprovalCaseDescriptor` had a binding that targeted a `review-deployment` capability with no worker registered — a silent dispatch failure. The evidence-collection worker now writes `reviewRequired: true` directly to context, which triggers the judgment binding cleanly.

## What this opens up

Scaffold#68 can now build a BFF layer over ops' case lifecycle REST endpoints — list active cases, inspect context, send signals. The umbrella case activates from real events, approval goes through structured review, and escalation actually pages someone.

The larger pattern is that ops' case system is no longer self-contained. It's externally consumable. Scaffold can compose ops cases as sub-cases of higher-level change management workflows. A deployment that touches three services could coordinate three approval sub-cases with a quorum gate. That composition wasn't possible when domain events, approvals, and escalation were all disconnected from the case lifecycle.
