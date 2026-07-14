# Handoff — casehub-ops

## Last Session
Closed #51 (credential rotation/expiration in K8sClientRegistry) and #35 (scaling-event child case — workers only, trigger deferred). Also replaced DriftConvergenceHandler with generic NodeConvergenceTracker, wired drift convergence (previously unwired). Filed 4 deferred issues (#53–#56). 1 commit on main after squash.

## Immediate Next Step
Pick next work. #25/#26 (adaptive ops consumers) can leverage real-time drift + scaling infrastructure. #16 (compliance demo) and #17 (infra demo) are unblocked. #52 (FaultPolicy tenancyId migration) is XS mechanical work partially done (app module fixed, 4 other modules remain). Run `/work` to start.

## Cross-Module
*None currently.*

## What's Left
- desiredstate#54 requiresHuman gating for deprovision · S · Low
- #52 FaultPolicy/SituationRecompiler tenancyId migration — 4 modules remain (infra, deployment, compliance, iot) · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #52 | FaultPolicy tenancyId migration (remaining modules) | XS | Low | Mechanical — app module done, 4 remain |
| #53 | Scaling bounds — min/max replicas per service | S | Med | Needs ScalingPolicy model |
| #54 | Scaling cooldown period | S | Med | Needs timestamp tracking in parent case |
| #55 | Reactive 401 credential refresh in K8s callers | S | Low | refreshClient() exists — wire callers |
| #56 | Scaling trigger mechanism | M | Med | What writes .scalingRequired/.scalingSpec |
| #43 | Approval workflow for provisioning | M | Med | Phase 3 |
| #45 | K8s-aware FaultPolicy responses | M | Med | Needs operational feedback |
| #25 | fsitrading adaptive ops | L | High | First real consumer — real-time drift + scaling now available |
| #26 | SOC adaptive ops | L | High | Second consumer |
| #16 | Compliance demo | M | Med | Unblocked — case model + real EvidenceCollectors |
| #17 | Infra demo | M | Med | Unblocked |
| #19 | Integration test hardening | M | Low | Unblocked |

## References
- Architecture: `ARC42STORIES.MD`
- Design spec: `docs/specs/2026-07-13-credential-rotation-scaling-event-design.md`
