# Handoff — casehub-ops

*Updated: parent#312, platform#117 closed — removed from backlog.*

## Last Session
Closed #12 (agentIds() on DeploymentProviderConfigStore). XS change — single method + 4 tests. ARC42STORIES.MD stale scan removed resolved desiredstate#38 from active risks and #8 from tech debt.

## Immediate Next Step
Pick next work. Remaining open issues are both deferred: #10 (IoTFaultPolicy — needs operational feedback), #11 (DetectionNodeSpec — needs RAS declarative API). Run `/work` to start if new work is identified.

## Cross-Module
**Blocked by:**
- `casehub-desiredstate` — SimpleTransitionExecutor has no WorkItem creation for requiresHuman=true (desiredstate#43) · S · Low

## What's Left
- desiredstate#43 WorkItem creation for requiresHuman — filed, open · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #10 | IoTFaultPolicy domain-specific fault responses | M | Med | Deferred — needs operational feedback |
| #11 | DetectionNodeSpec — RAS situation registration | M | Med | Deferred — requires RAS declarative API |

## References
- Architecture: `ARC42STORIES.MD` (project root)
