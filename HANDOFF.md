# Handoff — casehub-ops

## Last Session
Closed 4 S-scale issues on branch issue-10-s-issues-batch. IoTFaultPolicy: PROVISION_FAILED escalation (3 failures → iot-review node with HumanGating.ALL → WorkItem). ComplianceNodeProvisioner.resyncInterval() → 1 hour. #20 and #48 closed as already done (multi-provisioner dispatch and RAS migration shipped in prior sessions). ARC42STORIES.MD synced for #43 approval workflow and #10/#21 journal merge. CI fixed (re-run after desiredstate#84 deployed). PR#63 open for upstream.

## Immediate Next Step
Check PR#63 CI status. If green, merge. Then pick next work — #25 (fsitrading adaptive ops) is the natural next.

## What's Left
- PR#63 pending merge to upstream · XS · Low
- jackson-jq dependency convergence (desiredstate 1.6.0 vs platform-expression 1.0.0) — needs parent POM fix · S · Low
- 9 unstamped closed branches (pre-existing hygiene debt)
- 1 unrecovered spec on closed branch issue-27 · XS · Low
- Pre-existing: @QuarkusTest + H2 + Hibernate 6.6 JOINED inheritance DDL failure (GE-20260718-d18dc0)

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #25 | fsitrading adaptive ops | L | High | First real consumer — all scaling + approval infrastructure in place |
| #26 | SOC adaptive ops | L | High | Second consumer |
| #16 | Compliance demo | M | Med | Unblocked — case model + real EvidenceCollectors |
| #17 | Infra demo | M | Med | Unblocked |
| #45 | K8s-aware FaultPolicy responses | M | Med | Needs operational feedback |
| #19 | Integration test hardening | M | Low | Unblocked |

## References
- Architecture: `ARC42STORIES.MD`
- PR: casehubio/casehub-ops#63
- Garden: GE-20260718-d18dc0 (H2/Hibernate JOINED inheritance), GE-20260706-2ac0db (FaultPolicy CDI wiring)
