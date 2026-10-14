# Level 01: Hello LLM — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** TBD (create before starting)
**Issue group:** Level 01 of deployment topology integration tests

**Goal:** Create a data-driven integration test that compiles a YAML
topology declaring three CaseHub apps (eidos, engine, my-llm), plans
transitions in dependency order, and verifies claudony fleet API
receives register calls via WireMock.

**Architecture:** Extend `topology-tests` with a `DeploymentTopologyTestBase`
that registers deployment node types via `DeploymentNodeSpecFactoryProvider`.
YAML files in `topologies/deployment/` are auto-discovered by parameterized
tests. Claudony integration uses `HttpClaudonyFleetClient` → WireMock.

**Tech Stack:** JUnit 5, AssertJ, WireMock 3.x, Jackson YAML,
casehub-desiredstate-yaml, casehub-ops-deployment, casehub-ops-claudony

## Global Constraints

- Java 21, Maven, no CDI/Quarkus in topology-tests (plain JUnit 5)
- All tests must run offline (`mvn -o test`)
- Follow existing topology-tests patterns (TopologyTestBase, ReconciliationTestBase)
- YAML format: generic desiredstate format (`desiredState: / nodes:` with `type: casehub_application`)
- No new production code — only test code and test resources

---

## Batch 1: Compilation — YAML compiles with deployment types

### Task 1: Add deployment + claudony + WireMock dependencies to topology-tests

**Files:**
- Modify: `topology-tests/pom.xml`

**Interfaces:**
- Produces: deployment, claudony, and WireMock on the test classpath

- [ ] **Step 1: Write the failing test**

Verify the dependency is needed by checking that `CaseHubApplicationSpec`
is not currently resolvable from topology-tests:

```bash
mvn --batch-mode -o compile -pl topology-tests
```

Expected: compiles (no new code yet)

- [ ] **Step 2: Add dependencies to topology-tests/pom.xml**

Add after the existing `casehub-ops-infra` dependency:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-ops-deployment</artifactId>
    <version>${project.version}</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-ops-claudony</artifactId>
    <version>${project.version}</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.wiremock</groupId>
    <artifactId>wiremock-standalone</artifactId>
    <version>3.13.0</version>
    <scope>test</scope>
</dependency>
```

- [ ] **Step 3: Verify dependencies resolve**

```bash
mvn --batch-mode -o compile -pl topology-tests
```

Expected: PASS — if WireMock isn't cached locally, run without `-o` first.

- [ ] **Step 4: Commit**

```bash
git add topology-tests/pom.xml
git commit -m "feat(topology-tests): add deployment, claudony, and WireMock dependencies

Refs #<N>"
```

---

### Task 2: Extend TopologyTestBase to support configurable factory providers

**Files:**
- Modify: `topology-tests/src/test/java/io/casehub/ops/topology/compilation/TopologyTestBase.java`

**Interfaces:**
- Consumes: existing `compileExemplar(String resourcePath)` (passes only `InfraNodeSpecFactoryProvider`)
- Produces: new overload `compileExemplar(String resourcePath, List<String> factoryProviderClassNames)` that accepts arbitrary factory providers

- [ ] **Step 1: Write the failing test**

Create a test that calls the new overload (it doesn't exist yet):

```java
// In a scratch test or just verify compilation fails
TopologyTestBase.compileExemplar("topologies/deployment/01-hello-llm.yaml",
    List.of(DeploymentNodeSpecFactoryProvider.class.getName()));
```

Expected: does not compile — method doesn't exist.

- [ ] **Step 2: Add the overloaded method**

In `TopologyTestBase.java`, add a new method that accepts factory provider
class names. Refactor the existing `compileExemplar(String)` to delegate
to the new one with `InfraNodeSpecFactoryProvider`:

```java
public static GoalCompiler<Void> compileExemplar(String resourcePath,
                                                  List<String> factoryProviderClassNames) throws IOException {
    Map<String, String>     typeRegistry = buildTypeRegistry(factoryProviderClassNames);
    YamlGraph               yamlGraph    = parseYaml(resourcePath);
    Map<String, YamlModule> modules      = discoverModules();

    List<ResolvedInvariant> invariants = new ArrayList<>();
    for (var inv : yamlGraph.invariants().entrySet()) {
        invariants.add(YamlInvariantConverter.toDeclarativeInvariant(inv.getKey(), inv.getValue()));
    }

    YamlGraphRecorder recorder = new YamlGraphRecorder();

    if (yamlGraph.lifecycle() != null) {
        return recorder.createYamlLifecycleGoalCompiler(
                yamlGraph, typeRegistry,
                yamlGraph.variables() != null ? yamlGraph.variables() : Map.of(),
                invariants, modules, factoryProviderClassNames).getValue();
    }

    GraphDescriptor descriptor = toGraphDescriptor(yamlGraph, typeRegistry);
    return recorder.createYamlGoalCompiler(
            descriptor, typeRegistry,
            yamlGraph.variables() != null ? yamlGraph.variables() : Map.of(),
            invariants, yamlGraph, modules,
            List.of(), List.of(), factoryProviderClassNames).getValue();
}
```

Also add a `buildTypeRegistry(List<String> factoryProviderClassNames)` that
instantiates each provider and collects their type mappings:

```java
public static Map<String, String> buildTypeRegistry(List<String> factoryProviderClassNames) {
    Map<String, String> registry = buildTypeRegistry(); // infra types
    for (String className : factoryProviderClassNames) {
        try {
            NodeSpecFactoryProvider provider =
                (NodeSpecFactoryProvider) Class.forName(className)
                    .getDeclaredConstructor().newInstance();
            for (var entry : provider.provide().entrySet()) {
                // Map type name to the spec class produced by the factory
                registry.putIfAbsent(entry.getKey(), entry.getKey());
            }
        } catch (ReflectiveOperationException e) {
            throw new RuntimeException("Cannot instantiate " + className, e);
        }
    }
    return registry;
}
```

Update the original `compileExemplar(String)` to delegate:

```java
public static GoalCompiler<Void> compileExemplar(String resourcePath) throws IOException {
    return compileExemplar(resourcePath,
        List.of(InfraNodeSpecFactoryProvider.class.getName()));
}
```

- [ ] **Step 3: Verify existing infra tests still pass**

```bash
mvn --batch-mode -o test -pl topology-tests
```

Expected: all existing tests PASS (no behavioral change for infra types).

- [ ] **Step 4: Commit**

```bash
git add topology-tests/src/test/java/io/casehub/ops/topology/compilation/TopologyTestBase.java
git commit -m "refactor(topology-tests): add configurable factory providers to TopologyTestBase

Refs #<N>"
```

---

### Task 3: Create 01-hello-llm.yaml and INDEX.md

**Files:**
- Create: `topology-tests/src/test/resources/topologies/deployment/01-hello-llm.yaml`
- Create: `topology-tests/src/test/resources/topologies/deployment/INDEX.md`

**Interfaces:**
- Produces: YAML file at classpath `topologies/deployment/01-hello-llm.yaml`

- [ ] **Step 1: Create the YAML topology file**

```yaml
# ============================================================
# Level 01: Hello LLM
# ============================================================
# The minimum viable CaseHub app. Deploys three platform services
# via claudony: eidos (knowledge), engine (execution runtime),
# and an LLM app that uses them.
#
# Prerequisites: None — this is where you start.
#
# What you'll learn:
#   - How to declare CaseHub apps in YAML
#   - How dependency edges control deployment order
#   - How claudony registers and manages your apps
#
# After deployment, the reconciliation loop ensures:
#   1. eidos starts first (no dependencies)
#   2. engine starts after eidos
#   3. my-llm starts after engine
#
# Try it: change replicas or config values and re-reconcile.
# Only the changed nodes will be updated.
# ============================================================

desiredState:
  namespace: casehub
  name: hello-llm

nodes:
  eidos:
    type: casehub_application
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
    spec:
      applicationId: my-llm
      image: casehubio/my-llm-app:latest
      deploymentTarget: claudony
      replicas: 1
      config:
        llm.provider: anthropic
        llm.model: claude-sonnet-4-20250514
```

- [ ] **Step 2: Create INDEX.md**

```markdown
# Deployment Topology Tutorials

YAML-driven integration tests demonstrating CaseHub app deployment
via the desiredstate reconciliation loop. Each level adds a platform
capability on top of a working core.

## Levels

| # | File | What it teaches |
|---|------|-----------------|
| 01 | [01-hello-llm.yaml](01-hello-llm.yaml) | Deploy a complete LLM app — eidos + engine + LLM via claudony |
```

- [ ] **Step 3: Commit**

```bash
git add topology-tests/src/test/resources/topologies/deployment/
git commit -m "feat(topology-tests): add 01-hello-llm.yaml deployment topology

Tutorial-style YAML declaring eidos + engine + LLM app with
dependency ordering, deployed via claudony.

Refs #<N>"
```

---

### Task 4: Create DeploymentCompilationTest

**Files:**
- Create: `topology-tests/src/test/java/io/casehub/ops/topology/compilation/DeploymentCompilationTest.java`

**Interfaces:**
- Consumes: `TopologyTestBase.compileExemplar(String, List<String>)` from Task 2
- Consumes: `01-hello-llm.yaml` from Task 3

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.ops.topology.compilation;

import io.casehub.desiredstate.api.DesiredStateGraph;
import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.api.NodeType;
import io.casehub.ops.deployment.DeploymentNodeSpecFactoryProvider;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.MethodSource;

import java.io.IOException;
import java.io.InputStream;
import java.util.ArrayList;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class DeploymentCompilationTest extends TopologyTestBase {

    private static final List<String> DEPLOYMENT_PROVIDERS =
            List.of(DeploymentNodeSpecFactoryProvider.class.getName());

    static List<String> discoverDeploymentTopologies() throws IOException {
        List<String> paths = new ArrayList<>();
        // Scan classpath for topologies/deployment/*.yaml
        String dir = "topologies/deployment/";
        try (InputStream is = DeploymentCompilationTest.class.getClassLoader()
                .getResourceAsStream(dir)) {
            // Fallback: enumerate known files
        }
        // For now, list known files explicitly — auto-discovery
        // via classpath scanning requires additional infrastructure
        paths.add("topologies/deployment/01-hello-llm.yaml");
        return paths;
    }

    @ParameterizedTest
    @MethodSource("discoverDeploymentTopologies")
    void compilesWithoutErrors(String yamlPath) throws IOException {
        DesiredStateGraph graph = compileSingleGraph(yamlPath, DEPLOYMENT_PROVIDERS);
        assertThat(graph.nodes()).isNotEmpty();
    }

    @Test
    void helloLlm_hasThreeNodes() throws IOException {
        DesiredStateGraph graph = compileSingleGraph(
                "topologies/deployment/01-hello-llm.yaml", DEPLOYMENT_PROVIDERS);
        assertThat(graph.nodes()).hasSize(3);
    }

    @Test
    void helloLlm_allNodesAreCasehubApplication() throws IOException {
        DesiredStateGraph graph = compileSingleGraph(
                "topologies/deployment/01-hello-llm.yaml", DEPLOYMENT_PROVIDERS);
        graph.nodes().values().forEach(node ->
            assertThat(node.type()).isEqualTo(NodeType.of("casehub_application")));
    }

    @Test
    void helloLlm_engineDependsOnEidos() throws IOException {
        DesiredStateGraph graph = compileSingleGraph(
                "topologies/deployment/01-hello-llm.yaml", DEPLOYMENT_PROVIDERS);
        assertThat(graph.dependenciesOf(NodeId.of("engine")))
                .contains(NodeId.of("eidos"));
    }

    @Test
    void helloLlm_llmDependsOnEngine() throws IOException {
        DesiredStateGraph graph = compileSingleGraph(
                "topologies/deployment/01-hello-llm.yaml", DEPLOYMENT_PROVIDERS);
        assertThat(graph.dependenciesOf(NodeId.of("my-llm")))
                .contains(NodeId.of("engine"));
    }
}
```

- [ ] **Step 2: Add compileSingleGraph overload to TopologyTestBase**

The new test calls `compileSingleGraph(path, providers)` which doesn't exist yet.
Add to `TopologyTestBase`:

```java
public static DesiredStateGraph compileSingleGraph(String resourcePath,
                                                    List<String> factoryProviderClassNames) throws IOException {
    GoalCompiler<Void> compiler = compileExemplar(resourcePath, factoryProviderClassNames);
    CompilationResult  result   = compiler.compile(null, FACTORY);
    assertThat(result).isInstanceOf(CompilationResult.SingleGraph.class);
    return ((CompilationResult.SingleGraph) result).graph();
}
```

- [ ] **Step 3: Run tests**

```bash
mvn --batch-mode -o test -pl topology-tests -Dtest=DeploymentCompilationTest
```

Expected: all tests PASS — YAML compiles, 3 nodes, all `casehub_application`,
correct dependency edges.

- [ ] **Step 4: Verify existing infra tests still pass**

```bash
mvn --batch-mode -o test -pl topology-tests
```

Expected: all tests PASS.

- [ ] **Step 5: Commit**

```bash
git add topology-tests/src/test/java/io/casehub/ops/topology/compilation/DeploymentCompilationTest.java
git add topology-tests/src/test/java/io/casehub/ops/topology/compilation/TopologyTestBase.java
git commit -m "feat(topology-tests): DeploymentCompilationTest for 01-hello-llm

Data-driven compilation test for deployment topologies. Verifies
01-hello-llm.yaml compiles to 3 casehub_application nodes with
correct dependency ordering (eidos → engine → my-llm).

Refs #<N>"
```

---

## Batch 2: Reconciliation + Claudony verification

### Task 5: Create DeploymentReconciliationTest

**Files:**
- Create: `topology-tests/src/test/java/io/casehub/ops/topology/reconciliation/DeploymentReconciliationTest.java`
- Modify: `topology-tests/src/test/java/io/casehub/ops/topology/reconciliation/ReconciliationTestBase.java`

**Interfaces:**
- Consumes: `TopologyTestBase.compileSingleGraph(String, List<String>)` from Task 4
- Consumes: `ReconciliationTestBase.planFromEmpty()`, `executeTransition()`, `assertOrderedBefore()`

- [ ] **Step 1: Extend ReconciliationTestBase to support deployment types**

The `setUpReconciliation()` method currently calls `allInfraTypes()` to
configure the provisioner and adapter. Add a method for deployment types:

```java
protected static Set<NodeType> allDeploymentTypes() {
    Set<NodeType> types = new HashSet<>();
    types.add(NodeType.of("casehub_application"));
    return Set.copyOf(types);
}

protected void setUpDeploymentReconciliation() {
    provisioner = new FailableNodeProvisioner();
    Set<NodeType> allTypes = new HashSet<>(allInfraTypes());
    allTypes.addAll(allDeploymentTypes());
    provisioner.setHandledTypes(allTypes);
    planner = new TransitionPlanner();
    actualAdapter = new MockActualStateAdapter();
    actualAdapter.setHandledTypes(allTypes);
}
```

Make `allInfraTypes()` `protected` (currently `private`).

- [ ] **Step 2: Write the failing test**

```java
package io.casehub.ops.topology.reconciliation;

import io.casehub.desiredstate.api.DesiredStateGraph;
import io.casehub.desiredstate.api.TransitionPlan;
import io.casehub.desiredstate.api.TransitionResult;
import io.casehub.ops.deployment.DeploymentNodeSpecFactoryProvider;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;

import java.io.IOException;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

@Tag("reconciliation")
class DeploymentReconciliationTest extends ReconciliationTestBase {

    private static final List<String> DEPLOYMENT_PROVIDERS =
            List.of(DeploymentNodeSpecFactoryProvider.class.getName());

    @BeforeEach
    void setUp() {
        setUpDeploymentReconciliation();
    }

    @Test
    void helloLlm_plansThreeAdditions() throws IOException {
        DesiredStateGraph graph = compileSingleGraph(
                "topologies/deployment/01-hello-llm.yaml", DEPLOYMENT_PROVIDERS);
        TransitionPlan plan = planFromEmpty(graph);
        assertThat(plan.additions()).hasSize(3);
        assertThat(plan.removals()).isEmpty();
    }

    @Test
    void helloLlm_eidosBeforeEngine() throws IOException {
        DesiredStateGraph graph = compileSingleGraph(
                "topologies/deployment/01-hello-llm.yaml", DEPLOYMENT_PROVIDERS);
        TransitionPlan plan = planFromEmpty(graph);
        assertOrderedBefore(plan, "eidos", "engine");
    }

    @Test
    void helloLlm_engineBeforeLlm() throws IOException {
        DesiredStateGraph graph = compileSingleGraph(
                "topologies/deployment/01-hello-llm.yaml", DEPLOYMENT_PROVIDERS);
        TransitionPlan plan = planFromEmpty(graph);
        assertOrderedBefore(plan, "engine", "my-llm");
    }

    @Test
    void helloLlm_executesSuccessfully() throws IOException {
        DesiredStateGraph graph = compileSingleGraph(
                "topologies/deployment/01-hello-llm.yaml", DEPLOYMENT_PROVIDERS);
        TransitionPlan plan = planFromEmpty(graph);
        TransitionResult result = executeTransition(plan, "test-tenant");
        assertThat(result.allSucceeded()).isTrue();
    }

    @Test
    void helloLlm_idempotentAfterExecution() throws IOException {
        DesiredStateGraph graph = compileSingleGraph(
                "topologies/deployment/01-hello-llm.yaml", DEPLOYMENT_PROVIDERS);
        TransitionPlan plan = planFromEmpty(graph);
        executeTransition(plan, "test-tenant");

        // After execution, re-plan with actual state showing all nodes present
        actualAdapter.setAllPresent(graph);
        TransitionPlan replan = planWithActual(graph, actualAdapter.readActual(graph, "test-tenant"));
        assertThat(replan.additions()).isEmpty();
        assertThat(replan.removals()).isEmpty();
    }
}
```

- [ ] **Step 3: Run the reconciliation test**

```bash
mvn --batch-mode -o test -pl topology-tests -Dtest=DeploymentReconciliationTest -Dgroups=reconciliation
```

Expected: all tests PASS — plans 3 additions in dependency order, executes
successfully, idempotent re-plan produces no transitions.

- [ ] **Step 4: Verify existing tests still pass**

```bash
mvn --batch-mode -o test -pl topology-tests
mvn --batch-mode -o test -pl topology-tests -Preconciliation
```

Expected: all existing tests PASS.

- [ ] **Step 5: Commit**

```bash
git add topology-tests/src/test/java/io/casehub/ops/topology/reconciliation/
git commit -m "feat(topology-tests): DeploymentReconciliationTest for 01-hello-llm

Reconciliation tests verifying dependency-ordered provisioning
(eidos → engine → my-llm), successful execution, and idempotent
re-planning for the hello-llm deployment topology.

Refs #<N>"
```

---

### Task 6: Claudony WireMock integration test

**Files:**
- Create: `topology-tests/src/test/java/io/casehub/ops/topology/deployment/ClaudonyWireMockSupport.java`
- Create: `topology-tests/src/test/java/io/casehub/ops/topology/deployment/DeploymentTargetIntegrationTest.java`

**Interfaces:**
- Consumes: `TopologyTestBase.compileSingleGraph(String, List<String>)`
- Consumes: `ClaudonyDeploymentTarget`, `HttpClaudonyFleetClient` from claudony module
- Consumes: `DeploymentApplicationNodeSpec` from api module

- [ ] **Step 1: Create ClaudonyWireMockSupport**

Utility class that configures WireMock stubs for the claudony fleet API:

```java
package io.casehub.ops.topology.deployment;

import com.github.tomakehurst.wiremock.WireMockServer;
import com.github.tomakehurst.wiremock.client.WireMock;
import io.casehub.platform.api.registry.HealthStatus;

import static com.github.tomakehurst.wiremock.client.WireMock.*;

public final class ClaudonyWireMockSupport {

    private ClaudonyWireMockSupport() {}

    public static void stubRegisterApp(WireMockServer server) {
        server.stubFor(post(urlEqualTo("/api/claudony/apps/"))
                .willReturn(aResponse()
                        .withStatus(201)
                        .withHeader("Content-Type", "application/json")
                        .withBody("""
                            {"name":"stub","health":"HEALTHY","registeredAt":"2026-01-01T00:00:00Z"}
                            """)));
    }

    public static void stubGetAppHealthy(WireMockServer server, String appName) {
        server.stubFor(get(urlEqualTo("/api/claudony/apps/" + appName))
                .willReturn(aResponse()
                        .withStatus(200)
                        .withHeader("Content-Type", "application/json")
                        .withBody("""
                            {"name":"%s","health":"HEALTHY","registeredAt":"2026-01-01T00:00:00Z"}
                            """.formatted(appName))));
    }
}
```

- [ ] **Step 2: Write the failing test**

```java
package io.casehub.ops.topology.deployment;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.github.tomakehurst.wiremock.WireMockServer;
import com.github.tomakehurst.wiremock.core.WireMockConfiguration;
import io.casehub.desiredstate.api.DesiredStateGraph;
import io.casehub.desiredstate.api.DesiredNode;
import io.casehub.desiredstate.api.NodeStatus;
import io.casehub.ops.api.deployment.CaseHubApplicationSpec;
import io.casehub.ops.api.deployment.DeployConfig;
import io.casehub.ops.api.deployment.DeployResult;
import io.casehub.ops.api.deployment.DeploymentApplicationNodeSpec;
import io.casehub.ops.claudony.ClaudonyDeploymentTarget;
import io.casehub.ops.claudony.HttpClaudonyFleetClient;
import io.casehub.ops.deployment.DeploymentNodeSpecFactoryProvider;
import io.casehub.ops.topology.compilation.TopologyTestBase;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;

import java.io.IOException;
import java.util.List;
import java.util.Map;

import static com.github.tomakehurst.wiremock.client.WireMock.*;
import static org.assertj.core.api.Assertions.assertThat;

@Tag("reconciliation")
class DeploymentTargetIntegrationTest extends TopologyTestBase {

    private static final List<String> DEPLOYMENT_PROVIDERS =
            List.of(DeploymentNodeSpecFactoryProvider.class.getName());

    private WireMockServer wireMock;
    private ClaudonyDeploymentTarget claudonyTarget;

    @BeforeEach
    void setUp() {
        wireMock = new WireMockServer(WireMockConfiguration.wireMockConfig().dynamicPort());
        wireMock.start();

        ClaudonyWireMockSupport.stubRegisterApp(wireMock);
        ClaudonyWireMockSupport.stubGetAppHealthy(wireMock, "eidos");
        ClaudonyWireMockSupport.stubGetAppHealthy(wireMock, "engine");
        ClaudonyWireMockSupport.stubGetAppHealthy(wireMock, "my-llm");

        HttpClaudonyFleetClient fleetClient = new HttpClaudonyFleetClient(
                wireMock.baseUrl(), new ObjectMapper());
        claudonyTarget = new ClaudonyDeploymentTarget(fleetClient);
    }

    @AfterEach
    void tearDown() {
        wireMock.stop();
    }

    @Test
    void helloLlm_deploysAllAppsViaClaudony() throws IOException {
        DesiredStateGraph graph = compileSingleGraph(
                "topologies/deployment/01-hello-llm.yaml", DEPLOYMENT_PROVIDERS);

        for (DesiredNode node : graph.nodes().values()) {
            CaseHubApplicationSpec appSpec = extractAppSpec(node);
            DeployResult result = claudonyTarget.deploy(
                    appSpec.image(),
                    new DeployConfig(appSpec.applicationId(), appSpec.replicas(),
                            appSpec.config(), Map.of()));
            assertThat(result).isInstanceOf(DeployResult.Success.class);
        }

        wireMock.verify(3, postRequestedFor(urlEqualTo("/api/claudony/apps/")));
    }

    @Test
    void helloLlm_allAppsReportHealthy() throws IOException {
        DesiredStateGraph graph = compileSingleGraph(
                "topologies/deployment/01-hello-llm.yaml", DEPLOYMENT_PROVIDERS);

        for (DesiredNode node : graph.nodes().values()) {
            CaseHubApplicationSpec appSpec = extractAppSpec(node);
            NodeStatus status = claudonyTarget.status(appSpec.applicationId());
            assertThat(status).isEqualTo(NodeStatus.PRESENT);
        }
    }

    private CaseHubApplicationSpec extractAppSpec(DesiredNode node) {
        if (node.spec() instanceof DeploymentApplicationNodeSpec dns) {
            return dns.spec();
        }
        return (CaseHubApplicationSpec) node.spec();
    }
}
```

- [ ] **Step 3: Run the test**

```bash
mvn --batch-mode test -pl topology-tests -Dtest=DeploymentTargetIntegrationTest -Dgroups=reconciliation
```

Expected: all tests PASS — WireMock receives 3 register calls, all apps
report HEALTHY status.

Note: may need to run without `-o` first if WireMock jar not cached.

- [ ] **Step 4: Verify all tests still pass**

```bash
mvn --batch-mode -o test -pl topology-tests
mvn --batch-mode -o test -pl topology-tests -Preconciliation
```

Expected: all tests PASS.

- [ ] **Step 5: Commit**

```bash
git add topology-tests/src/test/java/io/casehub/ops/topology/deployment/
git commit -m "feat(topology-tests): claudony WireMock integration test

Verifies 01-hello-llm topology deploys all 3 apps through
ClaudonyDeploymentTarget → WireMock, and all report HEALTHY.

Refs #<N>"
```

---

## Post-Implementation

After both batches are complete:

1. Run the full topology-tests suite to verify no regressions
2. Update the spec document with any deviations discovered during implementation
3. Assess readiness for level 02 (+ ledger + work)

## References

- [2026-10-14-deployment-topology-integration-tests-design.md] — design spec this plan implements
- `topology-tests/src/test/java/io/casehub/ops/topology/compilation/TopologyTestBase.java:30-133` — existing test base
- `topology-tests/src/test/java/io/casehub/ops/topology/reconciliation/ReconciliationTestBase.java:30-98` — reconciliation base
- `deployment/src/main/java/io/casehub/ops/deployment/DeploymentNodeSpecFactoryProvider.java:13-32` — type registry
- `claudony/src/main/java/io/casehub/ops/claudony/ClaudonyDeploymentTarget.java:18-135` — deployment target
- `claudony/src/main/java/io/casehub/ops/claudony/HttpClaudonyFleetClient.java:18-185` — REST client
- `app/src/main/resources/META-INF/desiredstate/casehub-topology.yaml` — production topology format
- `api/src/main/java/io/casehub/ops/api/deployment/CaseHubApplicationSpec.java:12-37` — app spec record
