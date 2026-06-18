# Handoff — casehub-ops

## Last Session
Closed #7 (deployment app-level topology). Extended the deployment module with provider-specific agent config (`ProviderConfig` + `DeploymentProviderConfigStore`), case definition file loading (`DefinitionPayloadLoader` with classpath-first resolution), and layered drift detection (`NodeDriftChecker` SPI with 4 default implementations + `SpecHashStore`). Multi-file YAML support via `DeploymentGoalLoader`. 87 deployment tests, all green. Six companion issues filed across the ecosystem.

## Immediate Next Step
Pick next domain module. Run `/work` to start. Open issues: #3 (compliance), #4 (IoT). Also: casehubio/casehub-desiredstate#38 (TransitionPlanner DRIFTED fix) is prerequisite for drift self-healing — one-line change, could land quickly.

## Cross-Module
**Blocked by:**
- `casehub-desiredstate` — TransitionPlanner has no DRIFTED code path (#38); drift detection provides OTel observability but no self-healing until fixed · S · Low

**We're blocking:**
- `claudony` — can now read agent provider config from `DeploymentProviderConfigStore` (claudony#156) · M · Med
- `casehub-openclaw` — same (openclaw#36) · S · Low
- `casehub-eidos` — optional bridge module for rich agent drift detection (eidos#60) · M · Med
- `casehub-qhorus` — optional bridge module for rich channel drift detection (qhorus#287) · M · Med

## What's Left
- desiredstate#38 TransitionPlanner DRIFTED fix — filed, one-line change · S · Low
- PLATFORM.md sync (parent) — capability ownership updated, cross-repo deps added · S · Low (done)

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #3 | Epic 2: Compliance posture domain | XL | High | Largest market gap, GRC ~$50B |
| #4 | Epic 3: IoT desired state domain | L | Med | casehub-iot foundation exists |

## References
- Spec: `docs/superpowers/specs/2026-06-17-deployment-app-level-topology-design.md`
- Plan: `docs/superpowers/plans/2026-06-17-deployment-app-level-topology.md`
- Blog: `blog/2026-06-17-mdp01-deployment-app-level-topology.md`
- Companion issues: desiredstate#38, eidos#60, qhorus#287, engine#525, claudony#156, openclaw#36
