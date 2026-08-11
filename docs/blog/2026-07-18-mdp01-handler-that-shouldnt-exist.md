---
layout: post
title: "The Handler That Shouldn't Have Existed"
date: 2026-07-18
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ops]
tags: [approval, desiredstate, casehub-work, k8s]
---

I'd been treating the approval workflow as a feature to build. It turned out to be a feature to delete.

`OpsPendingApprovalHandler` sat in ops/api with a `ConcurrentHashMap` tracking pending approvals. It had `approve()` and `reject()` methods, an integration test, the lot. Meanwhile, one module away in `casehub-desiredstate-work`, `WorkItemPendingApprovalHandler` did the same thing — but with persistent WorkItems, callerRef-based idempotent polling, and proper lifecycle management through casehub-work. The ops handler was overriding it via CDI priority. The platform had the answer; the application was ignoring it.

The fix was architectural subtraction. Delete the in-memory handler, add `casehub-desiredstate-work` as a dependency, and the platform's handler activates by classpath presence. No new approval state management code needed.

What I actually built was around it. `K8sApprovalEvaluator` classifies K8s operations by risk — namespace deletion is CRITICAL, deployment deprovision is HIGH, configmap updates are LOW. The threshold sits at HIGH, so anything at that level or above returns `PendingApproval` instead of proceeding. The evaluator is pure classification logic: NodeType times StepAction produces a risk level. No dependencies, no configuration, no CDI wiring complexity.

`KubernetesNodeProvisioner` gained the same approval flow that `DeploymentNodeProvisioner` already had — evaluate risk, store the plan, return `PendingApproval` when needed, handle re-entry with stale-spec detection. The pattern was established; I followed it. The interesting part was where the CDI evaluator needed to live. The single-domain constraint (ARC42STORIES §2) means domain module evaluators activate by classpath presence — only one at a time. Since `K8sApprovalEvaluator` lives in the app module (always loaded), it would have conflicted with domain evaluators if both were `@ApplicationScoped`. But the app doesn't load domain modules — the evaluator is the only `ApprovalEvaluator` on the classpath.

The `ApprovalResource` REST API was the last piece. Four stubbed endpoints became real — list, get, approve, reject. The interesting design choice: approve triggers immediate reconciliation. Without it, the human approves and then waits up to five minutes for the next `resyncInterval()` cycle to notice. We emit a `StateEvent(nodeId, DRIFTED)` through `KubernetesEventSource` after completing the WorkItem, which kicks the reconciliation loop immediately.

The design review caught two things I'd missed. Cross-tenancy access control — `WorkItemService.completeFromSystem()` accepts any UUID and doesn't validate tenancy internally, so the REST endpoint needs to check `workItem.tenancyId` against the caller's tenancy before transitioning. And the actorId source — I'd specified it in both the request header and the request body, contradicting myself. Body won; headers are for infrastructure (tenancyId), body is for application data (who's approving).

The pattern emerging across ops is worth naming: the provisioner evaluates risk, the platform manages the human task lifecycle, and the REST API joins them with domain-specific enrichment. Each layer owns one concern. When something feels like it needs a new abstraction, it usually means the platform already has the answer and the application just needs to get out of the way.
