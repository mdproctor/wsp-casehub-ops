# Handoff — casehub-ops

## Last Session
Closed #64, #65, #66 on branch issue-66-upstream-api-adaptation. Fixed all compile errors from upstream API evolution (AgentDescriptor templates/goals/constraints, blocking SPI migration, jackson-jq convergence). Adopted ThresholdFaultPolicy in InfraFaultPolicy (8 node types) and DeploymentFaultPolicy (5 node types) — both escalate to human-review nodes after 3 PROVISION_FAILED events. Stamped 9 pre-existing unstamped closed branches. Recovered spec from issue-27 branch.

## Immediate Next Step
All S/XS issues and pre-existing compile failures are resolved. Pick up next priority work with `/work`.

## What's Left
- Pre-existing: @QuarkusTest + H2 + Hibernate 6.6 JOINED inheritance DDL failure (GE-20260718-d18dc0) — handled via @Disabled, needs TestContainers migration
- Pre-existing: IoT SPIs (DeviceProvider, DeviceRegistry) not yet migrated to blocking — upstream casehub-iot still uses Uni

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #25 | fsitrading adaptive ops | L | High | First real consumer |
| #26 | SOC adaptive ops | L | High | Second consumer |
| #16 | Compliance demo | M | Med | Unblocked |
| #17 | Infra demo | M | Med | Unblocked — InfraBackend SPI now blocking |
| #19 | Integration test hardening | M | Low | Unblocked |

## References
- Architecture: `ARC42STORIES.MD`
- Garden: GE-20260718-d18dc0 (H2+Hibernate DDL)
