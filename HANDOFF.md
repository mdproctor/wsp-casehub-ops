# Handoff — casehub-ops

## Last Session
Closed #6 (EndpointNodeSpec — 5th deployment node type). Endpoint provisioning via EndpointRegistry, drift detection via toDescriptor() + record equals(). Also closed #8 (worker import migration — not applicable, no Worker imports in ops). readActual(graph, tenancyId) SPI propagated to all 4 modules. Blog published. parent#312 filed for PLATFORM.md updates. platform#117 filed to verify EndpointRegistered CDI event firing.

## Immediate Next Step
Pick next work. Open issues: #10 (IoTFaultPolicy — M/Med, deferred pending operational feedback), #11 (DetectionNodeSpec — deferred pending RAS declarative API), #12 (agentIds() on DeploymentProviderConfigStore). Run `/work` to start.

## Cross-Module
**Blocked by:**
- `casehub-desiredstate` — SimpleTransitionExecutor has no WorkItem creation for requiresHuman=true (desiredstate#43) · S · Low
- `casehub-platform` — verify EndpointRegistered CDI event fires from real EndpointRegistry (platform#117) · XS · Low

## What's Left
- desiredstate#43 WorkItem creation for requiresHuman — filed, open · S · Low
- parent#312 PLATFORM.md update for endpoint node type — filed, open · XS · Low
- platform#117 EndpointRegistered event verification — filed, open · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #12 | agentIds() on DeploymentProviderConfigStore | XS | Low | Consumer enumeration |
| #10 | IoTFaultPolicy domain-specific fault responses | M | Med | Deferred — needs operational feedback |
| #11 | DetectionNodeSpec — RAS situation registration | M | Med | Deferred — requires RAS declarative API |

## References
- Architecture: `ARC42STORIES.MD` (project root)
- Spec: `docs/superpowers/specs/2026-06-26-deployment-endpoint-node-type-design.md`
- Blog: `blog/2026-06-26-mdp01-deployment-endpoint-topology.md`
