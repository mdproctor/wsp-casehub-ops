# Handoff — casehub-ops

## Last Session
Closed #36 (drift-remediation child case with full case model infrastructure), #44 (pluggable credential resolver), #42 (FaultPolicy 3-arg — already done). Built Phase 3 foundation: ApplicationCaseDescriptor with 6 bindings, CaseDefinitionRegistrar, DriftRemediationCaseDescriptor with classify/remediate/escalate workers, DriftSignalBridge (NODE_DRIFTED CloudEvent observer), DriftConvergenceHandler (NODE_RECOVERED convergence), K8sResourceHandler.readDiff() refactor, CredentialResolver wired into K8sClientRegistry with per-cluster trustCerts. Design-reviewed (2 rounds, 18 issues all resolved). 166 tests green. Filed #51 (credential rotation deferred).

## Immediate Next Step
Pick next work. #16 (compliance demo) and #17 (infra demo) can now use the case model. #25/#26 (adaptive ops consumers) can leverage drift detection infrastructure. Quick wins from #40–#46 still available. Run `/work` to start.

## Cross-Module
*None currently.*

## What's Left
- desiredstate#54 requiresHuman gating for deprovision · S · Low
- #51 credential rotation/expiration handling in K8sClientRegistry · S · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #40 | Active KubernetesEventSource (fabric8 Watch) | M | Med | Real-time drift detection |
| #41 | InfraBackend.readState should accept InfraNodeSpec | S | Low | API fix |
| #43 | Approval workflow for provisioning | M | Med | Phase 3 |
| #45 | K8s-aware FaultPolicy responses | M | Med | Needs operational feedback |
| #46 | Minor findings batch | S | Low | Code hygiene |
| #25 | fsitrading adaptive ops | L | High | First real consumer |
| #26 | SOC adaptive ops | L | High | Second consumer |
| #16 | Compliance demo | M | Med | Unblocked — case model + real EvidenceCollectors |
| #17 | Infra demo | M | Med | Unblocked |
| #19 | Integration test hardening | M | Low | Unblocked |

## References
- Design spec: `docs/superpowers/specs/2026-07-10-drift-credential-faultpolicy-design.md`
- Architecture: `ARC42STORIES.MD`
