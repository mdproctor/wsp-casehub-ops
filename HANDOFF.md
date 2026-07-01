# Handoff — casehub-ops

## Last Session
Closed #27 (reverse index) and #28 (requiresHuman unification). Unified four organic `requiresHuman` approaches into `NodeSpec.requiresHuman()` default method + `DesiredNode` OR composition. Cross-repo change to casehub-desiredstate (e90d503). Fixed IoT deprovision infinite fault cycle. Added reverse index to `DeploymentProviderConfigStore`. Also landed #33 (handledTypes/resyncInterval) on main before branching. Adversarial design review (10 rounds, $25.68). 2 garden entries (lifecycle gate asymmetry gotcha, record accessor technique). Filed desiredstate#54 (deprovision gap), desiredstate#55 (dungeon example), parent#337 (PLATFORM.md sync).

## Immediate Next Step
Pick next work. #32 (approval trigger — WorkItem/REST for approve/reject) is the natural follow-on. Or pick from the roadmap. Run `/work` to start.

## Cross-Module
**Blocked by:**
- `casehub-ras` — long-lived situation lifecycle + findAllActive() query (ras#20) blocks end-to-end adaptive ops demo · M · Med

**Enables:**
- `engine#584` — remains open until at least one consumer migrates

## What's Left
- ops#32 Approval trigger mechanism — wires WorkItem/REST to approve()/reject() · M · Med
- desiredstate#54 Add requiresHuman gating to executeDeprovision + onDeprovision() · S · Low
- desiredstate#55 Update dungeon example to use spec-level requiresHuman() · XS · Low
- parent#337 Sync PLATFORM.md for NodeSpec.requiresHuman() and reverse index · XS · Low
- ras#20 long-lived situation lifecycle — blocks adaptive ops demo · M · Med
- Pause stack: issue-18-real-evidence-collectors (1 paused branch)

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #32 | Approval trigger mechanism | M | Med | WorkItem/REST integration for approve()/reject() |
| #10 | IoTFaultPolicy domain-specific fault responses | M | Med | Deferred — needs operational feedback |
| #11 | DetectionNodeSpec — RAS situation registration | M | Med | Blocked by RAS declarative API |
| #16 | Compliance continuous posture demo | M | Med | Unblocked |
| #17 | Infra Terraform augmentation demo | M | Med | Unblocked |
| #18 | Real EvidenceCollector implementations | M | Med | Paused branch exists |
| #19 | Integration test hardening | M | Low | Unblocked |
| #25 | fsitrading adaptive ops | L | High | Blocked by ras#20 |
| #26 | SOC adaptive ops | L | High | Blocked by ras#20 |

## References
- Design spec: `docs/specs/issue-27-reverse-index-requires-human/2026-07-01-reverse-index-requires-human-design.md`
- Architecture: `ARC42STORIES.MD`
- Pause stack: issue-18-real-evidence-collectors (1 paused branch)
