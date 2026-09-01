# Integration Test Hardening Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #19 — test: integration test hardening — full reconciliation cycle for deployment and compliance
**Issue group:** #19

**Goal:** Add full reconciliation cycle integration tests for deployment and compliance modules, with shared stubs extracted to the testing/ module.

**Architecture:** Two new test classes (`DeploymentReconciliationIntegrationTest`, `ComplianceReconciliationIntegrationTest`) exercise the planner-in-the-loop cycle: YAML load → compile → readActual → TransitionPlanner.plan() → provision/deprovision → verify convergence. Shared stubs extracted from existing `DeploymentLifecycleIntegrationTest` to the `testing/` module.

**Tech Stack:** Java 21+, JUnit 5, AssertJ, Jackson YAML, casehub-desiredstate-runtime (TransitionPlanner, DefaultDesiredStateGraphFactory)

## Global Constraints

- No `@QuarkusTest` — pure unit-style wiring with `new`
- No external services — all in-memory stubs
- Test classes named `*IntegrationTest.java` (Surefire picks them up)
- `testing/` module is test-scope only
- Every green-field scenario verifies loop closure (second plan → empty)
- Deprovision verifies positive absence in backing stores

---

## Batch 1: Extract shared stubs to testing/ module

### Task 1: Add POM dependencies and extract deployment stubs

**Files:**
- Modify: `testing/pom.xml`
- Create: `testing/src/main/java/io/casehub/ops/testing/StubAgentRegistry.java`
- Create: `testing/src/main/java/io/casehub/ops/testing/StubChannelOperations.java`
- Create: `testing/src/main/java/io/casehub/ops/testing/StubChannelStore.java`
- Create: `testing/src/main/java/io/casehub/ops/testing/StubChannelBindingStore.java`
- Create: `testing/src/main/java/io/casehub/ops/testing/StubEndpointRegistry.java`
- Modify: `deployment/src/test/java/io/casehub/ops/deployment/DeploymentLifecycleIntegrationTest.java`

**Interfaces:**
- Produces: `StubAgentRegistry` implementing `AgentRegistry`, `StubChannelOperations` implementing `ChannelProvisionHandler.ChannelOperations` (tenancyId via constructor), `StubChannelStore` implementing `CrossTenantChannelStore`, `StubChannelBindingStore` implementing `ChannelBindingStore`, `StubEndpointRegistry` implementing `EndpointRegistry`

- [ ] **Step 1: Add dependencies to testing/pom.xml**

Add `casehub-eidos-api`, `casehub-qhorus-api`, `casehub-ops-compliance`, and `casehub-platform-api` dependencies to `testing/pom.xml`:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-eidos-api</artifactId>
</dependency>
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-qhorus-api</artifactId>
</dependency>
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-ops-compliance</artifactId>
</dependency>
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-api</artifactId>
</dependency>
```

- [ ] **Step 2: Extract StubAgentRegistry to testing/**

Copy the `StubAgentRegistry` inner class from `DeploymentLifecycleIntegrationTest.java` (lines 287-308) to a new top-level class in `testing/src/main/java/io/casehub/ops/testing/StubAgentRegistry.java`. Make it `public`. No other changes needed — it's already self-contained.

```java
package io.casehub.ops.testing;

import io.casehub.eidos.api.*;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.stream.Collectors;

public class StubAgentRegistry implements AgentRegistry {
    private final Map<String, AgentDescriptor> agents = new ConcurrentHashMap<>();

    @Override
    public void register(AgentDescriptor descriptor) {
        String key = descriptor.agentId() + ":" + descriptor.tenancyId();
        agents.put(key, descriptor);
    }

    @Override
    public Optional<AgentDescriptor> findById(String agentId, String tenancyId) {
        return Optional.ofNullable(agents.get(agentId + ":" + tenancyId));
    }

    @Override
    public List<AgentMatch> find(AgentQuery query) {
        return agents.values().stream()
                .map(d -> new AgentMatch(d, null))
                .collect(Collectors.toList());
    }

    public boolean contains(String agentId, String tenancyId) {
        return agents.containsKey(agentId + ":" + tenancyId);
    }
}
```

- [ ] **Step 3: Extract StubChannelStore, StubChannelBindingStore, StubChannelOperations, StubEndpointRegistry**

Extract each inner class from `DeploymentLifecycleIntegrationTest` to its own file in `testing/src/main/java/io/casehub/ops/testing/`. Key change for `StubChannelOperations`: parameterize `tenancyId` via constructor instead of referencing the test class constant.

`StubChannelOperations` constructor becomes:
```java
public StubChannelOperations(StubChannelStore channelStore, String tenancyId) {
    this.channelStore = channelStore;
    this.tenancyId = tenancyId;
}
```

`StubEndpointRegistry` — add a `contains` method for positive absence checks:
```java
public boolean contains(String path, String tenancyId) {
    return endpoints.containsKey(path + ":" + tenancyId);
}
```

- [ ] **Step 4: Update DeploymentLifecycleIntegrationTest to use shared stubs**

Remove the 5 inner stub classes from `DeploymentLifecycleIntegrationTest`. Add `import io.casehub.ops.testing.*;`. Update the `setUp()` method — the only change is `StubChannelOperations` now takes `tenancyId` as second constructor arg:

```java
channelOps = new StubChannelOperations(channelStore, TENANCY_ID);
```

- [ ] **Step 5: Verify existing tests still pass**

Run: `/opt/homebrew/bin/mvn --batch-mode -o test -pl testing,deployment -f /Users/mdproctor/claude/casehub/ops/pom.xml`

Expected: All existing deployment tests pass (including `DeploymentLifecycleIntegrationTest`).

- [ ] **Step 6: Commit**

```bash
git add testing/ deployment/src/test/java/io/casehub/ops/deployment/DeploymentLifecycleIntegrationTest.java
git commit -m "refactor(#19): extract shared test stubs to testing/ module"
```

---

## Batch 2: Deployment reconciliation integration test

### Task 2: DeploymentReconciliationIntegrationTest — YAML load + green-field + loop closure

**Files:**
- Create: `deployment/src/test/java/io/casehub/ops/deployment/DeploymentReconciliationIntegrationTest.java`
- Create: `deployment/src/test/resources/test-deployment/reconciliation-topology.yaml` (includes all 6 node types with detection)

**Interfaces:**
- Consumes: `StubAgentRegistry`, `StubChannelOperations`, `StubChannelStore`, `StubChannelBindingStore`, `StubEndpointRegistry` from `testing/`
- Produces: Full reconciliation test class with `setUp()` wiring, reused by later scenarios in this task

- [x] **Step 1: Create test YAML (5 types from YAML + detection programmatic)**

Create `deployment/src/test/resources/test-deployment/reconciliation-topology.yaml` with agent, channel, case type, trust policy, endpoint, and detection entries. Use the existing `topology.yaml` as base and add a detection entry:

```yaml
agents:
  - spec:
      agentId: recon-agent
      name: Reconciliation Agent
      slot: worker
    dependsOn: [recon/events]
channels:
  - spec:
      name: recon/work
      semantic: APPEND
    dependsOn: []
caseTypes:
  - spec:
      namespace: io.test
      name: recon-case
      version: "1.0"
    dependsOn: []
trust:
  - spec:
      capability: review
      threshold: 0.7
      minimumObservations: 5
      borderlineMargin: 0.1
      blendFactor: 0.5
      bootstrapEscalationRequired: false
    dependsOn: []
endpoints:
  - spec:
      path: recon/events
      type: SERVICE
      protocol: KAFKA
      properties:
        topic: recon.events
      capabilities: [RECEIVE]
    dependsOn: []
detections:
  - spec:
      detectionId: recon-detect
      situationType: anomaly
      description: test detection
    dependsOn: []
```

- [x] **Step 2: Write the green-field test with loop closure**

```java
@Test
void greenField_yamlLoad_provisionAll_loopClosure() {
    // Load from YAML
    var goals = goalLoader.load("test-deployment/reconciliation-topology.yaml");

    // Compile
    var desired = ((CompilationResult.SingleGraph) compiler.compile(goals, graphFactory)).graph();
    assertThat(desired.nodes()).hasSize(6);

    // ReadActual — all ABSENT
    var actual = adapter.readActual(desired, TENANCY_ID);
    for (var status : actual.statuses().values()) {
        assertThat(status).isEqualTo(NodeStatus.ABSENT);
    }

    // Plan
    var plan = planner.plan(desired, actual);
    assertThat(plan.additions()).isNotEmpty();

    // Execute provisions
    for (var step : plan.additions()) {
        if (step.action() == StepAction.PROVISION) {
            var result = provisioner.provision(step.node(), new ProvisionContext(TENANCY_ID, desired));
            assertThat(result).as("provisioning %s", step.node().id()).isInstanceOf(ProvisionResult.Success.class);
        }
    }

    // ReadActual — all PRESENT
    var actualAfter = adapter.readActual(desired, TENANCY_ID);
    for (var entry : actualAfter.statuses().entrySet()) {
        assertThat(entry.getValue()).as("node %s", entry.getKey()).isEqualTo(NodeStatus.PRESENT);
    }

    // Loop closure — second plan produces empty additions
    var secondPlan = planner.plan(desired, actualAfter);
    assertThat(secondPlan.additions()).isEmpty();
}
```

- [x] **Step 3: Write setUp() wiring**

Wire `DeploymentGoalLoader`, `DeploymentGoalCompiler`, `DeploymentActualStateAdapter`, `DeploymentNodeProvisioner`, `TransitionPlanner` with shared stubs. Follow the pattern from `DeploymentLifecycleIntegrationTest.setUp()` but add `goalLoader` and `planner`.

- [x] **Step 4: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn --batch-mode -o test -pl deployment -Dtest=DeploymentReconciliationIntegrationTest#greenField_yamlLoad_provisionAll_loopClosure -f /Users/mdproctor/claude/casehub/ops/pom.xml`

Expected: PASS

- [x] **Step 5: Add drift remediation scenario**

```java
@Test
void driftRemediation_modifySpec_recompile_remediate() {
    // Green-field first
    var goals = goalLoader.load("test-deployment/reconciliation-topology.yaml");
    var desired = ((CompilationResult.SingleGraph) compiler.compile(goals, graphFactory)).graph();
    var actual = adapter.readActual(desired, TENANCY_ID);
    var plan = planner.plan(desired, actual);
    for (var step : plan.additions()) {
        if (step.action() == StepAction.PROVISION) {
            provisioner.provision(step.node(), new ProvisionContext(TENANCY_ID, desired));
        }
    }

    // Modify agent name → recompile
    // Build modified goals programmatically (change agent name)
    var modifiedGoals = buildModifiedGoals(goals, "recon-agent", "Modified Agent Name");
    var modifiedDesired = ((CompilationResult.SingleGraph) compiler.compile(modifiedGoals, graphFactory)).graph();

    // ReadActual against modified desired → DRIFTED for agent
    var driftActual = adapter.readActual(modifiedDesired, TENANCY_ID);
    assertThat(driftActual.statusOf(NodeId.of("recon-agent"))).contains(NodeStatus.DRIFTED);

    // Plan → remediation steps
    var remediationPlan = planner.plan(modifiedDesired, driftActual);
    assertThat(remediationPlan.additions()).isNotEmpty();

    // Execute remediation
    for (var step : remediationPlan.additions()) {
        if (step.action() == StepAction.PROVISION) {
            var result = provisioner.provision(step.node(), new ProvisionContext(TENANCY_ID, modifiedDesired));
            assertThat(result).isInstanceOf(ProvisionResult.Success.class);
        }
    }

    // Verify converged
    var afterRemediation = adapter.readActual(modifiedDesired, TENANCY_ID);
    for (var entry : afterRemediation.statuses().entrySet()) {
        assertThat(entry.getValue()).as("node %s after remediation", entry.getKey()).isEqualTo(NodeStatus.PRESENT);
    }
}
```

- [x] **Step 6: Add node removal scenario (detection, not agent — AgentProvisionHandler has no deregister)**

```java
@Test
void nodeRemoval_removeAgent_deprovision_positiveAbsence() {
    // Green-field first
    var goals = goalLoader.load("test-deployment/reconciliation-topology.yaml");
    var desired = ((CompilationResult.SingleGraph) compiler.compile(goals, graphFactory)).graph();
    var actual = adapter.readActual(desired, TENANCY_ID);
    var plan = planner.plan(desired, actual);
    for (var step : plan.additions()) {
        if (step.action() == StepAction.PROVISION) {
            provisioner.provision(step.node(), new ProvisionContext(TENANCY_ID, desired));
        }
    }

    // Verify agent exists in registry
    assertThat(agentRegistry.contains("recon-agent", TENANCY_ID)).isTrue();

    // Recompile without the agent
    var reducedGoals = buildGoalsWithoutAgent(goals, "recon-agent");
    var reducedDesired = ((CompilationResult.SingleGraph) compiler.compile(reducedGoals, graphFactory)).graph();

    // Plan against old actual → DEPROVISION for removed agent
    var fullActual = adapter.readActual(desired, TENANCY_ID);
    var removalPlan = planner.plan(reducedDesired, fullActual);
    assertThat(removalPlan.removals()).isNotEmpty();

    // Execute deprovision
    for (var step : removalPlan.removals()) {
        if (step.action() == StepAction.DEPROVISION) {
            var result = provisioner.deprovision(step.node(), new DeprovisionContext(TENANCY_ID, reducedDesired));
            assertThat(result).isInstanceOf(DeprovisionResult.Success.class);
        }
    }

    // Positive absence — agent no longer in registry
    assertThat(agentRegistry.contains("recon-agent", TENANCY_ID)).isFalse();
}
```

- [x] **Step 7: Run all deployment tests (201 pass, FaultPolicy failures pre-existing)**

Run: `/opt/homebrew/bin/mvn --batch-mode -o test -pl deployment -f /Users/mdproctor/claude/casehub/ops/pom.xml`

Expected: All tests pass.

- [x] **Step 8: Commit** — `6f3aa73`
```

---

## Batch 3: Compliance reconciliation integration test

### Task 3: ComplianceReconciliationIntegrationTest — YAML load + green-field + drift + convergence

**Files:**
- Create: `testing/src/main/java/io/casehub/ops/testing/StubEvidenceCollector.java`
- Create: `compliance/src/test/java/io/casehub/ops/compliance/ComplianceReconciliationIntegrationTest.java`

**Interfaces:**
- Consumes: `StubEvidenceCollector` from `testing/`; real `ComplianceGoalLoader`, `ComplianceGoalCompiler`, `ComplianceActualStateAdapter`, `ComplianceNodeProvisioner`, `ComplianceEvidenceService`, `ComplianceFrameworkRegistry`, `ComplianceSpecHashStore`

- [x] **Step 1: Create StubEvidenceCollector** (already existed from Batch 1)

```java
package io.casehub.ops.testing;

import io.casehub.ops.api.compliance.*;

public class StubEvidenceCollector implements EvidenceCollector {
    private final String strategy;
    private EvidenceResult result = new EvidenceResult.Pass("stub evidence");

    public StubEvidenceCollector(String strategy) {
        this.strategy = strategy;
    }

    @Override
    public String strategy() { return strategy; }

    @Override
    public EvidenceResult collect(ComplianceControlSpec spec, String tenancyId) {
        return result;
    }

    public void setResult(EvidenceResult result) { this.result = result; }
}
```

- [x] **Step 2: Write the compliance test with setUp() wiring**

Key wiring detail: `ComplianceEvidenceService` uses its test constructor with `LedgerWriter` and `LatestEvidenceFinder` backed by a shared `List<ComplianceLedgerEntry>`:

```java
private List<ComplianceLedgerEntry> ledgerEntries = new ArrayList<>();

@BeforeEach
void setUp() {
    ledgerEntries.clear();
    var collectors = List.of(
        new StubEvidenceCollector("FILE_EXISTENCE"),
        new StubEvidenceCollector("LOG_DIRECTORY"),
        new StubEvidenceCollector("CERTIFICATE_EXPIRY"),
        new StubEvidenceCollector("CONFIG_HASH"));

    ComplianceEvidenceService.LedgerWriter writer = (entry, tenancyId) -> ledgerEntries.add(entry);
    ComplianceEvidenceService.LatestEvidenceFinder finder = (controlId, tenancyId) ->
        ledgerEntries.stream()
            .filter(e -> controlId.equals(e.controlId))
            .sorted(Comparator.comparing(e -> ((ComplianceLedgerEntry) e).occurredAt).reversed())
            .limit(1)
            .toList();

    evidenceService = new ComplianceEvidenceService(collectors, writer, finder);
    registry = new ComplianceFrameworkRegistry();
    specHashStore = new ComplianceSpecHashStore();
    var approvalEvaluator = new ComplianceApprovalEvaluator();
    var planStore = new io.casehub.ops.api.approval.InMemoryPlanStore();

    provisioner = new ComplianceNodeProvisioner(evidenceService, registry, specHashStore, approvalEvaluator, planStore);
    adapter = new ComplianceActualStateAdapter(evidenceService, specHashStore);
    compiler = new ComplianceGoalCompiler();
    goalLoader = new ComplianceGoalLoader();
    planner = new TransitionPlanner();
    graphFactory = new DefaultDesiredStateGraphFactory();
}
```

- [x] **Step 3: Write green-field + loop closure scenario**

```java
@Test
void greenField_yamlLoad_provisionAll_loopClosure() {
    var goals = goalLoader.load("test-compliance/all-controls.yaml");
    var desired = ((CompilationResult.SingleGraph) compiler.compile(goals, graphFactory)).graph();

    var actual = adapter.readActual(desired, TENANCY_ID);
    // All ABSENT initially (no evidence in ledger)
    for (var status : actual.statuses().values()) {
        assertThat(status).isEqualTo(NodeStatus.ABSENT);
    }

    var plan = planner.plan(desired, actual);
    assertThat(plan.additions()).isNotEmpty();

    for (var step : plan.additions()) {
        if (step.action() == StepAction.PROVISION) {
            var result = provisioner.provision(step.node(), new ProvisionContext(TENANCY_ID, desired));
            assertThat(result).as("provisioning %s", step.node().id()).isInstanceOf(ProvisionResult.Success.class);
        }
    }

    // All PRESENT after provisioning (evidence collected and fresh)
    var afterProvision = adapter.readActual(desired, TENANCY_ID);
    for (var entry : afterProvision.statuses().entrySet()) {
        assertThat(entry.getValue()).as("node %s", entry.getKey()).isEqualTo(NodeStatus.PRESENT);
    }

    // Loop closure
    var secondPlan = planner.plan(desired, afterProvision);
    assertThat(secondPlan.additions()).isEmpty();
}
```

- [x] **Step 4: Write drift + remediation scenario**

Modify a control spec (change `evidenceMaxAgeDays`), recompile, verify DRIFTED via spec hash, remediate, verify converged.

- [x] **Step 5: Write stable-state convergence scenario**

After green-field provision, verify readActual+plan produces zero transitions. This is the minimal loop closure test — separate from scenario 1 to keep it focused.

- [x] **Step 6: Run all compliance tests (81 pass, 0 failures)**

Run: `/opt/homebrew/bin/mvn --batch-mode -o test -pl compliance -f /Users/mdproctor/claude/casehub/ops/pom.xml`

Expected: All tests pass.

- [x] **Step 7: Full build blocked by pre-existing FaultPolicy.addReviewNode API evolution (deployment, infra, app)**

- [x] **Step 8: Commit** — `e0f2f55`

---

## References

- [2026-08-20-integration-test-hardening-design.md] — design spec this plan implements
- `iot/src/test/.../IoTReconciliationIntegrationTest.java` — reference pattern
- `deployment/src/test/.../DeploymentLifecycleIntegrationTest.java:287-488` — stub source for extraction
- `compliance/src/main/.../ComplianceEvidenceService.java:57-65` — test constructor wiring
- `compliance/src/main/.../ComplianceFrameworkRegistry.java` — in-memory, no stub needed
- `deployment/src/test/resources/test-deployment/topology.yaml` — existing test YAML
- `compliance/src/test/resources/test-compliance/all-controls.yaml` — existing test YAML
- GE-20260416-ca1c71 — Maven `*IT.java` naming convention
- GE-20260427-edbacd — CDI singleton state bleed
- GitHub #19
