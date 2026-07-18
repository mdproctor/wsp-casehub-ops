# Handoff — casehub-ops

## Last Session
Implemented #56 (scaling trigger mechanism). REST API + RAS situation-driven scaling triggers write `.scalingRequired` to the application case blackboard. Three new beans: `SituationScalingEvaluator` (confidence-proportional rules, max-wins, cooldown, periodic re-poll), `ScalingSignalBridge` (event→blackboard), `ScalingResource` (REST endpoint). Blackboard path unified. CaseSignaler extracted. HumanGating migration across all 7 modules. Design reviewed adversarially (7 rounds, 20 issues). PR #58 open to upstream. Build green, 284 app tests pass.

## Immediate Next Step
Pick next work. #25/#26 (adaptive ops consumers) can now exercise end-to-end scaling with real situation awareness. #43 (approval workflow) can gate scaling events. Run `/work` to start.

## Cross-Module
- HumanGating enum migration (desiredstate-api snapshot): all 7 modules migrated from `boolean requiresHuman` to `HumanGating` enum. Compliance and IoT GoalCompilers now map domain booleans to `HumanGating.ALL/NONE`.

## What's Left
- desiredstate#54 requiresHuman gating for deprovision · S · Low
- desiredstate: Cdi* bridge classes need no-args constructors upstream (currently worked around via app-level replacements + exclude-types)

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #43 | Approval workflow for provisioning | M | Med | Phase 3 |
| #45 | K8s-aware FaultPolicy responses | M | Med | Needs operational feedback |
| #25 | fsitrading adaptive ops | L | High | First real consumer — all scaling infrastructure now in place |
| #26 | SOC adaptive ops | L | High | Second consumer |
| #16 | Compliance demo | M | Med | Unblocked — case model + real EvidenceCollectors |
| #17 | Infra demo | M | Med | Unblocked |
| #19 | Integration test hardening | M | Low | Unblocked |

## References
- Architecture: `ARC42STORIES.MD`
- Design spec: `docs/specs/2026-07-17-scaling-trigger-mechanism-design.md`
