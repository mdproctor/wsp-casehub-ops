# CaseHub Application NodeSpec + NodeProvisioner Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #83 — feat: casehub-application NodeSpec + NodeProvisioner
**Issue group:** #83

**Goal:** Add a `casehub_application` node type to the deployment domain's sealed hierarchy so YAML-first case applications compose with existing topology nodes (load-balancer, HA, service-mesh) in the desiredstate graph.

**Architecture:** Extends `DeploymentNodeSpec` with `CaseHubApplicationSpec`. Two SPIs abstract the pipeline: `ApplicationBuildPipeline` (source → image ref, stubbed until engine#1018/#1019) and `ApplicationDeploymentTarget` (image ref → running container). The provisioner handler orchestrates both SPIs; the drift checker queries container status.

**Tech Stack:** Java 21, Quarkus (CDI), JUnit 5, AssertJ, Jackson

## Global Constraints

- Java 21, `-parameters` compiler flag
- No Quarkus extension — Jandex library, CDI classpath discovery
- Records for value types, sealed interfaces for type hierarchies
- `@JsonIgnoreProperties(ignoreUnknown = true)` on all Jackson-deserialized records
- Defensive copies in compact constructors: `List.copyOf()`, `Map.copyOf()`
- Test scope: JUnit 5 + AssertJ, plain unit tests with stubs (no `@QuarkusTest` for unit tests)
- Build: `mvn --batch-mode -o test -pl api,deployment,testing`

---

## Batch 1: API Types and SPI Contracts

### Task 1: CaseHubApplicationSpec + RuntimeConfig + sealed hierarchy + DeploymentGoals

**Files:**
- Create: `api/src/main/java/io/casehub/ops/api/deployment/RuntimeConfig.java`
- Create: `api/src/main/java/io/casehub/ops/api/deployment/CaseHubApplicationSpec.java`
- Modify: `api/src/main/java/io/casehub/ops/api/deployment/DeploymentNodeSpec.java:6-7` (add permit)
- Modify: `api/src/main/java/io/casehub/ops/api/deployment/DeploymentGoals.java:7-27` (add applications field)
- Create: `api/src/test/java/io/casehub/ops/api/deployment/RuntimeConfigTest.java`
- Create: `api/src/test/java/io/casehub/ops/api/deployment/CaseHubApplicationSpecTest.java`

**Interfaces:**
- Consumes: `DeploymentNodeSpec` (sealed interface), `NodeSpec` (desiredstate-api), `NodeType`, `NodeTypeId`
- Produces: `RuntimeConfig(engine, modules)`, `CaseHubApplicationSpec(applicationId, source, image, runtime, config, dependencies, replicas)` implementing `DeploymentNodeSpec`

- [ ] **Step 1: Write RuntimeConfig test**

```java
package io.casehub.ops.api.deployment;

import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.*;

class RuntimeConfigTest {

    @Test
    void validConfig() {
        var config = new RuntimeConfig("0.2", List.of("a2a", "react"));
        assertThat(config.engine()).isEqualTo("0.2");
        assertThat(config.modules()).containsExactly("a2a", "react");
    }

    @Test
    void nullEngineThrows() {
        assertThatIllegalArgumentException()
            .isThrownBy(() -> new RuntimeConfig(null, List.of()))
            .withMessageContaining("engine");
    }

    @Test
    void blankEngineThrows() {
        assertThatIllegalArgumentException()
            .isThrownBy(() -> new RuntimeConfig("  ", List.of()))
            .withMessageContaining("engine");
    }

    @Test
    void nullModulesDefaultsToEmpty() {
        var config = new RuntimeConfig("0.2", null);
        assertThat(config.modules()).isEmpty();
    }

    @Test
    void modulesAreDefensivelyCopied() {
        var mutable = new java.util.ArrayList<>(List.of("a2a"));
        var config = new RuntimeConfig("0.2", mutable);
        assertThatThrownBy(() -> config.modules().add("react"))
            .isInstanceOf(UnsupportedOperationException.class);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode -o test -pl api -Dtest=RuntimeConfigTest`
Expected: compilation failure — `RuntimeConfig` does not exist.

- [ ] **Step 3: Implement RuntimeConfig**

Create `api/src/main/java/io/casehub/ops/api/deployment/RuntimeConfig.java`:

```java
package io.casehub.ops.api.deployment;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
import java.util.List;

@JsonIgnoreProperties(ignoreUnknown = true)
public record RuntimeConfig(
    String engine,
    List<String> modules
) {
    public RuntimeConfig {
        if (engine == null || engine.isBlank())
            throw new IllegalArgumentException("engine version is required");
        modules = modules != null ? List.copyOf(modules) : List.of();
    }
}
```

- [ ] **Step 4: Run RuntimeConfigTest — verify pass**

Run: `mvn --batch-mode -o test -pl api -Dtest=RuntimeConfigTest`
Expected: all 5 tests PASS.

- [ ] **Step 5: Write CaseHubApplicationSpec test**

```java
package io.casehub.ops.api.deployment;

import io.casehub.desiredstate.api.NodeType;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import static org.assertj.core.api.Assertions.*;

class CaseHubApplicationSpecTest {

    @Test
    void validSpecWithImage() {
        var spec = new CaseHubApplicationSpec(
            "incident-app", null, "ghcr.io/acme/incident:1.0",
            new RuntimeConfig("0.2", List.of("a2a")),
            Map.of("llm", "anthropic"), List.of(), 2);
        assertThat(spec.applicationId()).isEqualTo("incident-app");
        assertThat(spec.nodeId()).isEqualTo("incident-app");
        assertThat(spec.nodeType()).isEqualTo(NodeType.of("casehub_application"));
        assertThat(spec.image()).isEqualTo("ghcr.io/acme/incident:1.0");
        assertThat(spec.source()).isNull();
        assertThat(spec.replicas()).isEqualTo(2);
    }

    @Test
    void validSpecWithSource() {
        var spec = new CaseHubApplicationSpec(
            "loan-app", "./apps/loan-processing", null,
            new RuntimeConfig("0.2", List.of("mcp")),
            Map.of(), List.of(), 1);
        assertThat(spec.source()).isEqualTo("./apps/loan-processing");
        assertThat(spec.image()).isNull();
    }

    @Test
    void nullApplicationIdThrows() {
        assertThatIllegalArgumentException()
            .isThrownBy(() -> new CaseHubApplicationSpec(
                null, "./apps/test", null,
                new RuntimeConfig("0.2", List.of()),
                Map.of(), List.of(), 1))
            .withMessageContaining("applicationId");
    }

    @Test
    void blankApplicationIdThrows() {
        assertThatIllegalArgumentException()
            .isThrownBy(() -> new CaseHubApplicationSpec(
                "  ", "./apps/test", null,
                new RuntimeConfig("0.2", List.of()),
                Map.of(), List.of(), 1))
            .withMessageContaining("applicationId");
    }

    @Test
    void neitherSourceNorImageThrows() {
        assertThatIllegalArgumentException()
            .isThrownBy(() -> new CaseHubApplicationSpec(
                "test-app", null, null,
                new RuntimeConfig("0.2", List.of()),
                Map.of(), List.of(), 1))
            .withMessageContaining("source or image");
    }

    @Test
    void replicasBelowOneDefaultsToOne() {
        var spec = new CaseHubApplicationSpec(
            "test-app", null, "img:1.0",
            new RuntimeConfig("0.2", List.of()),
            Map.of(), List.of(), 0);
        assertThat(spec.replicas()).isEqualTo(1);
    }

    @Test
    void configAndDependenciesDefensivelyCopied() {
        var config = new java.util.HashMap<String, Object>();
        config.put("key", "val");
        var deps = new java.util.ArrayList<>(List.of("com.example:ext:1.0"));
        var spec = new CaseHubApplicationSpec(
            "test-app", null, "img:1.0",
            new RuntimeConfig("0.2", List.of()),
            config, deps, 1);
        assertThatThrownBy(() -> spec.config().put("k2", "v2"))
            .isInstanceOf(UnsupportedOperationException.class);
        assertThatThrownBy(() -> spec.dependencies().add("another"))
            .isInstanceOf(UnsupportedOperationException.class);
    }
}
```

- [ ] **Step 6: Run test to verify it fails**

Run: `mvn --batch-mode -o test -pl api -Dtest=CaseHubApplicationSpecTest`
Expected: compilation failure — `CaseHubApplicationSpec` does not exist.

- [ ] **Step 7: Implement CaseHubApplicationSpec**

Create `api/src/main/java/io/casehub/ops/api/deployment/CaseHubApplicationSpec.java`:

```java
package io.casehub.ops.api.deployment;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
import io.casehub.desiredstate.api.NodeType;
import io.casehub.desiredstate.api.NodeTypeId;

import java.util.List;
import java.util.Map;

@NodeTypeId("casehub_application")
@JsonIgnoreProperties(ignoreUnknown = true)
public record CaseHubApplicationSpec(
    String applicationId,
    String source,
    String image,
    RuntimeConfig runtime,
    Map<String, Object> config,
    List<String> dependencies,
    int replicas
) implements DeploymentNodeSpec {

    public CaseHubApplicationSpec {
        if (applicationId == null || applicationId.isBlank())
            throw new IllegalArgumentException("applicationId is required");
        if (source == null && image == null)
            throw new IllegalArgumentException("source or image is required");
        config = config != null ? Map.copyOf(config) : Map.of();
        dependencies = dependencies != null ? List.copyOf(dependencies) : List.of();
        if (replicas < 1) replicas = 1;
    }

    @Override
    public String nodeId() { return applicationId; }

    @Override
    public NodeType nodeType() { return NodeType.of("casehub_application"); }
}
```

- [ ] **Step 8: Update DeploymentNodeSpec sealed permits**

Use `ide_read_file` to read `DeploymentNodeSpec.java`, then `ide_replace_member` to update the sealed permits list. Add `CaseHubApplicationSpec` as the 8th permit:

```java
public sealed interface DeploymentNodeSpec extends NodeSpec permits
    AgentNodeSpec, ChannelNodeSpec, CaseTypeNodeSpec, TrustPolicyNodeSpec,
    EndpointNodeSpec, DetectionNodeSpec, PoolNodeSpec, CaseHubApplicationSpec {
```

- [ ] **Step 9: Update DeploymentGoals — add applications field**

Use `ide_replace_member` on `DeploymentGoals` to add the `applications` field. The updated record:

```java
@JsonIgnoreProperties(ignoreUnknown = true)
public record DeploymentGoals(
        List<GoalEntry<AgentNodeSpec>> agents,
        List<GoalEntry<ChannelNodeSpec>> channels,
        List<GoalEntry<CaseTypeNodeSpec>> caseTypes,
        List<GoalEntry<TrustPolicyNodeSpec>> trust,
        List<GoalEntry<EndpointNodeSpec>> endpoints,
        List<GoalEntry<DetectionNodeSpec>> detections,
        List<GoalEntry<PoolNodeSpec>> pools,
        List<GoalEntry<CaseHubApplicationSpec>> applications,
        List<AdaptationRuleSpec> adaptations
) {
    public DeploymentGoals {
        agents = agents != null ? List.copyOf(agents) : List.of();
        channels = channels != null ? List.copyOf(channels) : List.of();
        caseTypes = caseTypes != null ? List.copyOf(caseTypes) : List.of();
        trust = trust != null ? List.copyOf(trust) : List.of();
        endpoints = endpoints != null ? List.copyOf(endpoints) : List.of();
        detections = detections != null ? List.copyOf(detections) : List.of();
        pools = pools != null ? List.copyOf(pools) : List.of();
        applications = applications != null ? List.copyOf(applications) : List.of();
        adaptations = adaptations != null ? List.copyOf(adaptations) : List.of();
    }
}
```

- [ ] **Step 10: Run all CaseHubApplicationSpec tests — verify pass**

Run: `mvn --batch-mode -o test -pl api -Dtest=CaseHubApplicationSpecTest,RuntimeConfigTest`
Expected: all tests PASS.

- [ ] **Step 11: Fix any existing tests broken by DeploymentGoals signature change**

Run: `mvn --batch-mode -o test -pl api,deployment`

Any existing tests constructing `DeploymentGoals` will fail because the record now has 9 fields instead of 8. Fix each call site to add the `applications` parameter (typically `List.of()`).

Check with `ide_find_references` on `DeploymentGoals` to find all constructors.

- [ ] **Step 12: Commit**

```bash
git add api/src/main/java/io/casehub/ops/api/deployment/RuntimeConfig.java \
       api/src/main/java/io/casehub/ops/api/deployment/CaseHubApplicationSpec.java \
       api/src/test/java/io/casehub/ops/api/deployment/RuntimeConfigTest.java \
       api/src/test/java/io/casehub/ops/api/deployment/CaseHubApplicationSpecTest.java
git add -u  # pick up modified DeploymentNodeSpec, DeploymentGoals, broken test fixes
git commit -m "feat(#83): CaseHubApplicationSpec + RuntimeConfig + sealed hierarchy update"
```

---

### Task 2: SPI Contracts — Build Pipeline + Deployment Target

**Files:**
- Create: `api/src/main/java/io/casehub/ops/api/deployment/ApplicationBuildPipeline.java`
- Create: `api/src/main/java/io/casehub/ops/api/deployment/BuildResult.java`
- Create: `api/src/main/java/io/casehub/ops/api/deployment/ApplicationDeploymentTarget.java`
- Create: `api/src/main/java/io/casehub/ops/api/deployment/DeployConfig.java`
- Create: `api/src/main/java/io/casehub/ops/api/deployment/DeployResult.java`
- Create: `api/src/main/java/io/casehub/ops/api/deployment/HealthCheckResult.java`
- Create: `api/src/main/java/io/casehub/ops/api/deployment/TargetStatus.java`
- Create: `api/src/main/java/io/casehub/ops/api/deployment/PrebuiltImagePipeline.java`
- Create: `api/src/test/java/io/casehub/ops/api/deployment/PrebuiltImagePipelineTest.java`
- Create: `testing/src/main/java/io/casehub/ops/testing/StubDeploymentTarget.java`

**Interfaces:**
- Consumes: `CaseHubApplicationSpec` (from Task 1)
- Produces: `ApplicationBuildPipeline.build(CaseHubApplicationSpec) → BuildResult`, `ApplicationDeploymentTarget.deploy/undeploy/status/healthCheck`, `PrebuiltImagePipeline` (@DefaultBean), `StubDeploymentTarget` (for tests)

- [ ] **Step 1: Write PrebuiltImagePipeline test**

```java
package io.casehub.ops.api.deployment;

import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import static org.assertj.core.api.Assertions.*;

class PrebuiltImagePipelineTest {

    @Test
    void returnsImageRefFromSpec() {
        var pipeline = new PrebuiltImagePipeline();
        var spec = new CaseHubApplicationSpec(
            "app-1", null, "ghcr.io/acme/app:1.0",
            new RuntimeConfig("0.2", List.of()),
            Map.of(), List.of(), 1);
        var result = pipeline.build(spec);
        assertThat(result).isInstanceOf(BuildResult.Success.class);
        assertThat(((BuildResult.Success) result).imageRef())
            .isEqualTo("ghcr.io/acme/app:1.0");
    }

    @Test
    void failsWhenNoImageSet() {
        var pipeline = new PrebuiltImagePipeline();
        var spec = new CaseHubApplicationSpec(
            "app-1", "./apps/test", null,
            new RuntimeConfig("0.2", List.of()),
            Map.of(), List.of(), 1);
        var result = pipeline.build(spec);
        assertThat(result).isInstanceOf(BuildResult.Failed.class);
        assertThat(((BuildResult.Failed) result).reason()).contains("no image");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode -o test -pl api -Dtest=PrebuiltImagePipelineTest`
Expected: compilation failure.

- [ ] **Step 3: Implement SPI interfaces and result types**

Create `api/src/main/java/io/casehub/ops/api/deployment/BuildResult.java`:

```java
package io.casehub.ops.api.deployment;

public sealed interface BuildResult {
    record Success(String imageRef) implements BuildResult {}
    record Failed(String reason) implements BuildResult {}
}
```

Create `api/src/main/java/io/casehub/ops/api/deployment/ApplicationBuildPipeline.java`:

```java
package io.casehub.ops.api.deployment;

public interface ApplicationBuildPipeline {
    BuildResult build(CaseHubApplicationSpec spec);
}
```

Create `api/src/main/java/io/casehub/ops/api/deployment/DeployConfig.java`:

```java
package io.casehub.ops.api.deployment;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
import java.util.Map;

@JsonIgnoreProperties(ignoreUnknown = true)
public record DeployConfig(
    String applicationId,
    int replicas,
    Map<String, Object> config,
    Map<String, String> env
) {
    public DeployConfig {
        config = config != null ? Map.copyOf(config) : Map.of();
        env = env != null ? Map.copyOf(env) : Map.of();
    }
}
```

Create `api/src/main/java/io/casehub/ops/api/deployment/DeployResult.java`:

```java
package io.casehub.ops.api.deployment;

public sealed interface DeployResult {
    record Success(String instanceId) implements DeployResult {}
    record Failed(String reason) implements DeployResult {}
}
```

Create `api/src/main/java/io/casehub/ops/api/deployment/HealthCheckResult.java`:

```java
package io.casehub.ops.api.deployment;

public sealed interface HealthCheckResult {
    record Healthy(String instanceId) implements HealthCheckResult {}
    record Unhealthy(String instanceId, String reason) implements HealthCheckResult {}
    record Unknown(String instanceId) implements HealthCheckResult {}
}
```

Note: `TargetStatus` already exists in `io.casehub.desiredstate.api.TargetStatus`. Use `io.casehub.desiredstate.api.NodeStatus` for status reporting (PRESENT, ABSENT, DRIFTED, UNKNOWN). No new `TargetStatus` type needed — the `ApplicationDeploymentTarget.status()` method returns `NodeStatus` directly.

Create `api/src/main/java/io/casehub/ops/api/deployment/ApplicationDeploymentTarget.java`:

```java
package io.casehub.ops.api.deployment;

import io.casehub.desiredstate.api.NodeStatus;

public interface ApplicationDeploymentTarget {
    DeployResult deploy(String imageRef, DeployConfig config);
    DeployResult undeploy(String instanceId);
    NodeStatus status(String instanceId);
    HealthCheckResult healthCheck(String instanceId);
}
```

- [ ] **Step 4: Implement PrebuiltImagePipeline**

Create `api/src/main/java/io/casehub/ops/api/deployment/PrebuiltImagePipeline.java`:

```java
package io.casehub.ops.api.deployment;

import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;

@ApplicationScoped
@DefaultBean
public class PrebuiltImagePipeline implements ApplicationBuildPipeline {

    @Override
    public BuildResult build(CaseHubApplicationSpec spec) {
        if (spec.image() == null || spec.image().isBlank()) {
            return new BuildResult.Failed(
                "PrebuiltImagePipeline: no image set on spec '" + spec.applicationId()
                + "' — source-based builds require a real ApplicationBuildPipeline implementation");
        }
        return new BuildResult.Success(spec.image());
    }
}
```

- [ ] **Step 5: Implement StubDeploymentTarget in testing module**

Create `testing/src/main/java/io/casehub/ops/testing/StubDeploymentTarget.java`:

```java
package io.casehub.ops.testing;

import io.casehub.desiredstate.api.NodeStatus;
import io.casehub.ops.api.deployment.*;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public class StubDeploymentTarget implements ApplicationDeploymentTarget {

    private final Map<String, String> deployed = new ConcurrentHashMap<>();

    @Override
    public DeployResult deploy(String imageRef, DeployConfig config) {
        String instanceId = config.applicationId() + "-instance";
        deployed.put(instanceId, imageRef);
        return new DeployResult.Success(instanceId);
    }

    @Override
    public DeployResult undeploy(String instanceId) {
        deployed.remove(instanceId);
        return new DeployResult.Success(instanceId);
    }

    @Override
    public NodeStatus status(String instanceId) {
        return deployed.containsKey(instanceId) ? NodeStatus.PRESENT : NodeStatus.ABSENT;
    }

    @Override
    public HealthCheckResult healthCheck(String instanceId) {
        if (deployed.containsKey(instanceId)) {
            return new HealthCheckResult.Healthy(instanceId);
        }
        return new HealthCheckResult.Unknown(instanceId);
    }

    public Map<String, String> deployedInstances() {
        return Map.copyOf(deployed);
    }
}
```

- [ ] **Step 6: Run PrebuiltImagePipelineTest — verify pass**

Run: `mvn --batch-mode -o test -pl api -Dtest=PrebuiltImagePipelineTest`
Expected: all 2 tests PASS.

- [ ] **Step 7: Run full api + testing build**

Run: `mvn --batch-mode -o test -pl api,testing`
Expected: all tests PASS, both modules compile.

- [ ] **Step 8: Commit**

```bash
git add api/src/main/java/io/casehub/ops/api/deployment/ApplicationBuildPipeline.java \
       api/src/main/java/io/casehub/ops/api/deployment/BuildResult.java \
       api/src/main/java/io/casehub/ops/api/deployment/ApplicationDeploymentTarget.java \
       api/src/main/java/io/casehub/ops/api/deployment/DeployConfig.java \
       api/src/main/java/io/casehub/ops/api/deployment/DeployResult.java \
       api/src/main/java/io/casehub/ops/api/deployment/HealthCheckResult.java \
       api/src/main/java/io/casehub/ops/api/deployment/PrebuiltImagePipeline.java \
       api/src/test/java/io/casehub/ops/api/deployment/PrebuiltImagePipelineTest.java \
       testing/src/main/java/io/casehub/ops/testing/StubDeploymentTarget.java
git commit -m "feat(#83): ApplicationBuildPipeline + ApplicationDeploymentTarget SPIs"
```

---

## Batch 2: Deployment Module Wiring

### Task 3: CaseHubAppProvisionHandler + CaseHubAppDriftChecker

**Files:**
- Create: `deployment/src/main/java/io/casehub/ops/deployment/handler/CaseHubAppProvisionHandler.java`
- Create: `deployment/src/main/java/io/casehub/ops/deployment/drift/CaseHubAppDriftChecker.java`
- Create: `deployment/src/test/java/io/casehub/ops/deployment/handler/CaseHubAppProvisionHandlerTest.java`
- Create: `deployment/src/test/java/io/casehub/ops/deployment/drift/CaseHubAppDriftCheckerTest.java`

**Interfaces:**
- Consumes: `CaseHubApplicationSpec`, `ApplicationBuildPipeline`, `ApplicationDeploymentTarget`, `DeployConfig`, `BuildResult`, `DeployResult`, `HealthCheckResult`, `StubDeploymentTarget`, `PrebuiltImagePipeline`, `ProvisionContext`, `DeprovisionContext`, `ProvisionResult`, `DeprovisionResult`, `NodeDriftChecker`, `NodeStatus`, `NodeSpec`
- Produces: `CaseHubAppProvisionHandler.provision(CaseHubApplicationSpec, ProvisionContext) → ProvisionResult`, `CaseHubAppProvisionHandler.deprovision(CaseHubApplicationSpec, DeprovisionContext) → DeprovisionResult`, `CaseHubAppDriftChecker.check(NodeSpec, String) → NodeStatus`, `CaseHubAppDriftChecker.nodeType() → "casehub_application"`

- [ ] **Step 1: Write CaseHubAppProvisionHandler test**

```java
package io.casehub.ops.deployment.handler;

import io.casehub.desiredstate.api.DeprovisionContext;
import io.casehub.desiredstate.api.DeprovisionResult;
import io.casehub.desiredstate.api.ProvisionContext;
import io.casehub.desiredstate.api.ProvisionResult;
import io.casehub.desiredstate.runtime.DefaultDesiredStateGraphFactory;
import io.casehub.ops.api.deployment.*;
import io.casehub.ops.testing.StubDeploymentTarget;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.*;

class CaseHubAppProvisionHandlerTest {

    private PrebuiltImagePipeline buildPipeline;
    private StubDeploymentTarget deployTarget;
    private CaseHubAppProvisionHandler handler;

    @BeforeEach
    void setUp() {
        buildPipeline = new PrebuiltImagePipeline();
        deployTarget = new StubDeploymentTarget();
        handler = new CaseHubAppProvisionHandler(buildPipeline, deployTarget);
    }

    @Test
    void provisionWithPrebuiltImage() {
        var spec = new CaseHubApplicationSpec(
            "incident-app", null, "ghcr.io/acme/incident:1.0",
            new RuntimeConfig("0.2", List.of("a2a")),
            Map.of(), List.of(), 1);
        var graph = new DefaultDesiredStateGraphFactory().empty();
        var result = handler.provision(spec, new ProvisionContext("tenant-1", graph));

        assertThat(result).isInstanceOf(ProvisionResult.Success.class);
        assertThat(deployTarget.deployedInstances()).containsKey("incident-app-instance");
        assertThat(deployTarget.deployedInstances().get("incident-app-instance"))
            .isEqualTo("ghcr.io/acme/incident:1.0");
    }

    @Test
    void provisionFailsWhenBuildFails() {
        var spec = new CaseHubApplicationSpec(
            "source-app", "./apps/test", null,
            new RuntimeConfig("0.2", List.of()),
            Map.of(), List.of(), 1);
        var graph = new DefaultDesiredStateGraphFactory().empty();
        var result = handler.provision(spec, new ProvisionContext("tenant-1", graph));

        assertThat(result).isInstanceOf(ProvisionResult.Failed.class);
        assertThat(((ProvisionResult.Failed) result).reason()).contains("no image");
    }

    @Test
    void deprovisionRemovesInstance() {
        var spec = new CaseHubApplicationSpec(
            "incident-app", null, "ghcr.io/acme/incident:1.0",
            new RuntimeConfig("0.2", List.of()),
            Map.of(), List.of(), 1);
        var graph = new DefaultDesiredStateGraphFactory().empty();

        handler.provision(spec, new ProvisionContext("tenant-1", graph));
        assertThat(deployTarget.deployedInstances()).isNotEmpty();

        var result = handler.deprovision(spec, new DeprovisionContext("tenant-1", graph));
        assertThat(result).isInstanceOf(DeprovisionResult.Success.class);
        assertThat(deployTarget.deployedInstances()).isEmpty();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode -o test -pl deployment -Dtest=CaseHubAppProvisionHandlerTest`
Expected: compilation failure — `CaseHubAppProvisionHandler` does not exist.

- [ ] **Step 3: Implement CaseHubAppProvisionHandler**

Create `deployment/src/main/java/io/casehub/ops/deployment/handler/CaseHubAppProvisionHandler.java`:

```java
package io.casehub.ops.deployment.handler;

import io.casehub.desiredstate.api.DeprovisionContext;
import io.casehub.desiredstate.api.DeprovisionResult;
import io.casehub.desiredstate.api.ProvisionContext;
import io.casehub.desiredstate.api.ProvisionResult;
import io.casehub.ops.api.deployment.*;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

@ApplicationScoped
public class CaseHubAppProvisionHandler {

    private final ApplicationBuildPipeline buildPipeline;
    private final ApplicationDeploymentTarget deployTarget;

    @Inject
    public CaseHubAppProvisionHandler(
            ApplicationBuildPipeline buildPipeline,
            ApplicationDeploymentTarget deployTarget) {
        this.buildPipeline = buildPipeline;
        this.deployTarget = deployTarget;
    }

    public ProvisionResult provision(CaseHubApplicationSpec spec, ProvisionContext context) {
        String imageRef;
        if (spec.image() != null && !spec.image().isBlank()) {
            imageRef = spec.image();
        } else {
            var buildResult = buildPipeline.build(spec);
            if (buildResult instanceof BuildResult.Failed f) {
                return new ProvisionResult.Failed("Build failed: " + f.reason());
            }
            imageRef = ((BuildResult.Success) buildResult).imageRef();
        }

        var deployConfig = new DeployConfig(
            spec.applicationId(), spec.replicas(), spec.config(), Map.of());
        var deployResult = deployTarget.deploy(imageRef, deployConfig);

        if (deployResult instanceof DeployResult.Failed f) {
            return new ProvisionResult.Failed("Deploy failed: " + f.reason());
        }

        var instanceId = ((DeployResult.Success) deployResult).instanceId();
        var health = deployTarget.healthCheck(instanceId);
        if (health instanceof HealthCheckResult.Unhealthy u) {
            return new ProvisionResult.Failed("Health check failed: " + u.reason());
        }

        return new ProvisionResult.Success();
    }

    public DeprovisionResult deprovision(CaseHubApplicationSpec spec, DeprovisionContext context) {
        var instanceId = spec.applicationId() + "-instance";
        var result = deployTarget.undeploy(instanceId);
        if (result instanceof DeployResult.Failed f) {
            return new DeprovisionResult.Failed("Undeploy failed: " + f.reason());
        }
        return new DeprovisionResult.Success();
    }
}
```

- [ ] **Step 4: Run CaseHubAppProvisionHandlerTest — verify pass**

Run: `mvn --batch-mode -o test -pl deployment -Dtest=CaseHubAppProvisionHandlerTest`
Expected: all 3 tests PASS.

- [ ] **Step 5: Write CaseHubAppDriftChecker test**

```java
package io.casehub.ops.deployment.drift;

import io.casehub.desiredstate.api.NodeStatus;
import io.casehub.ops.api.deployment.*;
import io.casehub.ops.testing.StubDeploymentTarget;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.*;

class CaseHubAppDriftCheckerTest {

    private StubDeploymentTarget deployTarget;
    private CaseHubAppDriftChecker checker;

    @BeforeEach
    void setUp() {
        deployTarget = new StubDeploymentTarget();
        checker = new CaseHubAppDriftChecker(deployTarget);
    }

    @Test
    void nodeTypeIsCasehubApplication() {
        assertThat(checker.nodeType()).isEqualTo("casehub_application");
    }

    @Test
    void absentWhenNotDeployed() {
        var spec = new CaseHubApplicationSpec(
            "app-1", null, "img:1.0",
            new RuntimeConfig("0.2", List.of()),
            Map.of(), List.of(), 1);
        assertThat(checker.check(spec, "tenant-1")).isEqualTo(NodeStatus.ABSENT);
    }

    @Test
    void presentWhenDeployed() {
        var spec = new CaseHubApplicationSpec(
            "app-1", null, "img:1.0",
            new RuntimeConfig("0.2", List.of()),
            Map.of(), List.of(), 1);
        var config = new DeployConfig("app-1", 1, Map.of(), Map.of());
        deployTarget.deploy("img:1.0", config);
        assertThat(checker.check(spec, "tenant-1")).isEqualTo(NodeStatus.PRESENT);
    }

    @Test
    void unknownForNonApplicationSpec() {
        var spec = new io.casehub.ops.api.deployment.ChannelNodeSpec(
            "ch-1", "test-channel", "general", List.of());
        assertThat(checker.check(spec, "tenant-1")).isEqualTo(NodeStatus.UNKNOWN);
    }
}
```

- [ ] **Step 6: Run test to verify it fails**

Run: `mvn --batch-mode -o test -pl deployment -Dtest=CaseHubAppDriftCheckerTest`
Expected: compilation failure — `CaseHubAppDriftChecker` does not exist.

- [ ] **Step 7: Implement CaseHubAppDriftChecker**

Create `deployment/src/main/java/io/casehub/ops/deployment/drift/CaseHubAppDriftChecker.java`:

```java
package io.casehub.ops.deployment.drift;

import io.casehub.desiredstate.api.NodeSpec;
import io.casehub.desiredstate.api.NodeStatus;
import io.casehub.ops.api.deployment.ApplicationDeploymentTarget;
import io.casehub.ops.api.deployment.CaseHubApplicationSpec;
import io.casehub.ops.api.deployment.NodeDriftChecker;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

@ApplicationScoped
public class CaseHubAppDriftChecker implements NodeDriftChecker {

    private final ApplicationDeploymentTarget deployTarget;

    @Inject
    public CaseHubAppDriftChecker(ApplicationDeploymentTarget deployTarget) {
        this.deployTarget = deployTarget;
    }

    @Override
    public String nodeType() {
        return "casehub_application";
    }

    @Override
    public NodeStatus check(NodeSpec spec, String tenancyId) {
        if (!(spec instanceof CaseHubApplicationSpec appSpec)) {
            return NodeStatus.UNKNOWN;
        }
        var instanceId = appSpec.applicationId() + "-instance";
        return deployTarget.status(instanceId);
    }
}
```

- [ ] **Step 8: Run CaseHubAppDriftCheckerTest — verify pass**

Run: `mvn --batch-mode -o test -pl deployment -Dtest=CaseHubAppDriftCheckerTest`
Expected: all 4 tests PASS.

- [ ] **Step 9: Commit**

```bash
git add deployment/src/main/java/io/casehub/ops/deployment/handler/CaseHubAppProvisionHandler.java \
       deployment/src/main/java/io/casehub/ops/deployment/drift/CaseHubAppDriftChecker.java \
       deployment/src/test/java/io/casehub/ops/deployment/handler/CaseHubAppProvisionHandlerTest.java \
       deployment/src/test/java/io/casehub/ops/deployment/drift/CaseHubAppDriftCheckerTest.java
git commit -m "feat(#83): CaseHubAppProvisionHandler + CaseHubAppDriftChecker"
```

---

### Task 4: Wire into DeploymentGoalCompiler, NodeProvisioner, ActualStateAdapter, ApprovalEvaluator

**Files:**
- Modify: `deployment/src/main/java/io/casehub/ops/deployment/DeploymentGoalCompiler.java:36-48` (add applications)
- Modify: `deployment/src/main/java/io/casehub/ops/deployment/DeploymentNodeProvisioner.java` (add handler + switch cases + handledTypes)
- Modify: `deployment/src/main/java/io/casehub/ops/deployment/DeploymentActualStateAdapter.java:39-47` (add handledTypes)
- Modify: `deployment/src/main/java/io/casehub/ops/deployment/DeploymentApprovalEvaluator.java:34-57` (add switch cases)

**Interfaces:**
- Consumes: `CaseHubAppProvisionHandler` (Task 3), `CaseHubAppDriftChecker` (Task 3), `CaseHubApplicationSpec` (Task 1)
- Produces: Updated wiring — `casehub_application` handled in all four deployment SPI implementations

- [ ] **Step 1: Update DeploymentGoalCompiler — add applications**

Use `ide_replace_member` on `DeploymentGoalCompiler.compile` to add:

```java
compileEntries(goals.applications(), nodes, dependencies);
```

after the `compileEntries(goals.pools(), ...)` call. The full method becomes:

```java
@Override
public CompilationResult compile(DeploymentGoals goals, DesiredStateGraphFactory factory) {
    List<DesiredNode> nodes = new ArrayList<>();
    List<Dependency> dependencies = new ArrayList<>();

    compileEntries(goals.agents(), nodes, dependencies);
    compileEntries(goals.channels(), nodes, dependencies);
    compileEntries(resolveCaseTypes(goals.caseTypes()), nodes, dependencies);
    compileEntries(goals.trust(), nodes, dependencies);
    compileEntries(goals.endpoints(), nodes, dependencies);
    compileEntries(goals.detections(), nodes, dependencies);
    compileEntries(goals.pools(), nodes, dependencies);
    compileEntries(goals.applications(), nodes, dependencies);

    return CompilationResult.single(factory.of(nodes, dependencies));
}
```

- [ ] **Step 2: Update DeploymentNodeProvisioner — add handler + switch cases + handledTypes**

Use `ide_replace_member` to update `DeploymentNodeProvisioner`:

Add `CaseHubAppProvisionHandler appHandler` field and inject it. Add `NodeType.of("casehub_application")` to `handledTypes()`. Add `case CaseHubApplicationSpec s -> appHandler.provision(s, context);` and the deprovision equivalent to both switch expressions.

The `handledTypes()` becomes:

```java
@Override
public Set<NodeType> handledTypes() {
    return Set.of(
            NodeType.of("agent"),
            NodeType.of("channel"),
            NodeType.of("case_type"),
            NodeType.of("trust_policy"),
            NodeType.of("endpoint"),
            NodeType.of("detection"),
            NodeType.of("pool"),
            NodeType.of("casehub_application"));
}
```

Add to `doProvision` switch:
```java
case CaseHubApplicationSpec s -> appHandler.provision(s, context);
```

Add to `doDeprovision` switch:
```java
case CaseHubApplicationSpec s -> appHandler.deprovision(s, context);
```

Update the constructor to inject `CaseHubAppProvisionHandler`:

```java
@Inject
public DeploymentNodeProvisioner(
        AgentRegistry agentRegistry,
        DeploymentProviderConfigStore providerConfigStore,
        ChannelProvisionHandler channelHandler,
        CaseTypeProvisionHandler caseTypeHandler,
        TrustPolicyProvisionHandler trustHandler,
        EndpointProvisionHandler endpointHandler,
        DetectionProvisionHandler detectionHandler,
        PoolProvisionHandler poolHandler,
        CaseHubAppProvisionHandler appHandler,
        SpecHashStore specHashStore,
        ApprovalEvaluator approvalEvaluator,
        PlanStore planStore) {
    this.agentHandler = new AgentProvisionHandler(agentRegistry, providerConfigStore);
    // ... existing ...
    this.appHandler = appHandler;
    // ... existing ...
}
```

- [ ] **Step 3: Update DeploymentActualStateAdapter — add handledTypes**

Add `NodeType.of("casehub_application")` to `handledTypes()`:

```java
public Set<NodeType> handledTypes() {
    return Set.of(
            NodeType.of("agent"),
            NodeType.of("channel"),
            NodeType.of("case_type"),
            NodeType.of("trust_policy"),
            NodeType.of("endpoint"),
            NodeType.of("detection"),
            NodeType.of("pool"),
            NodeType.of("casehub_application"));
}
```

No other change needed — `CaseHubAppDriftChecker` is discovered via `Instance<NodeDriftChecker>` CDI injection and registers itself with nodeType `"casehub_application"`.

- [ ] **Step 4: Update DeploymentApprovalEvaluator — add switch cases**

Add `CaseHubApplicationSpec` to both switch expressions:

In `classifyRisk`:
```java
case CaseHubApplicationSpec s -> RiskClassification.HIGH;
```

In `generateSummary`:
```java
case CaseHubApplicationSpec s -> verb + " application '" + s.applicationId()
        + "' (image=" + (s.image() != null ? s.image() : "build:" + s.source()) + ")";
```

- [ ] **Step 5: Run existing deployment tests — verify all pass**

Run: `mvn --batch-mode -o test -pl deployment`

Fix any tests broken by the constructor signature change on `DeploymentNodeProvisioner`. Tests that construct `DeploymentNodeProvisioner` directly will need the new `CaseHubAppProvisionHandler` parameter. Use `ide_find_references` on the `DeploymentNodeProvisioner` constructor to find all call sites.

- [ ] **Step 6: Run full build**

Run: `mvn --batch-mode -o test -pl api,deployment,testing`
Expected: all modules compile, all tests PASS.

- [ ] **Step 7: Commit**

```bash
git add -u
git commit -m "feat(#83): wire casehub_application into GoalCompiler, NodeProvisioner, ActualStateAdapter, ApprovalEvaluator"
```

---

## Batch 3: Integration Test

### Task 5: Reconciliation Integration Test

**Files:**
- Create: `deployment/src/test/java/io/casehub/ops/deployment/CaseHubAppReconciliationIntegrationTest.java`
- Test: same file

**Interfaces:**
- Consumes: All types and wiring from Tasks 1–4
- Produces: End-to-end verification: YAML goals → compile → plan → provision → actual state check

- [ ] **Step 1: Write integration test**

Follow the pattern from `DeploymentReconciliationIntegrationTest` — a plain JUnit test that wires up the compiler, planner, provisioner, and adapter manually (no CDI container).

```java
package io.casehub.ops.deployment;

import io.casehub.desiredstate.api.*;
import io.casehub.desiredstate.runtime.DefaultDesiredStateGraphFactory;
import io.casehub.desiredstate.runtime.DefaultTransitionPlanner;
import io.casehub.ops.api.approval.InMemoryPlanStore;
import io.casehub.ops.api.deployment.*;
import io.casehub.ops.deployment.drift.CaseHubAppDriftChecker;
import io.casehub.ops.deployment.handler.CaseHubAppProvisionHandler;
import io.casehub.ops.testing.StubDeploymentTarget;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.*;

class CaseHubAppReconciliationIntegrationTest {

    private StubDeploymentTarget deployTarget;
    private DeploymentGoalCompiler compiler;
    private DeploymentActualStateAdapter adapter;
    private SpecHashStore specHashStore;

    @BeforeEach
    void setUp() {
        deployTarget = new StubDeploymentTarget();
        compiler = new DeploymentGoalCompiler();
        specHashStore = new SpecHashStore();
        var driftChecker = new CaseHubAppDriftChecker(deployTarget);
        adapter = new DeploymentActualStateAdapter(List.of(driftChecker), specHashStore);
    }

    @Test
    void compileAndPlanApplicationNode() {
        var goals = new DeploymentGoals(
            List.of(), List.of(), List.of(), List.of(),
            List.of(), List.of(), List.of(),
            List.of(new GoalEntry<>(
                new CaseHubApplicationSpec(
                    "incident-app", null, "ghcr.io/acme/incident:1.0",
                    new RuntimeConfig("0.2", List.of("a2a")),
                    Map.of(), List.of(), 1),
                List.of())),
            List.of());

        var factory = new DefaultDesiredStateGraphFactory();
        var result = compiler.compile(goals, factory);
        var graph = result.graphs().get(0);

        assertThat(graph.nodes()).hasSize(1);
        assertThat(graph.nodes().values().iterator().next().spec())
            .isInstanceOf(CaseHubApplicationSpec.class);

        var actualState = adapter.readActual(graph, "tenant-1");
        assertThat(actualState.statuses().get(NodeId.of("incident-app")))
            .isEqualTo(NodeStatus.ABSENT);

        var planner = new DefaultTransitionPlanner();
        var plan = planner.plan(graph, actualState);
        assertThat(plan.additions()).hasSize(1);
        assertThat(plan.additions().get(0).id()).isEqualTo(NodeId.of("incident-app"));
    }

    @Test
    void applicationWithDependency() {
        var appEntry = new GoalEntry<>(
            new CaseHubApplicationSpec(
                "loan-app", null, "ghcr.io/acme/loan:1.0",
                new RuntimeConfig("0.2", List.of("mcp")),
                Map.of(), List.of(), 1),
            List.of("incident-app"));
        var incidentEntry = new GoalEntry<>(
            new CaseHubApplicationSpec(
                "incident-app", null, "ghcr.io/acme/incident:1.0",
                new RuntimeConfig("0.2", List.of("a2a")),
                Map.of(), List.of(), 1),
            List.of());

        var goals = new DeploymentGoals(
            List.of(), List.of(), List.of(), List.of(),
            List.of(), List.of(), List.of(),
            List.of(incidentEntry, appEntry),
            List.of());

        var factory = new DefaultDesiredStateGraphFactory();
        var result = compiler.compile(goals, factory);
        var graph = result.graphs().get(0);

        assertThat(graph.nodes()).hasSize(2);
        assertThat(graph.dependencies()).hasSize(1);
        var dep = graph.dependencies().get(0);
        assertThat(dep.from()).isEqualTo(NodeId.of("loan-app"));
        assertThat(dep.to()).isEqualTo(NodeId.of("incident-app"));
    }

    @Test
    void provisionAndVerifyActualState() {
        var spec = new CaseHubApplicationSpec(
            "incident-app", null, "ghcr.io/acme/incident:1.0",
            new RuntimeConfig("0.2", List.of("a2a")),
            Map.of(), List.of(), 1);
        var goals = new DeploymentGoals(
            List.of(), List.of(), List.of(), List.of(),
            List.of(), List.of(), List.of(),
            List.of(new GoalEntry<>(spec, List.of())),
            List.of());

        var factory = new DefaultDesiredStateGraphFactory();
        var graph = compiler.compile(goals, factory).graphs().get(0);

        // Provision
        var handler = new CaseHubAppProvisionHandler(
            new PrebuiltImagePipeline(), deployTarget);
        var provResult = handler.provision(spec,
            new ProvisionContext("tenant-1", graph));
        assertThat(provResult).isInstanceOf(ProvisionResult.Success.class);

        // Verify actual state shows PRESENT
        specHashStore.record(NodeId.of("incident-app"), spec);
        var actualState = adapter.readActual(graph, "tenant-1");
        assertThat(actualState.statuses().get(NodeId.of("incident-app")))
            .isEqualTo(NodeStatus.PRESENT);

        // Plan should have no transitions
        var planner = new DefaultTransitionPlanner();
        var plan = planner.plan(graph, actualState);
        assertThat(plan.additions()).isEmpty();
        assertThat(plan.removals()).isEmpty();
    }
}
```

- [ ] **Step 2: Run integration test**

Run: `mvn --batch-mode -o test -pl deployment -Dtest=CaseHubAppReconciliationIntegrationTest`
Expected: all 3 tests PASS.

- [ ] **Step 3: Run full project build**

Run: `mvn --batch-mode -o test`
Expected: all modules compile, all tests PASS across the entire project.

- [ ] **Step 4: Commit**

```bash
git add deployment/src/test/java/io/casehub/ops/deployment/CaseHubAppReconciliationIntegrationTest.java
git commit -m "feat(#83): end-to-end reconciliation integration test for casehub_application nodes"
```

---

## References

- [2026-10-13-casehub-application-nodespec-provisioner-design.md] — design spec this plan implements
- `api/src/main/java/io/casehub/ops/api/deployment/DeploymentNodeSpec.java` — sealed hierarchy being extended
- `api/src/main/java/io/casehub/ops/api/deployment/AgentNodeSpec.java` — record pattern for new NodeSpec
- `api/src/main/java/io/casehub/ops/api/deployment/GoalEntry.java` — generic goal entry wrapper
- `api/src/main/java/io/casehub/ops/api/deployment/DeploymentGoals.java` — goals record being extended
- `deployment/src/main/java/io/casehub/ops/deployment/DeploymentGoalCompiler.java` — compiler pattern
- `deployment/src/main/java/io/casehub/ops/deployment/DeploymentNodeProvisioner.java` — provisioner dispatch
- `deployment/src/main/java/io/casehub/ops/deployment/DeploymentActualStateAdapter.java` — adapter pattern
- `deployment/src/main/java/io/casehub/ops/deployment/DeploymentApprovalEvaluator.java` — approval classification
- `deployment/src/main/java/io/casehub/ops/deployment/handler/AgentProvisionHandler.java` — handler pattern
- `deployment/src/main/java/io/casehub/ops/deployment/drift/AgentDriftChecker.java` — drift checker pattern
- `deployment/src/test/java/io/casehub/ops/deployment/handler/AgentProvisionHandlerTest.java` — test pattern
- `testing/src/main/java/io/casehub/ops/testing/` — shared test fixtures
- casehubio/engine#1017 — parent epic
- casehubio/engine#1018 — schema-driven record generation (future build pipeline)
- casehubio/engine#1019 — casehub.yaml descriptor format (future build pipeline)
- casehubio/casehub-ops#117 — deployment profiles (DeploymentTarget aligns)
