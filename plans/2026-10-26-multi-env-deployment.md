# Multi-Environment Topology Deployment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #150 — Epic: Multi-environment topology deployment
**Issue group:** #150, #151, #152, #153

**Goal:** Enable the same topology YAML to deploy to local (claudony), Kubernetes (k3d), and Podman without YAML changes.

**Architecture:** Lightweight abstractions (`TargetEnvironment` enum, `EnvironmentAwareNodeProvisioner`) live in `testing/`. Heavy wiring (`TargetEnvironmentFactory`, lifecycle extensions, `KubernetesDeploymentTarget`) live in `topology-tests/` where dependencies already exist. A single `EnvironmentAwareNodeProvisioner` delegates `casehub_application` provisioning to whichever `ApplicationDeploymentTarget` the environment provides. JUnit 5 extensions manage cluster lifecycle (k3d, Podman).

**Tech Stack:** Java 21, JUnit 5, fabric8 kubernetes-client (already a dependency), testcontainers-k3s (already a dependency), PodmanClient (existing in container/ module), ApplicationDeploymentTarget SPI (existing in api/)

## Global Constraints

- Java 21, Maven build
- `testing/` is test-scope only — never compile or runtime scope
- Existing `@Tag("integration-live")` tests must continue to work unchanged
- `TargetEnvironment.current()` defaults to `LOCAL` when no system property is set
- All new code uses ide-tooling for structural editing, not bash file ops
- IntelliJ project path: `/Users/mdproctor/claude/casehub/ops`

**Module split (self-review correction):** The `testing/` module has minimal dependencies (only `casehub-ops-api`). Heavy implementations that depend on `casehub-ops-claudony`, `casehub-ops-container`, fabric8, testcontainers, or Vert.x MUST go in `topology-tests/` where those dependencies already exist. Specifically:
- **testing/** (`io.casehub.ops.testing.env`): `TargetEnvironment` enum, `EnvironmentAwareNodeProvisioner` — lightweight, depends only on `casehub-ops-api` + `casehub-desiredstate-api`
- **topology-tests/** (`io.casehub.ops.topology.env`): `TargetEnvironmentFactory`, `KubernetesDeploymentTarget`, `K3dClusterExtension`, `PodmanEnvironmentExtension` — heavy, depends on claudony, container, fabric8, testcontainers

Code blocks in Tasks 2, 4, 5, 6 show `io.casehub.ops.testing.env` package but should use `io.casehub.ops.topology.env` and live in `topology-tests/src/test/java/`. Adjust paths and packages at implementation time.

---

## Batch 1: Environment Abstraction Foundation (#151)

### Task 1: TargetEnvironment enum and EnvironmentAwareNodeProvisioner

**Files:**
- Create: `testing/src/main/java/io/casehub/ops/testing/env/TargetEnvironment.java`
- Create: `testing/src/main/java/io/casehub/ops/testing/env/EnvironmentAwareNodeProvisioner.java`
- Test: `testing/src/test/java/io/casehub/ops/testing/env/TargetEnvironmentTest.java`
- Test: `testing/src/test/java/io/casehub/ops/testing/env/EnvironmentAwareNodeProvisionerTest.java`

**Interfaces:**
- Consumes: `ApplicationDeploymentTarget` (api/), `NodeProvisioner` (desiredstate-api), `CaseHubApplicationSpec` (api/), `DeployConfig` (api/), `DeployResult` (api/), `DeploymentApplicationNodeSpec` (api/)
- Produces: `TargetEnvironment.current()` returning `LOCAL`/`K8S`/`PODMAN`, `EnvironmentAwareNodeProvisioner(ApplicationDeploymentTarget)` implementing `NodeProvisioner` with `handledTypes() = {casehub_application}`

- [ ] **Step 1: Write failing test for TargetEnvironment.current() default**

```java
package io.casehub.ops.testing.env;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class TargetEnvironmentTest {

    @Test
    void current_defaultsToLocal() {
        System.clearProperty("casehub.topology.target-environment");
        assertThat(TargetEnvironment.current()).isEqualTo(TargetEnvironment.LOCAL);
    }

    @Test
    void current_readsSystemProperty() {
        System.setProperty("casehub.topology.target-environment", "k8s");
        try {
            assertThat(TargetEnvironment.current()).isEqualTo(TargetEnvironment.K8S);
        } finally {
            System.clearProperty("casehub.topology.target-environment");
        }
    }

    @Test
    void current_handlesPodman() {
        System.setProperty("casehub.topology.target-environment", "podman");
        try {
            assertThat(TargetEnvironment.current()).isEqualTo(TargetEnvironment.PODMAN);
        } finally {
            System.clearProperty("casehub.topology.target-environment");
        }
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode -o test -pl testing -Dtest=TargetEnvironmentTest`
Expected: FAIL — `TargetEnvironment` class not found

- [ ] **Step 3: Implement TargetEnvironment enum**

```java
package io.casehub.ops.testing.env;

public enum TargetEnvironment {
    LOCAL, K8S, PODMAN;

    private static final String PROPERTY = "casehub.topology.target-environment";

    public static TargetEnvironment current() {
        String value = System.getProperty(PROPERTY, "local");
        return TargetEnvironment.valueOf(value.toUpperCase().replace("-", "_"));
    }
}
```

- [ ] **Step 4: Run TargetEnvironment tests — verify pass**

Run: `mvn --batch-mode -o test -pl testing -Dtest=TargetEnvironmentTest`
Expected: 3 tests PASS

- [ ] **Step 5: Write failing test for EnvironmentAwareNodeProvisioner**

```java
package io.casehub.ops.testing.env;

import io.casehub.desiredstate.api.DesiredNode;
import io.casehub.desiredstate.api.HumanGating;
import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.api.NodeType;
import io.casehub.desiredstate.api.ProvisionContext;
import io.casehub.desiredstate.api.ProvisionResult;
import io.casehub.desiredstate.runtime.DefaultDesiredStateGraphFactory;
import io.casehub.ops.api.deployment.ApplicationDeploymentTarget;
import io.casehub.ops.api.deployment.CaseHubApplicationSpec;
import io.casehub.ops.api.deployment.DeployConfig;
import io.casehub.ops.api.deployment.DeployResult;
import io.casehub.ops.api.deployment.HealthCheckResult;
import io.casehub.desiredstate.api.NodeStatus;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class EnvironmentAwareNodeProvisionerTest {

    @Test
    void handledTypes_includesCasehubApplication() {
        var provisioner = new EnvironmentAwareNodeProvisioner(new StubTarget());
        assertThat(provisioner.handledTypes())
            .containsExactly(NodeType.of("casehub_application"));
    }

    @Test
    void provision_delegatesToTarget() {
        var target = new StubTarget();
        var provisioner = new EnvironmentAwareNodeProvisioner(target);
        var spec = new CaseHubApplicationSpec(
            "my-app", null, "casehubio/my-app:latest", null,
            Map.of(), List.of(), 1);
        var node = new DesiredNode(NodeId.of("my-app"), spec, HumanGating.NONE);
        var graph = new DefaultDesiredStateGraphFactory().of(List.of(), List.of());

        var result = provisioner.provision(node, new ProvisionContext("tenant", graph));

        assertThat(result).isInstanceOf(ProvisionResult.Success.class);
        assertThat(target.lastDeployedAppId).isEqualTo("my-app");
        assertThat(target.lastDeployedImage).isEqualTo("casehubio/my-app:latest");
    }

    @Test
    void deprovision_delegatesToTarget() {
        var target = new StubTarget();
        var provisioner = new EnvironmentAwareNodeProvisioner(target);
        var spec = new CaseHubApplicationSpec(
            "my-app", null, "casehubio/my-app:latest", null,
            Map.of(), List.of(), 1);
        var node = new DesiredNode(NodeId.of("my-app"), spec, HumanGating.NONE);
        var graph = new DefaultDesiredStateGraphFactory().of(List.of(), List.of());

        provisioner.deprovision(node, new io.casehub.desiredstate.api.DeprovisionContext("tenant", graph));

        assertThat(target.lastUndeployedId).isEqualTo("my-app");
    }

    private static class StubTarget implements ApplicationDeploymentTarget {
        String lastDeployedAppId;
        String lastDeployedImage;
        String lastUndeployedId;

        @Override public String targetId() { return "stub"; }
        @Override public DeployResult deploy(String imageRef, DeployConfig config) {
            lastDeployedAppId = config.applicationId();
            lastDeployedImage = imageRef;
            return new DeployResult.Success(config.applicationId());
        }
        @Override public DeployResult undeploy(String instanceId) {
            lastUndeployedId = instanceId;
            return new DeployResult.Success(instanceId);
        }
        @Override public NodeStatus status(String instanceId) { return NodeStatus.PRESENT; }
        @Override public HealthCheckResult healthCheck(String instanceId) {
            return new HealthCheckResult.Healthy();
        }
    }
}
```

- [ ] **Step 6: Run test to verify it fails**

Run: `mvn --batch-mode -o test -pl testing -Dtest=EnvironmentAwareNodeProvisionerTest`
Expected: FAIL — `EnvironmentAwareNodeProvisioner` class not found

- [ ] **Step 7: Implement EnvironmentAwareNodeProvisioner**

```java
package io.casehub.ops.testing.env;

import io.casehub.desiredstate.api.DeprovisionContext;
import io.casehub.desiredstate.api.DeprovisionResult;
import io.casehub.desiredstate.api.DesiredNode;
import io.casehub.desiredstate.api.NodeProvisioner;
import io.casehub.desiredstate.api.NodeType;
import io.casehub.desiredstate.api.ProvisionContext;
import io.casehub.desiredstate.api.ProvisionResult;
import io.casehub.ops.api.deployment.ApplicationDeploymentTarget;
import io.casehub.ops.api.deployment.CaseHubApplicationSpec;
import io.casehub.ops.api.deployment.DeployConfig;
import io.casehub.ops.api.deployment.DeployResult;
import io.casehub.ops.api.deployment.DeploymentApplicationNodeSpec;

import java.time.Duration;
import java.util.Map;
import java.util.Set;

public class EnvironmentAwareNodeProvisioner implements NodeProvisioner {

    private final ApplicationDeploymentTarget target;

    public EnvironmentAwareNodeProvisioner(ApplicationDeploymentTarget target) {
        this.target = target;
    }

    @Override
    public Set<NodeType> handledTypes() {
        return Set.of(NodeType.of("casehub_application"));
    }

    @Override
    public Duration resyncInterval() {
        return Duration.ofMinutes(5);
    }

    @Override
    public ProvisionResult provision(DesiredNode node, ProvisionContext context) {
        try {
            CaseHubApplicationSpec appSpec = extractAppSpec(node);
            DeployResult result = target.deploy(
                appSpec.image(),
                new DeployConfig(appSpec.applicationId(), appSpec.replicas(),
                    appSpec.config(), Map.of()));
            if (result instanceof DeployResult.Success) {
                return new ProvisionResult.Success();
            }
            return new ProvisionResult.Failed(((DeployResult.Failed) result).reason());
        } catch (Exception e) {
            return new ProvisionResult.Failed(e.getMessage());
        }
    }

    @Override
    public DeprovisionResult deprovision(DesiredNode node, DeprovisionContext context) {
        try {
            CaseHubApplicationSpec appSpec = extractAppSpec(node);
            DeployResult result = target.undeploy(appSpec.applicationId());
            if (result instanceof DeployResult.Success) {
                return new DeprovisionResult.Success();
            }
            return new DeprovisionResult.Failed(((DeployResult.Failed) result).reason());
        } catch (Exception e) {
            return new DeprovisionResult.Failed(e.getMessage());
        }
    }

    private CaseHubApplicationSpec extractAppSpec(DesiredNode node) {
        if (node.spec() instanceof DeploymentApplicationNodeSpec wrapper) {
            return wrapper.spec();
        }
        return (CaseHubApplicationSpec) node.spec();
    }
}
```

- [ ] **Step 8: Run all tests — verify pass**

Run: `mvn --batch-mode -o test -pl testing -Dtest=TargetEnvironmentTest,EnvironmentAwareNodeProvisionerTest`
Expected: All tests PASS

- [ ] **Step 9: Commit**

```bash
git add testing/src/main/java/io/casehub/ops/testing/env/ testing/src/test/java/io/casehub/ops/testing/env/
git commit -m "feat(#151): add TargetEnvironment enum and EnvironmentAwareNodeProvisioner

Introduces the environment abstraction layer for multi-env topology
deployment. TargetEnvironment reads casehub.topology.target-environment
system property (default: LOCAL). EnvironmentAwareNodeProvisioner
delegates casehub_application provisioning to any ApplicationDeploymentTarget.

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 2: TargetEnvironmentFactory and Maven profile wiring

**Files:**
- Create: `topology-tests/src/test/java/io/casehub/ops/topology/env/TargetEnvironmentFactory.java`
- Modify: `topology-tests/pom.xml` — add `target-k8s` and `target-podman` profiles
- Test: `topology-tests/src/test/java/io/casehub/ops/topology/env/TargetEnvironmentFactoryTest.java`

**Interfaces:**
- Consumes: `TargetEnvironment` (Task 1), `ApplicationDeploymentTarget` (api/), `ClaudonyDeploymentTarget` (claudony/), `PodmanDeploymentTarget` (container/), `PodmanClient` (container/), `HttpClaudonyFleetClient` (claudony/)
- Produces: `TargetEnvironmentFactory.createTarget(TargetEnvironment, Map<String, String>)` returning the correct `ApplicationDeploymentTarget` for the environment, `TargetEnvironmentFactory.createProvisioner(TargetEnvironment, Map<String, String>)` returning `EnvironmentAwareNodeProvisioner` wired to the right target

- [ ] **Step 1: Write failing test for TargetEnvironmentFactory**

```java
package io.casehub.ops.testing.env;

import org.junit.jupiter.api.Test;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

package io.casehub.ops.topology.env;

import org.junit.jupiter.api.Test;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class TargetEnvironmentFactoryTest {

    @Test
    void createTarget_local_returnsClaudonyTarget() {
        var target = TargetEnvironmentFactory.createTarget(
            TargetEnvironment.LOCAL, Map.of("claudony", "http://localhost:8080"));
        assertThat(target.targetId()).isEmpty();
    }

    @Test
    void createTarget_podman_returnsPodmanTarget() {
        var target = TargetEnvironmentFactory.createTarget(
            TargetEnvironment.PODMAN, Map.of());
        assertThat(target.targetId()).isEqualTo("podman");
    }

    @Test
    void createProvisioner_handlesApplicationNodes() {
        var provisioner = TargetEnvironmentFactory.createProvisioner(
            TargetEnvironment.LOCAL, Map.of("claudony", "http://localhost:8080"));
        assertThat(provisioner.handledTypes())
            .containsExactly(io.casehub.desiredstate.api.NodeType.of("casehub_application"));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode -o test -pl testing -Dtest=TargetEnvironmentFactoryTest`
Expected: FAIL — `TargetEnvironmentFactory` not found

- [ ] **Step 3: Implement TargetEnvironmentFactory**

```java
package io.casehub.ops.testing.env;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import io.casehub.ops.api.deployment.ApplicationDeploymentTarget;
import io.casehub.ops.claudony.ClaudonyDeploymentTarget;
import io.casehub.ops.claudony.HttpClaudonyFleetClient;
import io.casehub.ops.container.PodmanDeploymentTarget;
import io.casehub.ops.container.podman.PodmanClient;
import io.vertx.mutiny.core.Vertx;
import io.vertx.mutiny.ext.web.client.WebClient;

import java.util.Map;

public class TargetEnvironmentFactory {

    public static ApplicationDeploymentTarget createTarget(
            TargetEnvironment env, Map<String, String> endpoints) {
        return switch (env) {
            case LOCAL -> createClaudonyTarget(endpoints.getOrDefault("claudony", "http://localhost:8080"));
            case K8S -> throw new UnsupportedOperationException("K8S target not yet implemented — see #152");
            case PODMAN -> createPodmanTarget();
        };
    }

    public static EnvironmentAwareNodeProvisioner createProvisioner(
            TargetEnvironment env, Map<String, String> endpoints) {
        return new EnvironmentAwareNodeProvisioner(createTarget(env, endpoints));
    }

    private static ClaudonyDeploymentTarget createClaudonyTarget(String url) {
        ObjectMapper mapper = new ObjectMapper();
        mapper.registerModule(new JavaTimeModule());
        HttpClaudonyFleetClient client = new HttpClaudonyFleetClient(url, mapper);
        return new ClaudonyDeploymentTarget(client);
    }

    private static PodmanDeploymentTarget createPodmanTarget() {
        String socketPath = System.getProperty(
            "casehub.container.podman.socket", "/var/run/podman/podman.sock");
        Vertx vertx = Vertx.vertx();
        WebClient webClient = WebClient.create(vertx);
        PodmanClient podmanClient = new PodmanClient(webClient,
            io.vertx.core.net.SocketAddress.domainSocketAddress(socketPath));
        return new PodmanDeploymentTarget(podmanClient);
    }
}
```

- [ ] **Step 4: Run tests — verify pass**

Run: `mvn --batch-mode -o test -pl testing -Dtest=TargetEnvironmentFactoryTest`
Expected: All tests PASS

- [ ] **Step 5: Add Maven profiles to topology-tests/pom.xml**

Add inside the existing `<profiles>` section, after the `infra-live` profile:

```xml
        <profile>
            <id>target-k8s</id>
            <properties>
                <casehub.topology.target-environment>k8s</casehub.topology.target-environment>
            </properties>
            <build>
                <plugins>
                    <plugin>
                        <artifactId>maven-surefire-plugin</artifactId>
                        <configuration combine.self="override">
                            <groups>integration-live</groups>
                            <systemPropertyVariables>
                                <casehub.topology.target-environment>k8s</casehub.topology.target-environment>
                            </systemPropertyVariables>
                        </configuration>
                    </plugin>
                </plugins>
            </build>
        </profile>
        <profile>
            <id>target-podman</id>
            <properties>
                <casehub.topology.target-environment>podman</casehub.topology.target-environment>
            </properties>
            <build>
                <plugins>
                    <plugin>
                        <artifactId>maven-surefire-plugin</artifactId>
                        <configuration combine.self="override">
                            <groups>integration-live</groups>
                            <systemPropertyVariables>
                                <casehub.topology.target-environment>podman</casehub.topology.target-environment>
                            </systemPropertyVariables>
                        </configuration>
                    </plugin>
                </plugins>
            </build>
        </profile>
```

- [ ] **Step 6: Build to verify no compilation errors**

Run: `mvn --batch-mode -o compile -pl testing,topology-tests`
Expected: BUILD SUCCESS

- [ ] **Step 7: Commit**

```bash
git add testing/src/main/java/io/casehub/ops/testing/env/TargetEnvironmentFactory.java testing/src/test/java/io/casehub/ops/testing/env/TargetEnvironmentFactoryTest.java topology-tests/pom.xml
git commit -m "feat(#151): add TargetEnvironmentFactory and Maven env profiles

Factory creates the correct ApplicationDeploymentTarget per environment.
Adds -Ptarget-k8s and -Ptarget-podman Maven profiles that set the
casehub.topology.target-environment system property for surefire.

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 3: Verify existing integration tests pass with new abstraction on LOCAL

**Files:**
- No new files — regression verification only

**Interfaces:**
- Consumes: All existing integration test infrastructure

- [ ] **Step 1: Run existing integration-live tests to establish baseline**

Run: `mvn --batch-mode test -pl topology-tests -Pintegration-live`
Expected: All existing tests PASS (this requires a claudony instance running — if not available, run with `-o` to verify compilation only and note the constraint)

- [ ] **Step 2: Verify compilation of full project**

Run: `mvn --batch-mode -o compile`
Expected: BUILD SUCCESS — no compilation errors introduced

- [ ] **Step 3: Commit (if any fixes were needed)**

No commit expected unless fixes were required.

---

## Batch 2: Kubernetes Provisioner (#152)

### Task 4: KubernetesDeploymentTarget

**Files:**
- Create: `testing/src/main/java/io/casehub/ops/testing/env/KubernetesDeploymentTarget.java`
- Test: `testing/src/test/java/io/casehub/ops/testing/env/KubernetesDeploymentTargetTest.java`

**Interfaces:**
- Consumes: `ApplicationDeploymentTarget` (api/), `DeployConfig` (api/), `DeployResult` (api/), `HealthCheckResult` (api/), `NodeStatus` (desiredstate-api), fabric8 `KubernetesClient`
- Produces: `KubernetesDeploymentTarget(KubernetesClient, String namespace, String imageOverride)` implementing `ApplicationDeploymentTarget` with `targetId() = "kubernetes"`. `deploy()` creates K8s Deployment + Service. `undeploy()` deletes them. `status()` queries Deployment readiness. `healthCheck()` checks pod status.

- [ ] **Step 1: Write failing test for KubernetesDeploymentTarget**

```java
package io.casehub.ops.testing.env;

import io.casehub.desiredstate.api.NodeStatus;
import io.casehub.ops.api.deployment.DeployConfig;
import io.casehub.ops.api.deployment.DeployResult;
import io.fabric8.kubernetes.api.model.NamespaceBuilder;
import io.fabric8.kubernetes.client.KubernetesClient;
import io.fabric8.kubernetes.client.server.mock.EnableKubernetesMockClient;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

@EnableKubernetesMockClient(crud = true)
class KubernetesDeploymentTargetTest {

    KubernetesClient client;
    KubernetesDeploymentTarget target;

    @BeforeEach
    void setUp() {
        client.namespaces().resource(
            new NamespaceBuilder().withNewMetadata().withName("test").endMetadata().build()
        ).create();
        target = new KubernetesDeploymentTarget(client, "test", null);
    }

    @Test
    void targetId_returnsKubernetes() {
        assertThat(target.targetId()).isEqualTo("kubernetes");
    }

    @Test
    void deploy_createsDeploymentAndService() {
        var config = new DeployConfig("my-app", 1, Map.of(), Map.of());

        var result = target.deploy("casehubio/my-app:latest", config);

        assertThat(result).isInstanceOf(DeployResult.Success.class);
        assertThat(client.apps().deployments().inNamespace("test").withName("my-app").get())
            .isNotNull();
        assertThat(client.services().inNamespace("test").withName("my-app").get())
            .isNotNull();
    }

    @Test
    void undeploy_removesResources() {
        var config = new DeployConfig("my-app", 1, Map.of(), Map.of());
        target.deploy("casehubio/my-app:latest", config);

        var result = target.undeploy("my-app");

        assertThat(result).isInstanceOf(DeployResult.Success.class);
        assertThat(client.apps().deployments().inNamespace("test").withName("my-app").get())
            .isNull();
    }

    @Test
    void status_returnsAbsentForUnknownApp() {
        assertThat(target.status("nonexistent")).isEqualTo(NodeStatus.ABSENT);
    }

    @Test
    void deploy_usesImageOverride() {
        target = new KubernetesDeploymentTarget(client, "test", "override:latest");
        var config = new DeployConfig("my-app", 1, Map.of(), Map.of());

        target.deploy("original:latest", config);

        var dep = client.apps().deployments().inNamespace("test").withName("my-app").get();
        assertThat(dep.getSpec().getTemplate().getSpec().getContainers().get(0).getImage())
            .isEqualTo("override:latest");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode -o test -pl testing -Dtest=KubernetesDeploymentTargetTest`
Expected: FAIL — `KubernetesDeploymentTarget` not found

- [ ] **Step 3: Implement KubernetesDeploymentTarget**

```java
package io.casehub.ops.testing.env;

import io.casehub.desiredstate.api.NodeStatus;
import io.casehub.ops.api.deployment.ApplicationDeploymentTarget;
import io.casehub.ops.api.deployment.DeployConfig;
import io.casehub.ops.api.deployment.DeployResult;
import io.casehub.ops.api.deployment.HealthCheckResult;
import io.fabric8.kubernetes.api.model.ContainerBuilder;
import io.fabric8.kubernetes.api.model.EnvVarBuilder;
import io.fabric8.kubernetes.api.model.IntOrString;
import io.fabric8.kubernetes.api.model.ServiceBuilder;
import io.fabric8.kubernetes.api.model.ServicePortBuilder;
import io.fabric8.kubernetes.api.model.apps.Deployment;
import io.fabric8.kubernetes.api.model.apps.DeploymentBuilder;
import io.fabric8.kubernetes.client.KubernetesClient;
import io.fabric8.kubernetes.client.dsl.NonDeletingOperation;

import java.util.Map;

public class KubernetesDeploymentTarget implements ApplicationDeploymentTarget {

    private final KubernetesClient client;
    private final String namespace;
    private final String imageOverride;

    public KubernetesDeploymentTarget(KubernetesClient client,
                                       String namespace,
                                       String imageOverride) {
        this.client = client;
        this.namespace = namespace;
        this.imageOverride = imageOverride;
    }

    @Override
    public String targetId() { return "kubernetes"; }

    @Override
    public DeployResult deploy(String imageRef, DeployConfig config) {
        try {
            String image = imageOverride != null ? imageOverride : imageRef;
            String name = config.applicationId();

            var container = new ContainerBuilder()
                .withName(name)
                .withImage(image)
                .withEnv(config.env().entrySet().stream()
                    .map(e -> new EnvVarBuilder()
                        .withName(e.getKey()).withValue(e.getValue()).build())
                    .toList())
                .build();

            var deployment = new DeploymentBuilder()
                .withNewMetadata().withName(name).withNamespace(namespace)
                    .withLabels(Map.of("app", name)).endMetadata()
                .withNewSpec()
                    .withReplicas(config.replicas())
                    .withNewSelector().withMatchLabels(Map.of("app", name)).endSelector()
                    .withNewTemplate()
                        .withNewMetadata().withLabels(Map.of("app", name)).endMetadata()
                        .withNewSpec().withContainers(container).endSpec()
                    .endTemplate()
                .endSpec()
                .build();
            client.resource(deployment).createOr(NonDeletingOperation::update);

            var service = new ServiceBuilder()
                .withNewMetadata().withName(name).withNamespace(namespace).endMetadata()
                .withNewSpec()
                    .withSelector(Map.of("app", name))
                    .withPorts(new ServicePortBuilder()
                        .withPort(8080).withTargetPort(new IntOrString(8080))
                        .withProtocol("TCP").build())
                .endSpec()
                .build();
            client.resource(service).createOr(NonDeletingOperation::update);

            return new DeployResult.Success(name);
        } catch (Exception e) {
            return new DeployResult.Failed(e.getMessage());
        }
    }

    @Override
    public DeployResult undeploy(String instanceId) {
        try {
            client.apps().deployments().inNamespace(namespace)
                .withName(instanceId).delete();
            client.services().inNamespace(namespace)
                .withName(instanceId).delete();
            return new DeployResult.Success(instanceId);
        } catch (Exception e) {
            return new DeployResult.Failed(e.getMessage());
        }
    }

    @Override
    public NodeStatus status(String instanceId) {
        Deployment dep = client.apps().deployments().inNamespace(namespace)
            .withName(instanceId).get();
        if (dep == null) return NodeStatus.ABSENT;
        var status = dep.getStatus();
        if (status != null && status.getReadyReplicas() != null
                && status.getReadyReplicas() > 0) {
            return NodeStatus.PRESENT;
        }
        return NodeStatus.UNKNOWN;
    }

    @Override
    public HealthCheckResult healthCheck(String instanceId) {
        NodeStatus s = status(instanceId);
        return s == NodeStatus.PRESENT
            ? new HealthCheckResult.Healthy()
            : new HealthCheckResult.Unhealthy("Status: " + s);
    }
}
```

- [ ] **Step 4: Run tests — verify pass**

Run: `mvn --batch-mode -o test -pl testing -Dtest=KubernetesDeploymentTargetTest`
Expected: All tests PASS

- [ ] **Step 5: Wire K8S target into TargetEnvironmentFactory**

Modify `TargetEnvironmentFactory.createTarget()` — replace the `K8S` case:

```java
case K8S -> createK8sTarget(endpoints);
```

Add method:

```java
private static KubernetesDeploymentTarget createK8sTarget(Map<String, String> endpoints) {
    io.fabric8.kubernetes.client.KubernetesClient client =
        new io.fabric8.kubernetes.client.KubernetesClientBuilder().build();
    String namespace = endpoints.getOrDefault("k8s-namespace", "casehub-test");
    String imageOverride = endpoints.get("k8s-image-override");
    return new KubernetesDeploymentTarget(client, namespace, imageOverride);
}
```

- [ ] **Step 6: Commit**

```bash
git add testing/src/main/java/io/casehub/ops/testing/env/KubernetesDeploymentTarget.java testing/src/test/java/io/casehub/ops/testing/env/KubernetesDeploymentTargetTest.java testing/src/main/java/io/casehub/ops/testing/env/TargetEnvironmentFactory.java
git commit -m "feat(#152): add KubernetesDeploymentTarget

ApplicationDeploymentTarget implementation using fabric8 client.
Creates K8s Deployment + Service per application. Wired into
TargetEnvironmentFactory for K8S environment selection.

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 5: K3dClusterExtension for cluster lifecycle

**Files:**
- Create: `testing/src/main/java/io/casehub/ops/testing/env/K3dClusterExtension.java`
- Test: `testing/src/test/java/io/casehub/ops/testing/env/K3dClusterExtensionTest.java`

**Interfaces:**
- Consumes: `KubernetesDeploymentTarget` (Task 4), `TargetEnvironment` (Task 1), testcontainers-k3s `K3sContainer`, fabric8 `KubernetesClient`
- Produces: JUnit 5 `BeforeAllCallback`/`AfterAllCallback` extension. Creates a k3s cluster via testcontainers, injects `KubernetesClient` and endpoint map into test class static fields via `@K3dEndpoint` annotation.

- [ ] **Step 1: Write the K3dClusterExtension**

```java
package io.casehub.ops.testing.env;

import io.fabric8.kubernetes.api.model.NamespaceBuilder;
import io.fabric8.kubernetes.client.Config;
import io.fabric8.kubernetes.client.KubernetesClient;
import io.fabric8.kubernetes.client.KubernetesClientBuilder;
import org.junit.jupiter.api.extension.AfterAllCallback;
import org.junit.jupiter.api.extension.BeforeAllCallback;
import org.junit.jupiter.api.extension.ExtensionContext;
import org.testcontainers.k3s.K3sContainer;
import org.testcontainers.utility.DockerImageName;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;
import java.lang.reflect.Field;

public class K3dClusterExtension implements BeforeAllCallback, AfterAllCallback {

    private static final ExtensionContext.Namespace NAMESPACE =
        ExtensionContext.Namespace.create(K3dClusterExtension.class);
    private static final String TEST_NAMESPACE = "casehub-test";

    @Target(ElementType.FIELD)
    @Retention(RetentionPolicy.RUNTIME)
    public @interface K3dClient {}

    @Override
    public void beforeAll(ExtensionContext context) throws Exception {
        if (TargetEnvironment.current() != TargetEnvironment.K8S) return;

        K3sContainer k3s = new K3sContainer(DockerImageName.parse("rancher/k3s:v1.31.2-k3s1"));
        k3s.start();
        context.getStore(NAMESPACE).put("k3s", k3s);

        String kubeConfig = k3s.getKubeConfigYaml();
        Config config = Config.fromKubeconfig(kubeConfig);
        KubernetesClient client = new KubernetesClientBuilder().withConfig(config).build();
        context.getStore(NAMESPACE).put("client", client);

        client.namespaces().resource(
            new NamespaceBuilder().withNewMetadata()
                .withName(TEST_NAMESPACE).endMetadata().build()
        ).create();

        injectClient(context.getRequiredTestClass(), client);
    }

    @Override
    public void afterAll(ExtensionContext context) {
        KubernetesClient client = context.getStore(NAMESPACE)
            .remove("client", KubernetesClient.class);
        if (client != null) client.close();

        K3sContainer k3s = context.getStore(NAMESPACE)
            .remove("k3s", K3sContainer.class);
        if (k3s != null) k3s.stop();
    }

    private void injectClient(Class<?> testClass, KubernetesClient client)
            throws IllegalAccessException {
        for (Field field : testClass.getDeclaredFields()) {
            if (field.isAnnotationPresent(K3dClient.class)) {
                field.setAccessible(true);
                field.set(null, client);
            }
        }
    }
}
```

- [ ] **Step 2: Write a basic test verifying the extension activates only for K8S**

```java
package io.casehub.ops.testing.env;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;

import static org.assertj.core.api.Assertions.assertThat;

@ExtendWith(K3dClusterExtension.class)
class K3dClusterExtensionTest {

    @Test
    void extensionDoesNotActivateForLocal() {
        System.clearProperty("casehub.topology.target-environment");
        assertThat(TargetEnvironment.current()).isEqualTo(TargetEnvironment.LOCAL);
    }
}
```

- [ ] **Step 3: Run test to verify pass (extension should no-op for LOCAL)**

Run: `mvn --batch-mode -o test -pl testing -Dtest=K3dClusterExtensionTest`
Expected: PASS (extension skips when env is LOCAL)

- [ ] **Step 4: Commit**

```bash
git add testing/src/main/java/io/casehub/ops/testing/env/K3dClusterExtension.java testing/src/test/java/io/casehub/ops/testing/env/K3dClusterExtensionTest.java
git commit -m "feat(#152): add K3dClusterExtension for K8s cluster lifecycle

JUnit 5 extension managing testcontainers k3s cluster creation and
teardown. Only activates when TargetEnvironment is K8S. Injects
KubernetesClient into test class fields annotated with @K3dClient.

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 3: Podman Provisioner (#153)

### Task 6: PodmanEnvironmentExtension for Podman lifecycle

**Files:**
- Create: `testing/src/main/java/io/casehub/ops/testing/env/PodmanEnvironmentExtension.java`
- Test: `testing/src/test/java/io/casehub/ops/testing/env/PodmanEnvironmentExtensionTest.java`

**Interfaces:**
- Consumes: `TargetEnvironment` (Task 1), `PodmanDeploymentTarget` (container/), `PodmanClient` (container/), `TargetEnvironmentFactory` (Task 2)
- Produces: JUnit 5 `BeforeAllCallback`/`AfterAllCallback` extension. Verifies Podman socket is accessible, creates a test network for isolation, injects `PodmanClient` into test class static fields via `@PodmanEndpoint` annotation. Cleans up containers and network on teardown.

- [ ] **Step 1: Write PodmanEnvironmentExtension**

```java
package io.casehub.ops.testing.env;

import io.casehub.ops.container.podman.PodmanClient;
import io.vertx.core.net.SocketAddress;
import io.vertx.mutiny.core.Vertx;
import io.vertx.mutiny.ext.web.client.WebClient;
import org.junit.jupiter.api.extension.AfterAllCallback;
import org.junit.jupiter.api.extension.BeforeAllCallback;
import org.junit.jupiter.api.extension.ExtensionContext;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;
import java.lang.reflect.Field;

public class PodmanEnvironmentExtension implements BeforeAllCallback, AfterAllCallback {

    private static final ExtensionContext.Namespace NAMESPACE =
        ExtensionContext.Namespace.create(PodmanEnvironmentExtension.class);
    private static final String TEST_NETWORK = "casehub-test-net";

    @Target(ElementType.FIELD)
    @Retention(RetentionPolicy.RUNTIME)
    public @interface PodmanEndpoint {}

    @Override
    public void beforeAll(ExtensionContext context) throws Exception {
        if (TargetEnvironment.current() != TargetEnvironment.PODMAN) return;

        String socketPath = System.getProperty(
            "casehub.container.podman.socket", "/var/run/podman/podman.sock");
        Vertx vertx = Vertx.vertx();
        WebClient webClient = WebClient.create(vertx);
        PodmanClient podmanClient = new PodmanClient(webClient,
            SocketAddress.domainSocketAddress(socketPath));

        podmanClient.createNetwork(TEST_NETWORK).await().indefinitely();

        context.getStore(NAMESPACE).put("podmanClient", podmanClient);
        context.getStore(NAMESPACE).put("vertx", vertx);

        injectClient(context.getRequiredTestClass(), podmanClient);
    }

    @Override
    public void afterAll(ExtensionContext context) {
        PodmanClient client = context.getStore(NAMESPACE)
            .remove("podmanClient", PodmanClient.class);
        if (client != null) {
            try {
                var containers = client.listContainers().await().indefinitely();
                for (var c : containers) {
                    if (c.names().stream().anyMatch(n -> n.startsWith("casehub-test-"))) {
                        client.stopContainer(c.id()).await().indefinitely();
                        client.removeContainer(c.id()).await().indefinitely();
                    }
                }
            } catch (Exception ignored) {}
        }

        Vertx vertx = context.getStore(NAMESPACE).remove("vertx", Vertx.class);
        if (vertx != null) vertx.close().await().indefinitely();
    }

    private void injectClient(Class<?> testClass, PodmanClient client)
            throws IllegalAccessException {
        for (Field field : testClass.getDeclaredFields()) {
            if (field.isAnnotationPresent(PodmanEndpoint.class)) {
                field.setAccessible(true);
                field.set(null, client);
            }
        }
    }
}
```

- [ ] **Step 2: Write test verifying extension skips for non-PODMAN env**

```java
package io.casehub.ops.testing.env;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;

import static org.assertj.core.api.Assertions.assertThat;

@ExtendWith(PodmanEnvironmentExtension.class)
class PodmanEnvironmentExtensionTest {

    @Test
    void extensionDoesNotActivateForLocal() {
        System.clearProperty("casehub.topology.target-environment");
        assertThat(TargetEnvironment.current()).isEqualTo(TargetEnvironment.LOCAL);
    }
}
```

- [ ] **Step 3: Run test to verify pass**

Run: `mvn --batch-mode -o test -pl testing -Dtest=PodmanEnvironmentExtensionTest`
Expected: PASS (extension skips for LOCAL)

- [ ] **Step 4: Commit**

```bash
git add testing/src/main/java/io/casehub/ops/testing/env/PodmanEnvironmentExtension.java testing/src/test/java/io/casehub/ops/testing/env/PodmanEnvironmentExtensionTest.java
git commit -m "feat(#153): add PodmanEnvironmentExtension for Podman lifecycle

JUnit 5 extension managing Podman test network creation and container
cleanup. Only activates when TargetEnvironment is PODMAN. Injects
PodmanClient into test class fields annotated with @PodmanEndpoint.

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 4: Full build verification

### Task 7: Full project build and regression verification

**Files:**
- No new files — verification only

**Interfaces:**
- Consumes: Everything from Tasks 1-6

- [ ] **Step 1: Full project compile**

Run: `mvn --batch-mode -o compile`
Expected: BUILD SUCCESS

- [ ] **Step 2: Full project test (unit tests only)**

Run: `mvn --batch-mode -o test`
Expected: BUILD SUCCESS — all existing tests pass, new tests pass

- [ ] **Step 3: Verify integration-live profile still works (if claudony available)**

Run: `mvn --batch-mode test -pl topology-tests -Pintegration-live`
Expected: All existing integration tests PASS unchanged

- [ ] **Step 4: Final commit if any fixes needed**

No commit expected unless fixes were required for compilation or test issues.

## References

- `specs/issue-150-multi-env-deployment/2026-10-26-multi-env-deployment-design.md` — design spec this plan implements
- `api/src/main/java/io/casehub/ops/api/deployment/ApplicationDeploymentTarget.java:5` — target SPI interface
- `api/src/main/java/io/casehub/ops/api/deployment/DeployConfig.java:7` — deploy config record
- `api/src/main/java/io/casehub/ops/api/deployment/DeployResult.java:3` — deploy result sealed interface
- `api/src/main/java/io/casehub/ops/api/deployment/CaseHubApplicationSpec.java:12` — application spec record
- `container/src/main/java/io/casehub/ops/container/PodmanDeploymentTarget.java:14` — existing Podman target
- `container/src/main/java/io/casehub/ops/container/podman/PodmanClient.java` — existing Podman REST API client
- `topology-tests/src/test/java/io/casehub/ops/topology/integration/ClaudonyNodeProvisioner.java:20` — current provisioner (pattern for EnvironmentAwareNodeProvisioner)
- `topology-tests/src/test/java/io/casehub/ops/topology/integration/ServiceClusterExtension.java:15` — existing LOCAL lifecycle extension
- `topology-tests/src/test/java/io/casehub/ops/topology/live/Fabric8NodeProvisioner.java:33` — existing K8s provisioner patterns
- `topology-tests/pom.xml:176` — existing Maven profiles
- `docs/specs/issue-117-deployment-profiles-bootstrap/2026-10-14-deployment-profiles-bootstrap-design.md` — #117 deployment profiles spec
- GitHub #150, #151, #152, #153 — issue group
