# Handover — Level 14 fault recovery

**Branch:** `main` (clean)
**Landed this session:** #154

## What was done

1. **#154** Level 14: Fault Recovery — topology YAML (`14-fault-recovery.yaml`), 4 compilation tests, 4 integration tests exercising FaultPolicy SPI with controlled failure injection via FailableClaudonyProvisioner. Tests cover: automatic recovery (transient failure, 2 faults below threshold 3), escalation to human review (persistent failure, 10 faults), fleet preservation (other nodes stay PRESENT), and audit trail verification (NODE_FAULTED + RECONCILIATION_COMPLETED CloudEvents). No production code changes — uses existing ClaudonyFaultPolicy unchanged.

2. Decision review caught and eliminated a multi-tier marker node design (FaultRecoveryMarkerSpec + no-op provisioner). The simpler single-tier approach proved sufficient: below threshold 3, convergence retries automatically; at threshold 3, human review escalates.

## Immediate Next Step

Candidates (all M-scale, need brainstorming):
- **#147** Level 11: Blocks use case — logistics summariser (M/Med)
- **#155** Level 15: Multi-tenancy — isolated tenants (M/High, builds on Level 10)
- **#148** Level 12: Ad-hoc LLM deployer — runtime deployment via RAS input (M/Med)

## References

- Level 14 topology: `topology-tests/src/test/resources/topologies/deployment/14-fault-recovery/`
- Design spec: `specs/issue-154-fault-recovery-detection-replacement/2026-10-24-fault-recovery-design.md` (in workspace)
- Decision review: `reviews/casehub-ops/issue-154-decision-20261024-212334/`
