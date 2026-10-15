# Fleet Scaling Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #133 — Level 04: Fleet scaling — deploy 3 LLM instances
**Issue group:** #133

**Goal:** Scale the hello-llm topology to a fleet of 3 differentiated
LLM instances using CSV data sources and forEach expansion, with
real-instance integration tests.

**Architecture:** The topology YAML defines a `data` block as a fleet
manifest (CSV with name + capability columns). A single `my-llm` node
template uses `forEach` to expand into 3 instances, each with a distinct
`applicationId` and capability config. Static eidos descriptors provide
per-instance identity as a Level A baseline. Integration tests deploy
against a real claudony instance.

**Tech Stack:** Java 26, Quarkus, desiredstate YAML (ForEachExpander +
CsvDataSource), JUnit 5, Testcontainers, WireMock

## Global Constraints

- Follow existing topology level patterns (01-03)
- Use `casehub_application` node type with `DeploymentNodeSpec` type registry
- Integration tests tagged `integration-live`, run with `-Pintegration-live`
- Compilation tests are standard JUnit (no profile needed)
- All topology YAMLs live in `topology-tests/src/test/resources/topologies/deployment/`

---

## Batch 1: Topology YAML + Compilation Tests

### Task 1: Create fleet topology YAML and verify compilation

**Files:**
- Create: `topology-tests/src/test/resources/topologies/deployment/04-llm-fleet.yaml`
- Modify: `topology-tests/src/test/java/io/casehub/ops/topology/compilation/DeploymentCompilationTest.java`

**Interfaces:**
- Consumes: `TopologyTestBase.compileSingleGraph(path, typeRegistry)`, `TopologyTestBase.buildTypeRegistryFrom(InfraNodeSpec.class, DeploymentNodeSpec.class)`
- Produces: `04-llm-fleet.yaml` topology file (consumed by Task 2 and Task 3)

- [ ] **Step 1: Write the failing compilation test**

Add `import java.util.List;` to `DeploymentCompilationTest.java`
(needed for `List.of` in the dependency test). Then use
`ide_insert_member` to add after the Level 03 section:

```java
// ── Level 04: Fleet Scaling ──────────────────────────────────────────

@Test
void llmFleet_hasSixNodes() throws IOException {
    DesiredStateGraph graph = compileSingleGraph(
            "topologies/deployment/04-llm-fleet.yaml", DEPLOYMENT_REGISTRY);
    assertThat(graph.nodes()).hasSize(6);
}

@Test
void llmFleet_allNodesAreCasehubApplication() throws IOException {
    DesiredStateGraph graph = compileSingleGraph(
            "topologies/deployment/04-llm-fleet.yaml", DEPLOYMENT_REGISTRY);
    graph.nodes().values().forEach(node ->
            assertThat(node.type()).isEqualTo(NodeType.of("casehub_application")));
}

@Test
void llmFleet_threeExpandedLlmNodes() throws IOException {
    DesiredStateGraph graph = compileSingleGraph(
            "topologies/deployment/04-llm-fleet.yaml", DEPLOYMENT_REGISTRY);
    assertThat(graph.nodes().keySet().stream()
            .map(NodeId::value)
            .filter(id -> id.startsWith("my-llm."))
            .toList())
            .containsExactlyInAnyOrder("my-llm.analyst", "my-llm.coder", "my-llm.assistant");
}

@Test
void llmFleet_allLlmNodesDependOnEngine() throws IOException {
    DesiredStateGraph graph = compileSingleGraph(
            "topologies/deployment/04-llm-fleet.yaml", DEPLOYMENT_REGISTRY);
    for (String llmId : List.of("my-llm.analyst", "my-llm.coder", "my-llm.assistant")) {
        assertThat(graph.dependenciesOf(NodeId.of(llmId)))
                .as("dependencies of %s", llmId)
                .contains(NodeId.of("engine"));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode -o test -pl topology-tests -Dtest=DeploymentCompilationTest#llmFleet_hasSixNodes`
Expected: FAIL — `04-llm-fleet.yaml` does not exist yet

- [ ] **Step 3: Create the topology YAML**

Create `topology-tests/src/test/resources/topologies/deployment/04-llm-fleet.yaml`:

```yaml
# ============================================================
# Level 04: Fleet Scaling
# ============================================================
# Scales the hello-llm topology from a single LLM app to a fleet
# of 3 differentiated LLM instances. Same app image, different
# per-instance configuration, all managed as desired state.
#
# Prerequisites: Level 01 (01-hello-llm.yaml)
#
# What you'll learn:
#   - How data blocks define fleet composition as structured data
#   - How forEach expands one node template into multiple instances
#   - How per-instance config differentiates fleet members
#   - How parameters flow through to eidos for identity generation
#
# Dependency graph:
#   manifest ──> eidos ──> engine ──┬──> my-llm.analyst
#                                   ├──> my-llm.coder
#                                   └──> my-llm.assistant
#
# The reconciliation loop ensures:
#   1. manifest starts first (no dependencies)
#   2. eidos starts after manifest
#   3. engine starts after eidos
#   4. All 3 LLM instances can deploy in parallel (all depend on engine)
# ============================================================

desiredState:
  namespace: casehub
  name: llm-fleet

data:
  llm-fleet:
    inline: |
      name:STRING,capability:STRING
      analyst,deep
      coder,fast
      assistant,balanced

nodes:
  manifest:
    type: casehub_application
    spec:
      applicationId: manifest
      image: casehubio/manifest:latest
      deploymentTarget: claudony
      replicas: 1

  eidos:
    type: casehub_application
    dependsOn: [manifest]
    spec:
      applicationId: eidos
      image: casehubio/eidos:latest
      deploymentTarget: claudony
      replicas: 1

  engine:
    type: casehub_application
    dependsOn: [eidos]
    spec:
      applicationId: engine
      image: casehubio/engine:latest
      deploymentTarget: claudony
      replicas: 1

  my-llm:
    type: casehub_application
    dependsOn: [engine]
    forEach:
      as: llm
      in: llm-fleet
    spec:
      applicationId: my-llm-${each.llm.name}
      image: casehubio/my-llm-app:latest
      deploymentTarget: claudony
      replicas: 1
      config:
        llm.capability: ${each.llm.capability}
        llm.identity-source: eidos
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn --batch-mode -o test -pl topology-tests -Dtest=DeploymentCompilationTest`
Expected: All Level 04 tests PASS (plus existing Level 01-03 tests)

- [ ] **Step 5: Commit**

```bash
git add topology-tests/src/test/resources/topologies/deployment/04-llm-fleet.yaml
git add topology-tests/src/test/java/io/casehub/ops/topology/compilation/DeploymentCompilationTest.java
git commit -m "feat(#133): Level 04 fleet topology — CSV data source + forEach expansion"
```

## Batch 2: Eidos Descriptors + Integration Test

### Task 2: Add fleet eidos descriptors

**Files:**
- Modify: `topology-tests/src/test/resources/META-INF/eidos/descriptors.yaml`

**Interfaces:**
- Consumes: nothing
- Produces: eidos descriptor entries for `my-llm-analyst`, `my-llm-coder`, `my-llm-assistant` (consumed by Task 3 integration test assertions)

- [ ] **Step 1: Add 3 fleet member descriptors**

Append to the existing `descriptors:` list in
`topology-tests/src/test/resources/META-INF/eidos/descriptors.yaml`:

```yaml
  - agentId: my-llm-analyst
    name: LLM Analyst
    slot: analyst
    disposition:
      conflictMode: collaborating
    capabilities:
      - name: conversation
        model:
          tier: DEEP
          capabilities: [text]
    briefing: >-
      You are an analyst agent specialising in deep reasoning
      and investigation. Part of a fleet of 3 LLM agents.

  - agentId: my-llm-coder
    name: LLM Coder
    slot: developer
    disposition:
      conflictMode: collaborating
    capabilities:
      - name: conversation
        model:
          tier: FAST
          capabilities: [text]
    briefing: >-
      You are a coding agent optimised for fast code generation
      and implementation. Part of a fleet of 3 LLM agents.

  - agentId: my-llm-assistant
    name: LLM Assistant
    slot: assistant
    disposition:
      conflictMode: collaborating
    capabilities:
      - name: conversation
        model:
          tier: BALANCED
          capabilities: [text]
    briefing: >-
      You are a general-purpose assistant providing balanced
      responses. Part of a fleet of 3 LLM agents.
```

- [ ] **Step 2: Commit**

```bash
git add topology-tests/src/test/resources/META-INF/eidos/descriptors.yaml
git commit -m "feat(#133): eidos descriptors for 3 fleet LLM instances"
```

### Task 3: Integration test — deploy fleet via real claudony

**Files:**
- Create: `topology-tests/src/test/java/io/casehub/ops/topology/integration/FleetScalingIntegrationTest.java`

**Interfaces:**
- Consumes: `TopologyTestBase.compileSingleGraph()`, `ClaudonyDeploymentTarget`, `HttpClaudonyFleetClient`, `@ServiceCluster`/`@ServiceEndpoint` annotations, `04-llm-fleet.yaml` (from Task 1), eidos descriptors (from Task 2)
- Produces: Integration test verifying real deployment of 3 fleet instances

- [ ] **Step 1: Write the integration test**

Create `FleetScalingIntegrationTest.java` using `ide_create_file`:

```java
package io.casehub.ops.topology.integration;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import io.casehub.desiredstate.api.DesiredNode;
import io.casehub.desiredstate.api.DesiredStateGraph;
import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.api.NodeStatus;
import io.casehub.ops.api.deployment.CaseHubApplicationSpec;
import io.casehub.ops.api.deployment.DeployConfig;
import io.casehub.ops.api.deployment.DeployResult;
import io.casehub.ops.api.deployment.DeploymentApplicationNodeSpec;
import io.casehub.ops.api.deployment.DeploymentNodeSpec;
import io.casehub.ops.api.infra.InfraNodeSpec;
import io.casehub.ops.claudony.ClaudonyDeploymentTarget;
import io.casehub.ops.claudony.HttpClaudonyFleetClient;
import io.casehub.ops.topology.compilation.TopologyTestBase;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;

import java.io.IOException;
import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

@Tag("integration-live")
@ServiceCluster(services = {
        @Service(name = "claudony", repo = "claudony/app")
})
class FleetScalingIntegrationTest extends TopologyTestBase {

    private static final Map<String, String> DEPLOYMENT_REGISTRY =
            buildTypeRegistryFrom(InfraNodeSpec.class, DeploymentNodeSpec.class);

    private static final List<String> FLEET_APP_IDS =
            List.of("my-llm-analyst", "my-llm-coder", "my-llm-assistant");

    @ServiceEndpoint("claudony")
    static String claudonyUrl;

    @Test
    void deploysThreeFleetInstancesViaRealClaudony() throws IOException {
        DesiredStateGraph graph = compileSingleGraph(
                "topologies/deployment/04-llm-fleet.yaml", DEPLOYMENT_REGISTRY);

        assertThat(graph.nodes()).hasSize(6);

        ObjectMapper mapper = new ObjectMapper();
        mapper.registerModule(new JavaTimeModule());
        HttpClaudonyFleetClient client = new HttpClaudonyFleetClient(claudonyUrl, mapper);
        ClaudonyDeploymentTarget target = new ClaudonyDeploymentTarget(client);

        for (DesiredNode node : graph.nodes().values()) {
            CaseHubApplicationSpec appSpec = extractAppSpec(node);
            DeployResult result = target.deploy(
                    appSpec.image(),
                    new DeployConfig(appSpec.applicationId(), appSpec.replicas(),
                            appSpec.config(), Map.of()));
            assertThat(result)
                    .as("Deploy %s", appSpec.applicationId())
                    .isInstanceOf(DeployResult.Success.class);
        }

        for (String appId : FLEET_APP_IDS) {
            NodeStatus status = target.status(appId);
            assertThat(status)
                    .as("Status of %s after deployment", appId)
                    .isNotEqualTo(NodeStatus.ABSENT);
        }
    }

    @Test
    void fleetInstancesHaveDistinctApplicationIds() throws IOException {
        DesiredStateGraph graph = compileSingleGraph(
                "topologies/deployment/04-llm-fleet.yaml", DEPLOYMENT_REGISTRY);

        List<String> llmAppIds = graph.nodes().entrySet().stream()
                .filter(e -> e.getKey().value().startsWith("my-llm."))
                .map(e -> extractAppSpec(e.getValue()).applicationId())
                .toList();

        assertThat(llmAppIds)
                .containsExactlyInAnyOrderElementsOf(FLEET_APP_IDS);
    }

    private CaseHubApplicationSpec extractAppSpec(DesiredNode node) {
        if (node.spec() instanceof DeploymentApplicationNodeSpec dns) {
            return dns.spec();
        }
        return (CaseHubApplicationSpec) node.spec();
    }
}
```

- [ ] **Step 2: Run the compilation-only test to verify it compiles**

Run: `mvn --batch-mode -o test -pl topology-tests -Dtest=FleetScalingIntegrationTest#fleetInstancesHaveDistinctApplicationIds`
Expected: PASS (this test doesn't need a real claudony — it only tests compilation)

- [ ] **Step 3: Run full integration test**

Run: `mvn --batch-mode test -pl topology-tests -Pintegration-live -Dtest=FleetScalingIntegrationTest`
Expected: Both tests PASS with real claudony instance

- [ ] **Step 4: Run full test suite to verify no regressions**

Run: `mvn --batch-mode -o test -pl topology-tests`
Expected: All existing tests PASS

- [ ] **Step 5: Commit**

```bash
git add topology-tests/src/test/java/io/casehub/ops/topology/integration/FleetScalingIntegrationTest.java
git commit -m "feat(#133): fleet scaling integration test — deploy 3 LLM instances via real claudony"
```

## References

- [2026-10-15-fleet-scaling-design.md] — design spec this plan implements
- [topology-tests/src/test/resources/topologies/deployment/01-hello-llm.yaml] — base topology pattern
- [topology-tests/src/test/java/io/casehub/ops/topology/compilation/DeploymentCompilationTest.java] — compilation test pattern
- [topology-tests/src/test/java/io/casehub/ops/topology/integration/HelloLlmIntegrationTest.java] — integration test pattern
- [topology-tests/src/test/java/io/casehub/ops/topology/deployment/ClaudonyWireMockSupport.java] — WireMock stubs
- [desiredstate/yaml/runtime ForEachExpanderTest.java:252-271] — CSV data source expansion
- [topology-tests/src/test/resources/META-INF/eidos/descriptors.yaml] — existing agent descriptor
- [GitHub #133] — focal issue
- [GitHub #131] — parent epic
