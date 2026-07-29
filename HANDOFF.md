# Handoff — casehub-ops

## Last Session
Fixed CI (#67) — adapted IoT tests to upstream blocking SPI migration (iot#78, iot#79). Also fixed casehub-desiredstate CI (#92) — stale `Uni.await()` in SituationDetectionTest was blocking SNAPSHOT publication. Created and landed slot 40 (#93) — eliminated all time-sensitive test patterns across desiredstate and ops.

## Immediate Next Step
CI is green. All housekeeping resolved. Pick up next priority work with `/work`.

## What's Left
- Pre-existing: @QuarkusTest + H2 + Hibernate 6.6 JOINED inheritance DDL failure (GE-20260718-d18dc0) — handled via @Disabled, needs TestContainers migration

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
