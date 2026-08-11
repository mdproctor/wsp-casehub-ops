---
layout: post
title: "Code Review Findings Decay: 3 of 5 Already Done"
date: 2026-06-14
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ops]
tags: [infra, code-review]
---

Filed issue #5 against the infra PoC as five code review findings. By the time I got to them — one session later — three were already implemented.

M3 (reject `terraform_workspace` with `standalone` backend) had both the validation and the test. M5 (`@Inject` on CDI constructors) was present on all three classes. M2 (validation asymmetry) was noted as correct in the issue itself. M1 (package structure) was a sealed-interface constraint, not actionable.

That left M4: `InMemoryResourceProvisioner` and `StandaloneBackend` both maintaining a `ConcurrentHashMap<NodeId, ResourceState>`. The provisioner was storing state it had no business owning — state management is the backend's job. The provisioner's role is to execute tasks and return outcomes. Removed the map, the `getState()` convenience method, and the store/remove calls from `execute()`. Net: -29 lines.

The broader point is about code review finding half-life. If the findings are filed as a batch against a moving codebase, some will be stale by the time someone picks them up. The fix is either to address findings inline during the review, or to re-verify each one before acting on it — which is what happened here, just unintentionally.
