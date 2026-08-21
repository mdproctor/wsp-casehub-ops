# Integration Test Hardening — Full Reconciliation Cycle

## Summary

Add full reconciliation cycle integration tests for the deployment and compliance modules, modeled after IoT's `IoTReconciliationIntegrationTest`. Each test exercises the complete planner-in-the-loop cycle: load goals from YAML → compile → read actual state → plan transitions → execute provisioning → verify convergence. Extract shared test stubs to the `testing/` module.

## Motivation

The CI fix session (2026-08-14) demonstrated the gap: upstream API evolution (`ThresholdFaultPolicy.tier()`, `WorkItem` record migration, reactive removal) broke compilation and tests silently. Unit tests passed while actual SPI integration was broken. Full reconciliation cycle tests would catch these breaks immediately because they wire real SPI implementations together with the `TransitionPlanner`. The YAML→GoalLoader→GoalCompiler boundary is exactly where upstream record/API changes manifest first.

## Architecture

### New Test Classes

**`DeploymentReconciliationIntegrationTest`** in `deployment/src/test/`:
- Wires real `DeploymentGoalLoader`, `DeploymentGoalCompiler`, `DeploymentActualStateAdapter`, `DeploymentNodeProvisioner` with shared stubs and `TransitionPlanner`
- No CDI, no `@QuarkusTest` — pure unit-style wiring with `new`
- Does NOT wire `DeploymentFaultPolicy` — fault escalation is tested in dedicated unit tests and is not part of the reconciliation cycle

**`ComplianceReconciliationIntegrationTest`** in `compliance/src/test/`:
- Wires real `ComplianceGoalLoader`, `ComplianceGoalCompiler`, `ComplianceActualStateAdapter`, `ComplianceNodeProvisioner` with `TransitionPlanner`
- `ComplianceEvidenceService` wired with `StubEvidenceCollector` and in-memory `LedgerWriter`/`LatestEvidenceFinder` lambdas backed by a `List<ComplianceLedgerEntry>` — this is the critical wiring detail that connects provisioner side-effects (evidence writing) to adapter readActual (evidence status queries)
- Uses real `ComplianceFrameworkRegistry` directly — it's already an in-memory `ConcurrentHashMap` store, no stub needed

### Relationship to Existing Tests

`DeploymentLifecycleIntegrationTest` remains as-is (minus inline stubs, which move to `testing/`). It tests SPI implementations individually — "does provisioning work? does drift detection work?" The new reconciliation test tests the planner-in-the-loop cycle — "does the system converge?" Different concerns, complementary coverage.

### Shared Stubs in `testing/`

Extract from `DeploymentLifecycleIntegrationTest`:
- `StubAgentRegistry`
- `StubChannelOperations` — parameterize tenancyId via constructor (currently hardcoded to test class constant)
- `StubChannelStore`
- `StubChannelBindingStore`
- `StubEndpointRegistry`

Add for compliance:
- `StubEvidenceCollector` — returns configurable `ControlEvidenceStatus` per control

**POM changes required:** `testing/pom.xml` needs additional dependencies:
- `casehub-eidos-api` (for `AgentRegistry`, `AgentDescriptor`, `AgentQuery`)
- `casehub-qhorus-api` (for `Channel`, `ChannelCreateRequest`, `CrossTenantChannelStore`, `ChannelBindingStore`)
- `casehub-ops-compliance` (for `EvidenceCollector`, `ControlEvidenceStatus`)

All test-scope. These are API modules — no runtime weight.

Update existing `DeploymentLifecycleIntegrationTest` to import shared stubs. The inline stub classes are removed from that test.

## Test Scenarios

### Deployment — 4 scenarios

1. **YAML-to-graph provision** — Load deployment goals from a test YAML resource via `DeploymentGoalLoader`. Compile. ReadActual → all ABSENT. `TransitionPlanner.plan()` produces PROVISION steps for all 6 node types (agent, channel, case type, trust policy, endpoint, detection). Execute each step. ReadActual → all PRESENT. **Loop closure:** plan again → empty transition plan (system has converged).

2. **Drift remediation** — After green-field provision, recompile with a modified spec (e.g. agent name change). `readActual()` reports DRIFTED. `TransitionPlanner.plan()` produces remediation steps. Execute. ReadActual → back to PRESENT. Loop closure verified.

3. **Node removal** — After green-field provision, recompile goals with one node removed. `TransitionPlanner.plan()` produces DEPROVISION step. Execute via `provisioner.deprovision()`. **Positive absence verification:** assert the agent is no longer in `StubAgentRegistry` (not just that readActual returns ABSENT — readActual returns ABSENT for any unknown node). Remaining nodes still PRESENT.

4. **Programmatic goals** — Same cycle as scenario 1 but with goals constructed in code (no YAML). Validates the GoalCompiler independent of the GoalLoader. Covers the case where consumers build goals programmatically rather than from config files.

### Compliance — 3 scenarios

1. **Green-field provision** — Load compliance goals from test YAML via `ComplianceGoalLoader`. Compile. Plan. Provision (registers controls in framework registry, collects initial evidence). ReadActual → all PRESENT. **Loop closure:** plan again → empty transition plan.

2. **Drift detection and remediation** — After provision, modify a control spec. ReadActual reports DRIFTED. Plan produces remediation steps. Execute. ReadActual → PRESENT. Loop closure verified.

3. **Stable state convergence** — After provision, verify that a second readActual+plan cycle produces zero transitions. This tests loop closure — the fundamental reconciliation invariant. Evidence status is visible through the adapter's readActual (via `ComplianceEvidenceService.evidenceStatus()`) because the in-memory ledger writer and finder are wired to the same backing list.

## Constraints

- No external services — all in-memory stubs
- No `@QuarkusTest` — pure unit-style wiring, millisecond execution
- Test classes named `*IntegrationTest.java` (not `*IT.java`) to ensure Surefire picks them up (per GE-20260416-ca1c71)
- Stubs in `testing/` module are test-scope only (per CLAUDE.md)
- Every green-field scenario verifies loop closure (convergence to stable state)
- Deprovision scenarios verify positive absence in backing stores, not just readActual status

## Out of Scope

- `@QuarkusTest` / H2 / Hibernate integration (blocked by GE-20260718-d18dc0)
- Infra module reconciliation test (can follow same pattern later)
- FaultPolicy escalation within the reconciliation loop (tested in dedicated FaultPolicy unit tests; not wired in reconciliation tests)

## References

- `iot/src/test/.../IoTReconciliationIntegrationTest.java` — reference pattern
- `deployment/src/test/.../DeploymentLifecycleIntegrationTest.java` — existing SPI-level test, source of stubs to extract
- `compliance/src/main/.../ComplianceFrameworkRegistry.java` — already in-memory, no stub needed
- `compliance/src/test/.../ComplianceEvidenceServiceTest.java` — reference for compliance wiring pattern
- GE-20260416-ca1c71 — Maven `*IT.java` naming convention (use `*IntegrationTest.java` for Surefire)
- GE-20260718-d18dc0 — H2+Hibernate DDL failure (why we avoid `@QuarkusTest`)
- GE-20260427-edbacd — CDI singleton state bleed (informed stub design: stateless stubs, no shared mutable state across test methods)
