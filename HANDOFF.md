*Updated: desiredstate#14, claudony#165 closed — removed from backlog.*

# Handoff — casehub-ops

## Last Session
Implemented ProvisionerConfigRegistry (#24) — engine SPI backed by DeploymentProviderConfigStore. Fixed the store's data model (List→Map) to enforce one-config-per-provider-per-agent structurally. Plain @ApplicationScoped displaces NoOpProvisionerConfigRegistry @DefaultBean by classpath presence. Adversarial design review (15 issues, 12 verified, all resolved) corrected the CDI pattern from @Alternative @Priority(1) to @ApplicationScoped and caught 7 other spec improvements. 1 squashed commit on main (dfd8cc7).

## Immediate Next Step
Pick next work. Consumer migration (Claudony → claudony#165, OpenClaw) is the natural follow-on — those repos need to call ProvisionerConfigRegistry.configFor() instead of passing config directly. Or pick from the roadmap. Run `/work` to start.

## Cross-Module
**Blocked by:**
- `casehub-ras` — long-lived situation lifecycle + findAllActive() query (ras#20) blocks end-to-end adaptive ops demo · M · Med

**Enables:**
- `engine#584` — remains open until at least one consumer migrates

## What's Left
- ops#13 PendingApproval provisioner support — filed, unblocked (desiredstate#14 closed) · M · Med
- ops#27 Reverse index for declaredAgentIds — filed, deferred (O(n) acceptable for now) · XS · Low
- ras#20 long-lived situation lifecycle — filed, blocks adaptive ops demo · M · Med
- desiredstate#49 SituationSource SPI — **shipped** (3 commits on desiredstate main, pushed)

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #10 | IoTFaultPolicy domain-specific fault responses | M | Med | Deferred — needs operational feedback |
| #11 | DetectionNodeSpec — RAS situation registration | M | Med | Deferred — requires RAS declarative API |
| #13 | PendingApproval provisioner support | M | Med | Unblocked — desiredstate#14 closed |
| #15 | Deployment declarative topology demo | M | Med | Superseded by #25 adaptive ops demo |
| #16 | Compliance continuous posture demo | M | Med | Unblocked — real EvidenceCollectors |
| #17 | Infra Terraform augmentation demo | M | Med | Unblocked — desiredstate#14 closed |
| #18 | Real EvidenceCollector implementations | M | Med | Feeds #16, unblocked |
| #19 | Integration test hardening | M | Low | Unblocked |
| #20 | Multi-provisioner dispatch readiness | M | Med | Prep for desiredstate#18 |
| #21 | Reconciliation scheduling metadata | S | Low | Prep for desiredstate#19 |
| #22 | Deployment drift remediation strategies | M | High | Needs design |
| #23 | Cross-domain dependency graphs | L | High | Needs design |
| #25 | fsitrading adaptive ops — first consumer | L | High | Blocked by ras#20 |
| #26 | SOC adaptive ops — second consumer | L | High | Blocked by ras#20 |
| #27 | Reverse index for declaredAgentIds | XS | Low | Deferred optimisation |

## References
- Design spec: `specs/issue-24-provisioner-config-registry/2026-06-29-provisioner-config-registry-design.md`
- Architecture: `ARC42STORIES.MD`
- Pause stack: issue-18-real-evidence-collectors (1 paused branch)
