# Handoff — casehub-ops

## Last Session
Brainstormed #38 (stubbed UI screens) — scope pivoted: generic UI screens → scaffold epic (future), ops building blocks → blocks-ui, backend → wire everything. Design spec + 7 decisions captured, reviewed (light decision review + light structure review), revised. Implementation plan: 9 tasks across 3 workstreams. Completed Tasks 1-3: CveStore persistence (Flyway V6), updateServiceImage + rollbackToDeployment on ApplicationLifecycleService, CveResponseCaseDescriptor (four-phase CVE remediation, 15 tests).

Epic #29 progress: 4/6 issues done (batches 1-3 complete). Batch 4 (#38) in progress — 3/9 tasks done.

## Immediate Next Step
Run `/work` in slot 86 to continue #38. Task 4 next: ServiceUpgradeCaseDescriptor (same four-phase pattern as CveResponse). Then Tasks 5-9: CveStatusObserver + registrar update, ApplicationEventBroadcaster, ScalingService extraction, REST wiring, blocks-ui components.

## What's Left
- Pre-existing: @QuarkusTest + H2 + Hibernate 6.6 JOINED inheritance DDL failure (GE-20260718-d18dc0) — handled via @Disabled, needs TestContainers migration
- Pre-existing: @QuarkusTest offline config failure — upstream config properties missing in offline mode (GE-20260814-8bc7ef). Non-QuarkusTest unit tests (244) pass clean.
- GitHub Packages token expired — refresh before next session for online builds
- `epic_manager.py advance` bug — doesn't skip already-done issues. Needs fix in soredium.
- blocks-ui needs adding to slot 86 via `work-slot add-repo` before Task 9

## References
- Spec: `specs/issue-38-stubbed-ui-screens/2026-08-14-stubbed-ui-screens-full-implementation-design.md`
- Decisions: `specs/issue-38-stubbed-ui-screens/decisions.md`
- Plan: `plans/2026-08-14-stubbed-ui-screens.md`
- Architecture: `ARC42STORIES.MD`
- Garden: GE-20260814-8bc7ef (QuarkusTest offline), GE-20260814-cb5551 (WorkerResult API), GE-20260814-eee153 (case descriptor test pattern)
