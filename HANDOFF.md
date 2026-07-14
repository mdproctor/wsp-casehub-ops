# Handoff — casehub-ops

## Last Session
Closed 4 S/XS issues: #52 (FaultPolicy tenancyId migration — 4 domain modules), #55 (reactive 401 credential refresh via K8sClientRegistry.withRetryOn401), #53+#54 (ScalingPolicy with min/max bounds clamping and cooldown enforcement in evaluateScaling worker). Also fixed pre-existing compile errors in deployment and IoT modules. 3 commits on main.

## Immediate Next Step
Pick next work. #25/#26 (adaptive ops consumers) can leverage real-time drift + scaling infrastructure. #16 (compliance demo) and #17 (infra demo) are unblocked. Run `/work` to start.

## Cross-Module
*None currently.*

## What's Left
- desiredstate#54 requiresHuman gating for deprovision · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #56 | Scaling trigger mechanism — what writes scalingRequired/scalingSpec | M | Med | ScalingPolicy now exists to enforce bounds/cooldown |
| #43 | Approval workflow for provisioning | M | Med | Phase 3 |
| #45 | K8s-aware FaultPolicy responses | M | Med | Needs operational feedback |
| #25 | fsitrading adaptive ops | L | High | First real consumer — all infrastructure now in place |
| #26 | SOC adaptive ops | L | High | Second consumer |
| #16 | Compliance demo | M | Med | Unblocked — case model + real EvidenceCollectors |
| #17 | Infra demo | M | Med | Unblocked |
| #19 | Integration test hardening | M | Low | Unblocked |

## References
- Architecture: `ARC42STORIES.MD`
