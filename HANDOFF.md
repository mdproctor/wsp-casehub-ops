# Handoff — casehub-ops

## Last Session
Built and shipped adaptive ops — RAS-driven topology adaptation for casehub-ops-deployment. Deep research identified fsitrading and SOC as ideal first consumers. SituationSource SPI in desiredstate-api, AdaptiveTopologyManager in ops-deployment with hysteresis/cooldown/periodic re-poll, full adaptation rule stack (scale/add/update actions). Adversarial design review (12 issues, all resolved). 3 desiredstate commits + 12 ops commits on main. Garden entry GE-20260629-ba6ff8 (cross-repo subagent branch contamination).

## Immediate Next Step
Pick next work. The adaptive ops infrastructure is complete but needs RAS long-lived situations (ras#20) before end-to-end demo. Options: start fsitrading or SOC implementation (both scaffolded), or pick from the roadmap (#18-23). Run `/work` to start.

## Cross-Module
**Blocked by:**
- `casehub-ras` — long-lived situation lifecycle + findAllActive() query (ras#20) blocks end-to-end adaptive ops demo · M · Med
- `casehub-desiredstate` — ReconciliationLoop PendingApproval workflow (desiredstate#14) blocks ops#13 · M · Med

## What's Left
- ops#13 PendingApproval provisioner support — filed, blocked by desiredstate#14 · M · Med
- ras#20 long-lived situation lifecycle — filed, blocks adaptive ops demo · M · Med
- desiredstate#49 SituationSource SPI — **shipped** (3 commits on desiredstate main, pushed)

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #10 | IoTFaultPolicy domain-specific fault responses | M | Med | Deferred — needs operational feedback |
| #11 | DetectionNodeSpec — RAS situation registration | M | Med | Deferred — requires RAS declarative API |
| #13 | PendingApproval provisioner support | M | Med | Blocked by desiredstate#14 |
| #15 | Deployment declarative topology demo | M | Med | Superseded by #25 adaptive ops demo |
| #16 | Compliance continuous posture demo | M | Med | Unblocked — real EvidenceCollectors |
| #17 | Infra Terraform augmentation demo | M | Med | Human gate blocked by desiredstate#14 |
| #18 | Real EvidenceCollector implementations | M | Med | Feeds #16, unblocked |
| #19 | Integration test hardening | M | Low | Unblocked |
| #20 | Multi-provisioner dispatch readiness | M | Med | Prep for desiredstate#18 |
| #21 | Reconciliation scheduling metadata | S | Low | Prep for desiredstate#19 |
| #22 | Deployment drift remediation strategies | M | High | Needs design |
| #23 | Cross-domain dependency graphs | L | High | Needs design |
| #25 | fsitrading adaptive ops — first consumer | L | High | Blocked by ras#20 |
| #26 | SOC adaptive ops — second consumer | L | High | Blocked by ras#20 |

## References
- Design spec: `docs/superpowers/specs/2026-06-29-adaptive-ops-design.md`
- Implementation plan: `docs/superpowers/plans/2026-06-29-adaptive-ops.md`
- Architecture: `ARC42STORIES.MD`
