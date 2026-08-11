---
layout: post
title: "Two Mechanisms, Not One"
date: 2026-07-01
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ops]
tags: [approval, desiredstate, provisioner, architecture]
series: issue-13-pending-approval-provisioner
---

I started this branch thinking issue #13 was about making provisioners return `PendingApproval`. The issue body described a "HumanNodeProvisioner" that returns `PendingApproval` with a plan artifact, gets called back after approval. Straightforward.

Then I read the actual runtime code.

`SimpleTransitionExecutor` has two completely separate approval paths. When `DesiredNode.requiresHuman()` is true, the executor delegates to `HumanNodeHandler` — the provisioner is never called. When the provisioner returns `PendingApproval`, a different mechanism kicks in: `PendingApprovalHandler` records the pending state, and on the next reconciliation cycle the executor checks whether approval has been granted before re-calling the provisioner.

The issue body conflated these. "HumanNodeProvisioner must return PendingApproval" doesn't make sense — human nodes bypass the provisioner entirely. What ops actually needed was the second path: provisioners that evaluate risk dynamically and request approval when a change exceeds a threshold.

Once that was clear, the design fell into place. Shared approval types in `ops-api` — `RiskClassification`, `ApprovalThresholds`, `ApprovalPlan`, `ApprovalEvaluator` — with each domain providing its own evaluator. The infra domain already had the infrastructure for this: `ProvisionPhase.PLAN/APPLY`, `InfraBackend.plan()`, `RiskThresholds`. All designed for a plan/apply lifecycle that was never wired up. This branch connects the wiring.

The stale-approval guard was the most interesting design problem. Between the moment a human approves a plan and the next reconciliation cycle, the desired spec might have changed. If the provisioner blindly applies the old approval to a new spec, it authorises a change nobody reviewed. The design review pushed the solution from `int specHash` (hashCode-based, collision-prone) to storing the full `NodeSpec` as `originalSpec` and comparing via `equals()`. Records give you field-by-field structural equality for free — collision-free.

The `ConcurrentHashMap` race was a good catch. The handler's `approve()` and `reject()` methods used a get-check-put pattern that looked thread-safe because `ConcurrentHashMap` is "thread-safe." Individual operations are atomic; compound sequences are not. Two concurrent `approve()` calls could both read PENDING and both transition. The fix — `ConcurrentHashMap.compute()` with a boolean array to capture the transition result out of the lambda — is standard Java concurrency, but easy to miss when the surrounding code already uses a concurrent collection.

The IoT domain exposed a modelling defect worth noting. `IoTNodeProvisioner` returns `Failed("physical devices cannot be auto-provisioned")` for `PhysicalDeviceSpec` — but the node isn't failing, it needs a human. `Failed` triggers fault feedback every reconciliation cycle, mutating the desired-state graph repeatedly for a node that's simply waiting for physical installation. The correct mechanism is `requiresHuman=true` so the executor returns `Skipped`. Filed as #28 — a separate concern from #13 but discovered through the same first-principles analysis.

All four domain provisioners now have the approval flow structurally present. Deployment evaluates by spec type — only `TrustPolicyNodeSpec` (HIGH risk) triggers approval with default thresholds. Infra delegates to `InfraBackend.plan()` for real risk assessment. Compliance and IoT have stub evaluators that auto-approve everything — the flow is present, the policy is future work. The approval trigger mechanism (WorkItem integration, REST endpoints) is filed as #32.
