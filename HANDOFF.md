# Handoff — casehub-ops

## Last Session
Closed #45 on branch issue-45-k8s-faultpolicy-graph-mut. Extracted ThresholdFaultPolicy as a reusable count-based escalation component in casehub-desiredstate-api. Refactored IoTFaultPolicy to delegate (existing tests pass unchanged). Implemented KubernetesFaultPolicy — K8s resources escalate to human review after 3 PROVISION_FAILED events. API hardened: FaultEvent.detail and StepOutcome reason fields now reject null. Also migrated WorkerFunction.Sync (2→3 arg) and SettingsScope.root(tenancyId) across ops. Design review (4 rounds, $4.88) renamed component from EscalatingFaultPolicy to ThresholdFaultPolicy and moved it from runtime to api module. 3 garden entries submitted.

## Immediate Next Step
PR#63 is still open from the prior session (CI was red). Check its status — if still failing, investigate and fix.

## What's Left
- PR#63 pending merge to upstream · XS · Low
- jackson-jq dependency convergence (desiredstate 1.6.0 vs platform-expression 1.0.0) — needs parent POM fix · S · Low
- 9 unstamped closed branches (pre-existing hygiene debt)
- 1 unrecovered spec on closed branch issue-27 · XS · Low
- Pre-existing: @QuarkusTest + H2 + Hibernate 6.6 JOINED inheritance DDL failure (GE-20260718-d18dc0)
- Pre-existing: InfraBackend SPI changed upstream (blocking → reactive) — infra module doesn't compile · M · Med
- Pre-existing: app module test compilation failures from WorkerFunction.Sync raw type issues · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #64 | InfraFaultPolicy adoption of ThresholdFaultPolicy | S | Low | Unblocked by #45 — strong candidate per spec |
| #65 | DeploymentFaultPolicy evaluation | S | Med | Needs design — one-shot registrations |
| #25 | fsitrading adaptive ops | L | High | First real consumer |
| #26 | SOC adaptive ops | L | High | Second consumer |
| #16 | Compliance demo | M | Med | Unblocked |
| #17 | Infra demo | M | Med | Blocked by InfraBackend SPI fix |
| #19 | Integration test hardening | M | Low | Unblocked |

## References
- Architecture: `ARC42STORIES.MD`
- Spec: `docs/specs/2026-07-23-k8s-faultpolicy-escalation-design.md`
- Garden: GE-20260724-ce18be (SettingsScope.root tenancyId), GE-20260724-b3e9a6 (WorkerFunction.Sync), GE-20260724-20ab99 (api vs runtime placement)
- Cross-repo: desiredstate branch `issue-45-threshold-fault-policy` (closed, landed)
