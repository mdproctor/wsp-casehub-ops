# Integration Test Hardening — Full Reconciliation Cycle

## Summary

Add full reconciliation cycle integration tests for the deployment and compliance modules, modeled after IoT's `IoTReconciliationIntegrationTest`. Each test exercises the complete planner-in-the-loop cycle: compile goals → read actual state → plan transitions → execute provisioning → verify. Extract shared test stubs to the `testing/` module.

## Motivation

The CI fix session (2026-08-14) demonstrated the gap: upstream API evolution (`ThresholdFaultPolicy.tier()`, `WorkItem` record migration, reactive removal) broke compilation and tests silently. Unit tests passed while actual SPI integration was broken. Full reconciliation cycle tests would catch these breaks immediately because they wire real SPI implementations together with the `TransitionPlanner`.

## Architecture

### New Test Classes

**`DeploymentReconciliationIntegrationTest`** in `deployment/src/test/`:
- Wires real `DeploymentGoalCompiler`, `DeploymentActualStateAdapter`, `DeploymentNodeProvisioner`, `DeploymentFaultPolicy` with shared stubs and `TransitionPlanner`
- No CDI, no `@QuarkusTest` — pure unit-style wiring with `new`

**`ComplianceReconciliationIntegrationTest`** in `compliance/src/test/`:
- Wires real `ComplianceGoalCompiler`, `ComplianceActualStateAdapter`, `ComplianceNodeProvisioner` with shared stubs and `TransitionPlanner`

### Shared Stubs in `testing/`

Extract from `DeploymentLifecycleIntegrationTest`:
- `StubAgentRegistry`
- `StubChannelOperations`
- `StubChannelStore`
- `StubChannelBindingStore`
- `StubEndpointRegistry`

Add for compliance:
- `StubComplianceFrameworkRegistry`
- `StubEvidenceCollector`

Update existing `DeploymentLifecycleIntegrationTest` to import shared stubs. The inline stub classes are removed from that test — this is the only modification to existing code.

## Test Scenarios

### Deployment — 3 scenarios

1. **Green-field provision** — Start with empty actual state. Compile deployment goals (agent, channel, case type, trust policy, endpoint). `TransitionPlanner.plan()` produces PROVISION steps for all 5 nodes. Execute each step via `DeploymentNodeProvisioner`. Read actual state — all nodes PRESENT.

2. **Drift remediation** — After green-field provision, recompile with a modified spec (e.g. agent name change). `readActual()` reports DRIFTED. `TransitionPlanner.plan()` produces UPDATE steps. Execute. Read actual — back to PRESENT with updated values.

3. **Node removal** — After green-field provision, recompile goals with one node removed. `TransitionPlanner.plan()` produces DEPROVISION step. Execute. Read actual — removed node ABSENT, others still PRESENT.

### Compliance — 3 scenarios

1. **Green-field provision** — Declare compliance controls (SOC2 control specs). Compile. Plan. Provision (registers controls in framework registry). Verify all PRESENT.

2. **Drift detection** — After provision, modify a control spec. Read actual reports DRIFTED. Plan produces remediation steps. Execute. Verify PRESENT.

3. **Evidence collection** — After provision, verify `ComplianceEvidenceService` produces evidence status for each registered control via the evidence collectors.

## Constraints

- No external services — all in-memory stubs
- No `@QuarkusTest` — pure unit-style wiring, millisecond execution
- Test classes named `*IntegrationTest.java` (not `*IT.java`) to ensure Surefire picks them up (per GE-20260416-ca1c71)
- Stubs in `testing/` module are test-scope only (per CLAUDE.md)

## Out of Scope

- YAML goal loading (GoalLoader concern, tested separately)
- `@QuarkusTest` / H2 / Hibernate integration (blocked by GE-20260718-d18dc0)
- Infra module reconciliation test (can follow same pattern later)
- FaultPolicy escalation within the reconciliation loop (tested in FaultPolicy unit tests)

## References

- `iot/src/test/.../IoTReconciliationIntegrationTest.java` — reference pattern
- `deployment/src/test/.../DeploymentLifecycleIntegrationTest.java` — existing SPI-level test, source of stubs to extract
- GE-20260416-ca1c71 — Maven `*IT.java` naming convention (use `*IntegrationTest.java` for Surefire)
- GE-20260718-d18dc0 — H2+Hibernate DDL failure (why we avoid `@QuarkusTest`)
- GE-20260427-edbacd — CDI singleton state bleed (informed stub design: stateless stubs, no shared mutable state across test methods)
