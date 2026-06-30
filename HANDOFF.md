*Updated: ADR round 1 implementor response — ops#13 pending approval provisioner design review.*

# Handoff — casehub-ops

## Last Session
ADR design review (round 1 implementor) for ops#13 PendingApproval provisioner spec. Updated spec on branch `issue-13-pending-approval-provisioner` (2c70dbb): removed `requireSecondReviewer`, replaced `specHash` with `originalSpec`, added plan cleanup, pattern matching for PlanDetail, documented module placement and performance implications. Filed ops#32 for approval trigger mechanism.

## Immediate Next Step
Pre-existing build failures in ops-deployment need fixing before merging:
1. `ChannelDriftChecker` — imports reference `io.casehub.qhorus.api.store` which doesn't exist yet (uncommitted change on `issue-13-pending-approval-provisioner` branch)
2. `AgentProvisionHandlerTest` — `AgentCapability` constructor gained a new field

Pick next work or fix the pre-existing failures. Run `/work` to start.

## Cross-Module
**Blocked by:**
- Pre-existing build failures (see above) block full `mvn install`

**Enables:**
- `casehub-ras#22` — this was the ops-side of the cross-repo migration

## What's Left
- ops#13 PendingApproval provisioner support — filed, unblocked (desiredstate#14 closed) · M · Med
- ops#27 Reverse index for declaredAgentIds — filed, deferred (O(n) acceptable for now) · XS · Low
- Pre-existing build failures — ChannelDriftChecker, AgentProvisionHandlerTest

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #10 | IoTFaultPolicy domain-specific fault responses | M | Med | Deferred — needs operational feedback |
| #11 | DetectionNodeSpec — RAS situation registration | M | Med | Deferred — requires RAS declarative API |
| #13 | PendingApproval provisioner support | M | Med | Unblocked — desiredstate#14 closed |
| #15 | Deployment declarative topology demo | M | Med | Superseded by #25 adaptive ops demo |
| #16 | Compliance continuous posture demo | M | Med | Unblocked — real EvidenceCollectors |
| #17 | Infra Terraform augmentation demo | M | Med | Unblocked — desiredstate#14 closed |
| #18 | Real EvidenceCollector implementations | M | Med | Feeds #16, unblocked |
| #19 | Integration test hardening | M | Low | Unblocked |
| #20 | Multi-provisioner dispatch readiness | M | Med | Prep for desiredstate#18 |
| #21 | Reconciliation scheduling metadata | S | Low | Prep for desiredstate#19 |
| #22 | Deployment drift remediation strategies | M | High | Needs design |
| #23 | Cross-domain dependency graphs | L | High | Needs design |
| #25 | fsitrading adaptive ops — first consumer | L | High | ras#22 unblocked |
| #26 | SOC adaptive ops — second consumer | L | High | ras#22 unblocked |
| #27 | Reverse index for declaredAgentIds | XS | Low | Deferred optimisation |

## References
- Design spec: `specs/issue-24-provisioner-config-registry/2026-06-29-provisioner-config-registry-design.md`
- Architecture: `ARC42STORIES.MD`
- Pause stack: issue-18-real-evidence-collectors (1 paused branch)
