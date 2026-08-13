# Handoff — casehub-ops

## Last Session
Implemented compliance-remediation child case (#37) — fourth child case type with real workers (after incident-response #34, drift-remediation, scaling-event). Four-phase worker chain: assess → remediate → verify → escalate. Limited auto-fix for LOG_RETENTION and ENCRYPTION_AT_REST via new `ApplicationLifecycleService.updateServiceConfig()`; all other control types escalate. No compliance module dependency — signal-only architecture using case blackboard as integration layer. Also advanced epic #29 from batch 2 (#31) to batch 3 (#37), fixed upstream `CaseDefinitionRegistry` API change (Uni→CaseMetaModel), updated contributor guide. Landed as 12a53bc on main, issue closed.

Epic #29 progress: 4/6 issues done (batches 1-3 complete). Remaining: batch 4 (#38 stubbed UI screens), batch 5 (#47 wire RAS health monitoring, blocked by casehub-ras#6).

## Immediate Next Step
Run `/work` in slot 86 to advance to #38 (stubbed UI screens) or `/work end` to close the slot at the batch 3 boundary.

## What's Left
- Pre-existing: @QuarkusTest + H2 + Hibernate 6.6 JOINED inheritance DDL failure (GE-20260718-d18dc0) — handled via @Disabled, needs TestContainers migration
- Pre-existing: KubernetesFaultPolicyTest — 3 failures from upstream CaseDefinitionRegistry API change (Uni→CaseMetaModel)
- GitHub Packages token expired — `mvn install` fails with 401 Unauthorized. Refresh token before next session.
- `epic_manager.py advance` bug — doesn't skip already-done issues. Needs fix in soredium.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #38 | Stubbed UI screens — full implementation | M | Med | Batch 4 of epic #29 |
| #47 | Wire RAS health monitoring | M | Med | Batch 5 — blocked by casehub-ras#6 |
| #25 | fsitrading adaptive ops | L | High | First real consumer |
| #26 | SOC adaptive ops | L | High | Second consumer |
| #16 | Compliance demo | M | Med | Unblocked |

## References
- Architecture: `ARC42STORIES.MD`
- Spec: `docs/specs/issue-29-service-lifecycle/2026-08-12-compliance-remediation-child-case-design.md`
- Diary: `docs/blog/2026-08-12-mdp01-case-that-doesnt-call-anyone.md`
- Garden: GE-20260718-d18dc0 (H2+Hibernate DDL)
