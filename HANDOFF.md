# Handoff — casehub-ops

## Last Session
Closed #18 — real EvidenceCollector implementations. Replaced 6 stub collectors with 4 strategy-based implementations (FileExistence, LogDirectory, CertificateExpiry, ConfigHash). SPI change: controlType()→strategy() separates compliance domain from verification method. Safety-net catch in ComplianceEvidenceService. Also fixed ledger API migration (runtime→api), FaultPolicy 2→3 arg and GoalCompiler→CompilationResult across compliance, infra, iot. 344 tests green.

## Immediate Next Step
Pick next work. #16 (compliance demo) is the natural follow-on — real evidence collection is now available. Or #25/#26 (adaptive ops consumers), or quick wins from #40–#46. Run `/work` to start.

## Cross-Module
*None currently.*

## What's Left
- desiredstate#54 requiresHuman gating for deprovision · S · Low
- desiredstate#55 Update dungeon example · XS · Low
- deployment module pre-existing compile failure (ChannelProvisionHandler references unavailable qhorus Channel classes)

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #40 | Active KubernetesEventSource (fabric8 Watch) | M | Med | Real-time drift detection |
| #41 | InfraBackend.readState should accept InfraNodeSpec | S | Low | API fix |
| #42 | FaultPolicy 2→3 arg migration | XS | Low | Partially done (compliance/infra/iot fixed, deployment pending) |
| #43 | Approval workflow for provisioning | M | Med | Phase 3 |
| #44 | Pluggable credential resolver | S | Med | Enhancement |
| #45 | K8s-aware FaultPolicy responses | M | Med | Needs operational feedback |
| #46 | Minor findings batch | S | Low | Code hygiene |
| #25 | fsitrading adaptive ops | L | High | First real consumer |
| #26 | SOC adaptive ops | L | High | Second consumer |
| #16 | Compliance demo | M | Med | Unblocked — real EvidenceCollectors now available |
| #17 | Infra demo | M | Med | Unblocked |
| #19 | Integration test hardening | M | Low | Unblocked |

## References
- Design spec: `docs/superpowers/specs/2026-06-29-real-evidence-collectors-design.md`
- Architecture: `ARC42STORIES.MD`
