# Handoff — casehub-ops

## Last Session
Closed #40 (active KubernetesEventSource with fabric8 Watch for real-time drift detection), #41 (InfraBackend.readState/detectDrift API — added InfraNodeSpec param), #46 (Phase 2 minor findings — key parsing, duplicate parseServices, K8s selector, @Scheduled timeouts, redundant predicate). ARC42STORIES.MD synced. 3 commits on main after squash.

## Immediate Next Step
Pick next work. #25/#26 (adaptive ops consumers) can leverage real-time drift infrastructure. #16 (compliance demo) and #17 (infra demo) are unblocked. Run `/work` to start.

## Cross-Module
*None currently.*

## What's Left
- desiredstate#54 requiresHuman gating for deprovision · S · Low
- #51 credential rotation/expiration handling in K8sClientRegistry · S · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #43 | Approval workflow for provisioning | M | Med | Phase 3 |
| #45 | K8s-aware FaultPolicy responses | M | Med | Needs operational feedback |
| #25 | fsitrading adaptive ops | L | High | First real consumer — real-time drift now available |
| #26 | SOC adaptive ops | L | High | Second consumer |
| #16 | Compliance demo | M | Med | Unblocked — case model + real EvidenceCollectors |
| #17 | Infra demo | M | Med | Unblocked |
| #19 | Integration test hardening | M | Low | Unblocked |

## References
- Architecture: `ARC42STORIES.MD`
