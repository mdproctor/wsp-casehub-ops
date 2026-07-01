# Handoff — casehub-ops

## Last Session
Implemented PendingApproval provisioner support (#13) — shared approval types in `io.casehub.ops.api.approval`, `OpsPendingApprovalHandler` with atomic state machine, domain-specific `ApprovalEvaluator` for all four domains, provisioner re-entry flow with stale-spec guard. Adversarial design review (4 rounds, 16 issues, 12 verified). Subagent-driven implementation (7 tasks, 1 blocker fixed — ConcurrentHashMap race condition). Infra plan/apply lifecycle wired up, resolving longstanding TODO. 1 squashed commit on main (a7cb274). Filed follow-up issues #28 (requiresHuman + HumanNodeHandler) and #32 (approval trigger mechanism).

## Immediate Next Step
Pick next work. #32 (approval trigger — WorkItem/REST integration with approve()/reject()) is the natural follow-on to make the approval mechanism usable end-to-end. Or pick from the roadmap. Run `/work` to start.

## Cross-Module
**Blocked by:**
- `casehub-ras` — long-lived situation lifecycle + findAllActive() query (ras#20) blocks end-to-end adaptive ops demo · M · Med

**Enables:**
- `engine#584` — remains open until at least one consumer migrates

## What's Left
- ops#28 requiresHuman + HumanNodeHandler — filed, fixes IoT modeling defect (PhysicalDeviceSpec returns Failed instead of Skipped) · M · Med
- ops#32 Approval trigger mechanism — filed, wires WorkItem/REST to approve()/reject() · M · Med
- ops#27 Reverse index for declaredAgentIds — filed, deferred (O(n) acceptable for now) · XS · Low
- ras#20 long-lived situation lifecycle — filed, blocks adaptive ops demo · M · Med
- Pause stack: issue-18-real-evidence-collectors (1 paused branch)

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #28 | requiresHuman + HumanNodeHandler | M | Med | Fixes IoT modeling defect, separate approval mechanism |
| #32 | Approval trigger mechanism | M | Med | WorkItem/REST integration for approve()/reject() |
| #10 | IoTFaultPolicy domain-specific fault responses | M | Med | Deferred — needs operational feedback |
| #11 | DetectionNodeSpec — RAS situation registration | M | Med | Deferred — requires RAS declarative API |
| #16 | Compliance continuous posture demo | M | Med | Unblocked — real EvidenceCollectors |
| #17 | Infra Terraform augmentation demo | M | Med | Unblocked |
| #18 | Real EvidenceCollector implementations | M | Med | Feeds #16, paused branch exists |
| #19 | Integration test hardening | M | Low | Unblocked |
| #25 | fsitrading adaptive ops — first consumer | L | High | Blocked by ras#20 |
| #26 | SOC adaptive ops — second consumer | L | High | Blocked by ras#20 |
| #27 | Reverse index for declaredAgentIds | XS | Low | Deferred optimisation |

## References
- Design spec: `specs/issue-13-pending-approval-provisioner/2026-06-30-pending-approval-provisioner-design.md`
- Architecture: `ARC42STORIES.MD`
- Pause stack: issue-18-real-evidence-collectors (1 paused branch)
