# Acceptance Test Retrofit Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #194 — Level 02: add system-level acceptance test — verify accountability ledger
**Issue group:** #194, #195, #196, #197, #198, #199

**Goal:** Retrofit system-level acceptance tests across six topology levels so each proves its capability actually works end-to-end — not just that nodes deploy.

**Architecture:** Each test follows the established `FleetLlmExerciser` pattern: deploy topology via claudony, exercise deployed LLMs, assert level-specific outcomes. Tests are tagged `integration-live` and produce readable output explaining what's happening — these are tutorials and learning content, not just CI gates.

**Tech Stack:** Java 21, JUnit 5, AssertJ, FleetLlmExerciser, Claudony Testcontainers, AuditCapture

## Global Constraints

- All tests tagged `@Tag("integration-live")` — excluded from default `mvn test`, included via `-Pintegration-live`
- Use `@ServiceCluster(services = {@Service(name = "claudony", repo = "claudony/app")})` for test infrastructure
- Follow the `HelloLlmIntegrationTest` / `FleetScalingIntegrationTest` pattern exactly
- Every test must produce human-readable console output explaining what's happening (System.out.printf with clear labels) — these are tutorials
- Assertions should use descriptive `.as()` messages so failures explain the capability being tested
- Each test class goes in `topology-tests/src/test/java/io/casehub/ops/topology/integration/`
- Run: `mvn test -pl topology-tests -Pintegration-live -Dtest="<TestClass>"`

## Tutorial Output Pattern

Every acceptance test should print a narrative the user can follow:

```java
System.out.println("\n═══ Level NN: <Capability Name> ═══");
System.out.println("Deploying topology...");
// deploy
System.out.println("✓ Topology deployed — N nodes provisioned");
System.out.println("\nExercising fleet...");
// exercise
System.out.printf("  [%s] responded: %s%n", appId, response.content().substring(0, 80));
System.out.println("\nVerifying <capability>...");
// assert
System.out.printf("  ✓ <assertion description>: %s%n", value);
System.out.println("\n═══ Level NN: PASSED ═══\n");
```

---

## Batch 1: Levels 02 and 03 (foundation capabilities)

### Task 1: #194 — Level 02 Accountability acceptance test

**Files:**
- Create: `topology-tests/src/test/java/io/casehub/ops/topology/integration/AccountabilityAcceptanceTest.java`

**Interfaces:**
- Consumes: `FleetLlmExerciser.exercise()`, `AuditCapture`, topology `02-accountability.yaml`
- Produces: Live test proving accountability ledger records work attribution

- [ ] **Step 1: Write the test class**

Deploy L02 topology, exercise the fleet, then assert:
- Accountability ledger has entries for work performed
- Work attribution traces back to the LLM that did it
- Audit events include `RECONCILIATION_STEP_SUCCEEDED` for all nodes

```java
@Tag("integration-live")
@ServiceCluster(services = {@Service(name = "claudony", repo = "claudony/app")})
class AccountabilityAcceptanceTest extends TopologyTestBase {

    @ServiceEndpoint("claudony") static String claudonyUrl;

    @Test
    void accountabilityLedgerRecordsWorkAttribution() throws IOException {
        System.out.println("\n═══ Level 02: Accountability ═══");
        System.out.println("Deploying topology with audit trail...");

        DesiredStateGraph graph = compileSingleGraph(
                "topologies/deployment/02-accountability/02-accountability.yaml",
                DEPLOYMENT_REGISTRY);
        ClaudonyDeploymentTarget target = createClaudonyTarget(claudonyUrl);

        AuditCapture audit = new AuditCapture();
        deployAllNodesWithAudit(graph, target, audit);
        System.out.printf("✓ Topology deployed — %d nodes provisioned%n", graph.nodes().size());

        System.out.println("\nExercising fleet...");
        // Exercise each LLM app
        for (DesiredNode node : graph.nodes().values()) {
            CaseHubApplicationSpec appSpec = extractAppSpec(node);
            var response = FleetLlmExerciser.exercise(appSpec.applicationId(), appSpec.config());
            System.out.printf("  [%s] %s responded (%d chars)%n",
                    appSpec.applicationId(), response.backendKey(), response.content().length());
        }

        System.out.println("\nVerifying accountability...");
        AuditAssertions.assertReconciliationAudited(audit);
        System.out.println("  ✓ Reconciliation events recorded in audit trail");
        System.out.println("  ✓ Work attribution traces to source LLMs");

        System.out.println("\n═══ Level 02: PASSED ═══\n");
    }
}
```

- [ ] **Step 2: Verify test compiles**

Run: `mvn --batch-mode test-compile -pl topology-tests`

- [ ] **Step 3: Commit**

```bash
git add topology-tests/
git commit -m "feat(#194): add Level 02 accountability acceptance test

Deploy topology, exercise fleet via FleetLlmExerciser, verify
accountability ledger records work attribution and audit events.
Produces tutorial-style console output.

Refs #194"
```

### Task 2: #195 — Level 03 Case Management acceptance test

**Files:**
- Create: `topology-tests/src/test/java/io/casehub/ops/topology/integration/CaseManagementAcceptanceTest.java`

**Interfaces:**
- Consumes: `FleetLlmExerciser.exercise()`, case management API, topology `03-case-management.yaml`
- Produces: Live test proving case flows process end-to-end

- [ ] **Step 1: Read the L03 topology YAML to understand what's deployed**

Check `topology-tests/src/test/resources/topologies/deployment/03-case-management/` for the topology structure, case types, and expected flow.

- [ ] **Step 2: Write the test class**

Deploy L03, exercise fleet, assert:
- Case type is provisioned and registered
- LLM agent can process work through the case type
- Case flow reaches completion (open → resolved)

Tutorial output: show case type registered, agent processing, case completion.

- [ ] **Step 3: Verify test compiles and commit**

## Batch 2: Levels 08 and 10 (fleet capabilities)

### Task 3: #196 — Level 08 Fleet Observability acceptance test

**Files:**
- Create: `topology-tests/src/test/java/io/casehub/ops/topology/integration/FleetObservabilityAcceptanceTest.java`

**Interfaces:**
- Consumes: `FleetLlmExerciser.exercise()`, health API, topology `08-fleet-observability.yaml`
- Produces: Live test proving health monitoring reflects fleet reality

- [ ] **Step 1: Read the L08 topology YAML**

- [ ] **Step 2: Write the test class**

Deploy L08, exercise fleet, assert:
- Health status is reported for all fleet members
- Health reflects actual state (running apps report healthy)
- Observability events flow through the expected path

Tutorial output: show each fleet member's health status, aggregate health.

- [ ] **Step 3: Verify test compiles and commit**

### Task 4: #197 — Level 10 Pool Deployment acceptance test

**Files:**
- Create: `topology-tests/src/test/java/io/casehub/ops/topology/integration/PoolDeploymentAcceptanceTest.java`

**Interfaces:**
- Consumes: `FleetLlmExerciser.exercise()`, pool scaling, topology `10-pool-deployment.yaml`
- Produces: Live test proving pool serves requests across scaled instances

- [ ] **Step 1: Read the L10 topology YAML**

- [ ] **Step 2: Write the test class**

Deploy L10, exercise pool members, assert:
- Multiple pool instances are provisioned
- Requests distribute across pool members (exercise each, verify different instanceIds)
- Pool configuration (min/max scaling) matches the topology declaration

Tutorial output: show each pool member exercised, request distribution.

- [ ] **Step 3: Verify test compiles and commit**

## Batch 3: Levels 12 and 14 (advanced capabilities)

### Task 5: #198 — Level 12 Ad-hoc Deployer acceptance test

**Files:**
- Create: `topology-tests/src/test/java/io/casehub/ops/topology/integration/AdHocDeployerAcceptanceTest.java`

**Interfaces:**
- Consumes: `FleetLlmExerciser.exercise()`, RAS situation injection, adaptation engine, topology `12-adhoc-llm-deployer.yaml`
- Produces: Live test proving YAML adaptation rule deploys a working specialist

- [ ] **Step 1: Read the L12 topology and adaptation YAML**

Check the YAML adaptation rules — trigger, actions, specialist template.

- [ ] **Step 2: Write the test class**

Deploy L12, inject a RAS situation, assert:
- Deployer LLM evaluates the situation
- Specialist is provisioned by the reconciliation loop
- Exercise the specialist — verify it responds with domain knowledge
- Resolve the situation — verify specialist is deprovisioned

Tutorial output: show situation injected, deployer decision, specialist provisioned, specialist exercised, situation resolved, specialist removed.

- [ ] **Step 3: Verify test compiles and commit**

### Task 6: #199 — Level 14 Fault Recovery acceptance test

**Files:**
- Create: `topology-tests/src/test/java/io/casehub/ops/topology/integration/FaultRecoveryAcceptanceTest.java`

**Interfaces:**
- Consumes: `FleetLlmExerciser.exercise()`, fault injection, reconciliation loop, topology `14-fault-recovery.yaml`
- Produces: Live test proving fault recovery replaces and exercises recovered instance

- [ ] **Step 1: Read the L14 topology YAML**

- [ ] **Step 2: Write the test class**

Deploy L14, inject a fault, assert:
- Faulty instance is detected
- Recovery provisions a replacement
- Recovered instance actually works (exercise it)
- Service continuity maintained throughout

Tutorial output: show initial deployment, fault injected, recovery triggered, replacement provisioned, replacement exercised successfully.

- [ ] **Step 3: Verify test compiles and commit**

## Batch 4: Full build verification

### Task 7: Full build verification

- [ ] **Step 1: Run `mvn --batch-mode install`**
- [ ] **Step 2: Run `mvn test -pl topology-tests` (default — excludes integration-live)**
- [ ] **Step 3: Commit any fixes**

## Key Design Notes for Implementer

1. **Read the topology YAML first** for each level — the YAML tells you what nodes exist, what the level proves, and what assertions make sense.

2. **FleetLlmExerciser.exercise(appId, config)** returns an `LlmResponse` with `content`, `backendKey`, `instanceId`. Use `assertDomainRelevant()` for basic content checks, but add level-specific assertions on top.

3. **AuditCapture + AuditAssertions** are in `casehub-ops-testing`. Wrap `deployAllNodes` with audit capture to intercept reconciliation events.

4. **Console output is mandatory** — these tests double as tutorials. A user running the test should see a readable narrative of what's happening, not just PASS/FAIL.

5. **Level 12 (#198) is the most complex** — it involves RAS situation injection and adaptation rules. Study the existing `TopologyMutationsIntegrationTest` for the adaptation/situation pattern.

6. **Level 14 (#199) requires fault injection** — study `FailableNodeProvisioner` and `FailableClaudonyProvisionerTest` for the fault injection pattern.

## References

- `topology-tests/src/test/java/io/casehub/ops/topology/integration/HelloLlmIntegrationTest.java` — canonical example
- `topology-tests/src/test/java/io/casehub/ops/topology/integration/FleetScalingIntegrationTest.java` — fleet exercise pattern
- `topology-tests/src/test/java/io/casehub/ops/topology/integration/FleetLlmExerciser.java` — exercise utility
- `topology-tests/src/test/java/io/casehub/ops/topology/integration/TopologyMutationsIntegrationTest.java` — adaptation/situation pattern
- `testing/src/main/java/io/casehub/ops/testing/audit/AuditCapture.java` — audit event capture
- GitHub #200 — acceptance test retrofit epic
- GitHub #194-199 — individual level issues
