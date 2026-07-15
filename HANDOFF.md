# Handoff — casehub-ops

## Last Session
Fixed CI (#57): CDI wiring for 3 new desiredstate SPI beans — SituationRecompilerEngine (List producer + stub), CdiActualStateAdapterRouter and CdiMergedEventSource (app-level replacements with no-args constructors, upstream Cdi* excluded). Build green, 249 tests pass.

## Immediate Next Step
Pick next work. #43 (approval workflow) is still open. #25/#26 (adaptive ops consumers) can leverage real-time drift + scaling infrastructure. #16 (compliance demo) and #17 (infra demo) are unblocked. Run `/work` to start.

## Cross-Module
*None currently.*

## What's Left
- desiredstate#54 requiresHuman gating for deprovision · S · Low
- desiredstate: Cdi* bridge classes need no-args constructors upstream (currently worked around via app-level replacements + exclude-types)

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
