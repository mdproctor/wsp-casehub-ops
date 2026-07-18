# Handoff — casehub-ops

## Last Session
Implemented #43 (approval workflow for provisioning operations — Phase 3). Deleted in-memory OpsPendingApprovalHandler, activated platform's WorkItemPendingApprovalHandler via casehub-desiredstate-work dependency. K8sApprovalEvaluator classifies risk by NodeType×StepAction (namespace deletion CRITICAL, deployment deprovision HIGH). KubernetesNodeProvisioner gains approval flow with re-entry and stale-spec detection. ApprovalResource REST API with tenancy isolation and immediate reconciliation trigger. Design reviewed adversarially (3 rounds, 12 issues). Build green, 296 app tests pass.

## Immediate Next Step
Pick next work. #25/#26 (adaptive ops consumers) can now exercise end-to-end scaling with approval gates. #16/#17 (demos) are unblocked. Run `/work` to start.

## What's Left
- desiredstate#54 requiresHuman gating for deprovision · S · Low
- desiredstate: Cdi* bridge classes need no-args constructors upstream (currently worked around via app-level replacements + exclude-types)
- Persistent PlanStore (InMemoryPlanStore loses plan data on restart — provisioner re-evaluates gracefully but human re-approves) · S · Med
- Context-aware K8s risk classification (prod namespace = CRITICAL, dev = LOW) · S · Med
- Fine-grained approval authorization (role-based gates for CRITICAL operations) · S · Med
- 11 unstamped closed branches (pre-existing hygiene debt)
- 1 unrecovered blog on closed branch issue-29 (2026-07-06-mdp01-everything-is-a-case.md)

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #25 | fsitrading adaptive ops | L | High | First real consumer — all scaling + approval infrastructure now in place |
| #26 | SOC adaptive ops | L | High | Second consumer |
| #16 | Compliance demo | M | Med | Unblocked — case model + real EvidenceCollectors |
| #17 | Infra demo | M | Med | Unblocked |
| #45 | K8s-aware FaultPolicy responses | M | Med | Needs operational feedback |
| #19 | Integration test hardening | M | Low | Unblocked |

## References
- Architecture: `ARC42STORIES.MD`
- Design spec: `docs/specs/2026-07-18-approval-workflow-design.md`
- Blog: `blog/2026-07-18-mdp01-handler-that-shouldnt-exist.md`
