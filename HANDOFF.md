# Handoff — casehub-ops

## Last Session
Implemented DetectionNodeSpec (#11) — 6th deployment node type for RAS situation registration. Full SPI quad: spec record in ops-api, GoalCompiler wiring, DetectionProvisionHandler (via SituationRegistrar), DetectionDriftChecker. Cross-repo: extracted SituationRegistrar interface to casehub-ras-api. Also created epic slot 86 for #29 (service lifecycle), removed dead casehub-engine-blackboard dep, submitted 3 garden entries. Landed on main, issue closed.

## Immediate Next Step
Pick up next priority work with `/work`. Epic slot 86 (#29) is ready — open a CLI in `/Users/mdproctor/claude/casehub/slots/86/ops` and run `/work`.

## Cross-Module
**Enabled** (we delivered, downstream unblocked):
- `casehub-ras` — SituationRegistrar interface extracted (gates future ops DetectionNodeSpec provisioning in app/) · S · Low

## What's Left
- Pre-existing: @QuarkusTest + H2 + Hibernate 6.6 JOINED inheritance DDL failure (GE-20260718-d18dc0) — handled via @Disabled, needs TestContainers migration

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #29 | Epic: Service lifecycle management | — | — | Slot 86 ready, 5 batches |
| #25 | fsitrading adaptive ops | L | High | First real consumer |
| #26 | SOC adaptive ops | L | High | Second consumer |
| #16 | Compliance demo | M | Med | Unblocked |
| #17 | Infra demo | M | Med | Unblocked — InfraBackend SPI blocking |
| #19 | Integration test hardening | M | Low | Unblocked |

## References
- Architecture: `ARC42STORIES.MD`
- Garden: GE-20260718-d18dc0 (H2+Hibernate DDL), GE-20260806-272a90 (deployment node type pattern)
- Spec: `docs/specs/issue-11-detection-node-spec/2026-08-05-detection-node-spec-design.md`
