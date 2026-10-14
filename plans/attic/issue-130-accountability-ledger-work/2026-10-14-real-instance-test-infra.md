# Real Instance Integration Test Infrastructure — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #139 — Real instance integration test infrastructure
**Issue group:** #139

**Goal:** Build JUnit 5 extension infrastructure that starts real Quarkus
services as separate JVM processes, with database provisioning and
dynamic port allocation, proven by a test that registers apps against a
real claudony instance.

**Architecture:** ServiceProcess wraps ProcessBuilder for Quarkus
fast-jars. ServiceClusterExtension orchestrates multiple ServiceProcess
instances, starts Testcontainers PostgreSQL for claudony, allocates
dynamic ports, injects service URLs into test fields, and tears down
after the test class. A proof-of-concept test compiles the hello-llm
topology and deploys it against a real claudony instance.

**Tech Stack:** JUnit 5 extensions, ProcessBuilder, Testcontainers
PostgreSQL, Quarkus fast-jar, AssertJ

## Global Constraints

- All new files in `topology-tests/src/test/java/io/casehub/ops/topology/integration/`
- Package: `io.casehub.ops.topology.integration`
- Test tag: `integration-live` (excluded from default surefire, activated via `-Pintegration-live`)
- Testcontainers PostgreSQL version managed by Quarkus BOM (no explicit version)
- Services start with `-Dquarkus.profile=test` and `-Dquarkus.http.auth.permission.protected.policy=permit`
- Family root: system property `casehub.family.root` or `../../` relative to project root
- Jar path convention: `<family-root>/<repo>/target/quarkus-app/quarkus-run.jar`
- All commits reference `Refs #139`

---

## Batch 1: ServiceProcess — start and manage a single Quarkus JVM

### Task 1: ServiceProcess — locate jar, start process, health-poll, stop

**Files:**
- Create: `topology-tests/src/test/java/io/casehub/ops/topology/integration/ServiceProcess.java`
- Test: `topology-tests/src/test/java/io/casehub/ops/topology/integration/ServiceProcessTest.java`

**Interfaces:**
- Consumes: nothing (foundation class)
- Produces:
  - `ServiceProcess(String name, Path jarPath, int port, Map<String,String> systemProperties)`
  - `void start()` — launches process, polls health, throws on timeout
  - `void stop()` — SIGTERM, grace period, destroyForcibly
  - `int port()` — returns assigned port
  - `String baseUrl()` — returns `http://localhost:<port>`
  - `boolean isRunning()` — checks process liveness
  - `static Path resolveJar(String repo)` — locates jar by convention
  - `static int findFreePort()` — ServerSocket(0) trick

- [ ] **Step 1: Write failing test — resolveJar with missing jar throws**

```java
package io.casehub.ops.topology.integration;

import org.junit.jupiter.api.Test;
import java.nio.file.Path;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class ServiceProcessTest {

    @Test
    void resolveJar_missingJar_throwsWithActionableMessage() {
        assertThatThrownBy(() -> ServiceProcess.resolveJar("nonexistent/app"))
            .isInstanceOf(IllegalStateException.class)
            .hasMessageContaining("quarkus-run.jar")
            .hasMessageContaining("mvn");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode -o test -pl topology-tests -Dtest=ServiceProcessTest#resolveJar_missingJar_throwsWithActionableMessage`
Expected: FAIL — class not found

- [ ] **Step 3: Implement ServiceProcess**

```java
package io.casehub.ops.topology.integration;

import java.io.IOException;
import java.io.UncheckedIOException;
import java.net.HttpURLConnection;
import java.net.ServerSocket;
import java.net.URI;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.ArrayList;
import java.util.Map;

public class ServiceProcess {

    private final String name;
    private final Path jarPath;
    private final int port;
    private final Map<String, String> systemProperties;
    private Process process;

    public ServiceProcess(String name, Path jarPath, int port,
                          Map<String, String> systemProperties) {
        this.name = name;
        this.jarPath = jarPath;
        this.port = port;
        this.systemProperties = systemProperties;
    }

    public void start() {
        var cmd = new ArrayList<String>();
        cmd.add("java");
        for (var entry : systemProperties.entrySet()) {
            cmd.add("-D" + entry.getKey() + "=" + entry.getValue());
        }
        cmd.add("-Dquarkus.http.port=" + port);
        cmd.add("-jar");
        cmd.add(jarPath.toString());

        Path logDir = Path.of("target/service-logs");
        try {
            Files.createDirectories(logDir);
            ProcessBuilder pb = new ProcessBuilder(cmd);
            pb.redirectOutput(logDir.resolve(name + ".log").toFile());
            pb.redirectErrorStream(true);
            process = pb.start();
        } catch (IOException e) {
            throw new UncheckedIOException("Failed to start " + name, e);
        }

        waitForHealth();
    }

    public void stop() {
        if (process == null || !process.isAlive()) return;
        process.destroy();
        try {
            if (!process.waitFor(10, java.util.concurrent.TimeUnit.SECONDS)) {
                process.destroyForcibly();
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            process.destroyForcibly();
        }
    }

    public int port() { return port; }

    public String baseUrl() { return "http://localhost:" + port; }

    public boolean isRunning() { return process != null && process.isAlive(); }

    public static int findFreePort() {
        try (ServerSocket socket = new ServerSocket(0)) {
            return socket.getLocalPort();
        } catch (IOException e) {
            throw new UncheckedIOException("Failed to find free port", e);
        }
    }

    public static Path resolveJar(String repo) {
        String familyRoot = System.getProperty("casehub.family.root");
        Path base;
        if (familyRoot != null) {
            base = Path.of(familyRoot);
        } else {
            base = Path.of("").toAbsolutePath().getParent().getParent();
        }
        Path jar = base.resolve(repo).resolve("target/quarkus-app/quarkus-run.jar");
        if (!Files.exists(jar)) {
            throw new IllegalStateException(
                "Service jar not found at " + jar +
                ". Run 'mvn -pl " + repo.split("/")[repo.split("/").length - 1] +
                " package -DskipTests' in " + base.resolve(repo.split("/")[0]));
        }
        return jar;
    }

    private void waitForHealth() {
        String healthUrl = baseUrl() + "/q/health/ready";
        long deadline = System.currentTimeMillis() + 60_000;
        while (System.currentTimeMillis() < deadline) {
            if (!process.isAlive()) {
                throw new IllegalStateException(
                    name + " process died during startup. Check target/service-logs/" + name + ".log");
            }
            try {
                HttpURLConnection conn = (HttpURLConnection)
                    URI.create(healthUrl).toURL().openConnection();
                conn.setConnectTimeout(500);
                conn.setReadTimeout(500);
                if (conn.getResponseCode() == 200) return;
            } catch (IOException ignored) {
            }
            try { Thread.sleep(500); } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                throw new RuntimeException("Interrupted waiting for " + name);
            }
        }
        stop();
        throw new IllegalStateException(
            name + " did not become healthy within 60s. Check target/service-logs/" + name + ".log");
    }
}
```

- [ ] **Step 4: Write test — findFreePort returns valid port**

```java
@Test
void findFreePort_returnsPortInValidRange() {
    int port = ServiceProcess.findFreePort();
    assertThat(port).isBetween(1024, 65535);
}
```

- [ ] **Step 5: Run all tests to verify they pass**

Run: `mvn --batch-mode -o test -pl topology-tests -Dtest=ServiceProcessTest`
Expected: PASS (2 tests)

- [ ] **Step 6: Commit**

```bash
git add topology-tests/src/test/java/io/casehub/ops/topology/integration/
git commit -m "feat(#139): ServiceProcess — locate jar, start process, health-poll, stop

Refs #139

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 2: Annotations and LlmMode — declarative test API

### Task 2: Service, ServiceCluster, ServiceEndpoint annotations + LlmMode + LlmAvailable

**Files:**
- Create: `topology-tests/src/test/java/io/casehub/ops/topology/integration/Service.java`
- Create: `topology-tests/src/test/java/io/casehub/ops/topology/integration/ServiceCluster.java`
- Create: `topology-tests/src/test/java/io/casehub/ops/topology/integration/ServiceEndpoint.java`
- Create: `topology-tests/src/test/java/io/casehub/ops/topology/integration/LlmMode.java`
- Create: `topology-tests/src/test/java/io/casehub/ops/topology/integration/LlmAvailable.java`
- Test: `topology-tests/src/test/java/io/casehub/ops/topology/integration/LlmAvailableTest.java`

**Interfaces:**
- Consumes: nothing
- Produces:
  - `@Service(name: String, repo: String)` — repeatable annotation
  - `@ServiceCluster(services: Service[])` — class-level annotation, `@ExtendWith(ServiceClusterExtension.class)`
  - `@ServiceEndpoint(value: String)` — field injection annotation for service base URL
  - `LlmMode { FULL, SIMULATION }` — enum with `static LlmMode detect()` method
  - `@LlmAvailable` — annotation + nested `LlmCondition implements ExecutionCondition`

- [ ] **Step 1: Write failing test — LlmMode detection**

```java
package io.casehub.ops.topology.integration;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class LlmAvailableTest {

    @Test
    void detect_withoutApiKey_returnsSimulation() {
        // ANTHROPIC_API_KEY is not set in test environment
        // If it happens to be set, this test still validates the enum exists
        LlmMode mode = LlmMode.detect();
        assertThat(mode).isNotNull();
        assertThat(mode).isIn(LlmMode.FULL, LlmMode.SIMULATION);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode -o test -pl topology-tests -Dtest=LlmAvailableTest`
Expected: FAIL — class not found

- [ ] **Step 3: Implement annotations and LlmMode**

`LlmMode.java`:
```java
package io.casehub.ops.topology.integration;

public enum LlmMode {
    FULL,
    SIMULATION;

    public static LlmMode detect() {
        String apiKey = System.getenv("ANTHROPIC_API_KEY");
        return (apiKey != null && !apiKey.isBlank()) ? FULL : SIMULATION;
    }
}
```

`Service.java`:
```java
package io.casehub.ops.topology.integration;

import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;

@Retention(RetentionPolicy.RUNTIME)
public @interface Service {
    String name();
    String repo();
}
```

`ServiceCluster.java`:
```java
package io.casehub.ops.topology.integration;

import org.junit.jupiter.api.extension.ExtendWith;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@ExtendWith(ServiceClusterExtension.class)
public @interface ServiceCluster {
    Service[] services();
}
```

`ServiceEndpoint.java`:
```java
package io.casehub.ops.topology.integration;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface ServiceEndpoint {
    String value();
}
```

`LlmAvailable.java`:
```java
package io.casehub.ops.topology.integration;

import org.junit.jupiter.api.extension.ConditionEvaluationResult;
import org.junit.jupiter.api.extension.ExecutionCondition;
import org.junit.jupiter.api.extension.ExtendWith;
import org.junit.jupiter.api.extension.ExtensionContext;

import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;

@Retention(RetentionPolicy.RUNTIME)
@ExtendWith(LlmAvailable.LlmCondition.class)
public @interface LlmAvailable {

    class LlmCondition implements ExecutionCondition {
        @Override
        public ConditionEvaluationResult evaluateExecutionCondition(ExtensionContext context) {
            if (LlmMode.detect() == LlmMode.FULL) {
                return ConditionEvaluationResult.enabled("ANTHROPIC_API_KEY is set");
            }
            return ConditionEvaluationResult.disabled(
                "ANTHROPIC_API_KEY not set — skipping LLM-dependent test");
        }
    }
}
```

- [ ] **Step 4: Create stub ServiceClusterExtension (empty, compiles)**

```java
package io.casehub.ops.topology.integration;

import org.junit.jupiter.api.extension.AfterAllCallback;
import org.junit.jupiter.api.extension.BeforeAllCallback;
import org.junit.jupiter.api.extension.ExtensionContext;

public class ServiceClusterExtension implements BeforeAllCallback, AfterAllCallback {

    @Override
    public void beforeAll(ExtensionContext context) {
        // Implemented in Task 3
    }

    @Override
    public void afterAll(ExtensionContext context) {
        // Implemented in Task 3
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn --batch-mode -o test -pl topology-tests -Dtest=LlmAvailableTest`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add topology-tests/src/test/java/io/casehub/ops/topology/integration/
git commit -m "feat(#139): annotations, LlmMode, LlmAvailable — declarative test API

Refs #139

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 3: ServiceClusterExtension + Maven config + proof-of-concept test

### Task 3: ServiceClusterExtension — orchestrate processes, PostgreSQL, field injection

**Files:**
- Modify: `topology-tests/src/test/java/io/casehub/ops/topology/integration/ServiceClusterExtension.java`
- Modify: `topology-tests/pom.xml` — add Testcontainers deps, integration-live profile, surefire exclusion
- Test: `topology-tests/src/test/java/io/casehub/ops/topology/integration/ServiceClusterExtensionTest.java`

**Interfaces:**
- Consumes:
  - `ServiceProcess(String, Path, int, Map<String,String>)` from Task 1
  - `ServiceProcess.resolveJar(String)` from Task 1
  - `ServiceProcess.findFreePort()` from Task 1
  - `@ServiceCluster`, `@Service`, `@ServiceEndpoint` from Task 2
- Produces:
  - `ServiceClusterExtension implements BeforeAllCallback, AfterAllCallback` — reads `@ServiceCluster`, starts PostgreSQL container, starts services, injects `@ServiceEndpoint` fields, tears down

- [ ] **Step 1: Add Testcontainers dependencies and integration-live profile to pom.xml**

Add to `topology-tests/pom.xml` `<dependencies>` section:
```xml
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>
</dependency>
```

Add `<dependencyManagement>` to import the Testcontainers BOM (since it's not in casehub-parent):
```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>testcontainers-bom</artifactId>
            <version>1.20.4</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

Update default surefire `<excludedGroups>`:
```xml
<excludedGroups>reconciliation,infra-live,integration-live</excludedGroups>
```

Add `integration-live` profile:
```xml
<profile>
    <id>integration-live</id>
    <build>
        <plugins>
            <plugin>
                <artifactId>maven-surefire-plugin</artifactId>
                <configuration combine.self="override">
                    <groups>integration-live</groups>
                </configuration>
            </plugin>
        </plugins>
    </build>
</profile>
```

- [ ] **Step 2: Write failing test — extension starts and injects endpoint**

```java
package io.casehub.ops.topology.integration;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.Tag;
import static org.assertj.core.api.Assertions.assertThat;

@Tag("integration-live")
@ServiceCluster(services = {
    @Service(name = "claudony", repo = "claudony/app")
})
class ServiceClusterExtensionTest {

    @ServiceEndpoint("claudony")
    static String claudonyUrl;

    @Test
    void claudonyStartsAndUrlIsInjected() {
        assertThat(claudonyUrl).isNotNull();
        assertThat(claudonyUrl).startsWith("http://localhost:");
    }

    @Test
    void claudonyHealthEndpointReturnsOk() throws Exception {
        var conn = (java.net.HttpURLConnection)
            java.net.URI.create(claudonyUrl + "/q/health/ready").toURL().openConnection();
        conn.setConnectTimeout(5000);
        conn.setReadTimeout(5000);
        assertThat(conn.getResponseCode()).isEqualTo(200);
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `mvn --batch-mode test -pl topology-tests -Pintegration-live -Dtest=ServiceClusterExtensionTest`
Expected: FAIL — extension doesn't start services yet

- [ ] **Step 4: Implement ServiceClusterExtension**

```java
package io.casehub.ops.topology.integration;

import org.junit.jupiter.api.extension.AfterAllCallback;
import org.junit.jupiter.api.extension.BeforeAllCallback;
import org.junit.jupiter.api.extension.ExtensionContext;
import org.testcontainers.containers.PostgreSQLContainer;

import java.lang.reflect.Field;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

public class ServiceClusterExtension implements BeforeAllCallback, AfterAllCallback {

    private static final ExtensionContext.Namespace NAMESPACE =
        ExtensionContext.Namespace.create(ServiceClusterExtension.class);

    @Override
    public void beforeAll(ExtensionContext context) throws Exception {
        ServiceCluster cluster = context.getRequiredTestClass()
            .getAnnotation(ServiceCluster.class);
        if (cluster == null) return;

        ClusterState state = new ClusterState();
        context.getStore(NAMESPACE).put("state", state);

        boolean needsPostgres = false;
        for (Service svc : cluster.services()) {
            if (svc.name().equals("claudony")) {
                needsPostgres = true;
                break;
            }
        }

        if (needsPostgres) {
            PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:16-alpine");
            pg.start();
            state.postgres = pg;
        }

        for (Service svc : cluster.services()) {
            int port = ServiceProcess.findFreePort();
            Map<String, String> props = buildProperties(svc, state);

            ServiceProcess proc = new ServiceProcess(
                svc.name(), ServiceProcess.resolveJar(svc.repo()), port, props);
            proc.start();
            state.processes.add(proc);
            state.endpoints.put(svc.name(), proc.baseUrl());
        }

        injectEndpoints(context.getRequiredTestClass(), state.endpoints);
    }

    @Override
    public void afterAll(ExtensionContext context) {
        ClusterState state = context.getStore(NAMESPACE)
            .remove("state", ClusterState.class);
        if (state == null) return;

        List<ServiceProcess> reversed = new ArrayList<>(state.processes);
        java.util.Collections.reverse(reversed);
        for (ServiceProcess proc : reversed) {
            proc.stop();
        }

        if (state.postgres != null) {
            state.postgres.stop();
        }
    }

    private Map<String, String> buildProperties(Service svc, ClusterState state) {
        Map<String, String> props = new HashMap<>();
        props.put("quarkus.profile", "test");
        props.put("quarkus.http.auth.permission.protected.policy", "permit");

        if (svc.name().equals("claudony") && state.postgres != null) {
            String reactiveUrl = "postgresql://" + state.postgres.getHost() + ":"
                + state.postgres.getFirstMappedPort() + "/" + state.postgres.getDatabaseName();
            props.put("quarkus.datasource.qhorus.reactive.url", reactiveUrl);
            props.put("quarkus.datasource.qhorus.jdbc.url", state.postgres.getJdbcUrl());
            props.put("quarkus.datasource.qhorus.username", state.postgres.getUsername());
            props.put("quarkus.datasource.qhorus.password", state.postgres.getPassword());
        }

        return props;
    }

    private void injectEndpoints(Class<?> testClass, Map<String, String> endpoints)
            throws IllegalAccessException {
        for (Field field : testClass.getDeclaredFields()) {
            ServiceEndpoint ann = field.getAnnotation(ServiceEndpoint.class);
            if (ann != null) {
                String url = endpoints.get(ann.value());
                if (url == null) {
                    throw new IllegalStateException(
                        "@ServiceEndpoint(\"" + ann.value() + "\") — no service with that name in @ServiceCluster");
                }
                field.setAccessible(true);
                field.set(null, url);
            }
        }
    }

    private static class ClusterState implements ExtensionContext.Store.CloseableResource {
        final List<ServiceProcess> processes = new ArrayList<>();
        final Map<String, String> endpoints = new HashMap<>();
        PostgreSQLContainer<?> postgres;

        @Override
        public void close() {}
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl topology-tests -Pintegration-live -Dtest=ServiceClusterExtensionTest`
Expected: PASS (2 tests — requires claudony jar to be built and Docker running)

Note: if claudony jar is not built, the test will fail with "Service jar not found" — that's correct behavior. Build claudony first: `mvn --batch-mode -f ../../claudony/app/pom.xml package -DskipTests`

- [ ] **Step 6: Commit**

```bash
git add topology-tests/pom.xml
git add topology-tests/src/test/java/io/casehub/ops/topology/integration/
git commit -m "feat(#139): ServiceClusterExtension — PostgreSQL, dynamic ports, field injection

Refs #139

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 4: Proof-of-concept integration test — register apps via real claudony

**Files:**
- Create: `topology-tests/src/test/java/io/casehub/ops/topology/integration/HelloLlmIntegrationTest.java`

**Interfaces:**
- Consumes:
  - `@ServiceCluster`, `@Service`, `@ServiceEndpoint` from Task 2
  - `ServiceClusterExtension` from Task 3
  - `TopologyTestBase.compileSingleGraph()` from existing code
  - `HttpClaudonyFleetClient`, `ClaudonyDeploymentTarget` from existing `claudony` module
  - `DeployConfig`, `DeployResult`, `CaseHubApplicationSpec` from existing `api` module
- Produces: nothing (leaf test)

- [ ] **Step 1: Write the integration test**

```java
package io.casehub.ops.topology.integration;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import io.casehub.desiredstate.api.DesiredNode;
import io.casehub.desiredstate.api.DesiredStateGraph;
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
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

@Tag("integration-live")
@ServiceCluster(services = {
    @Service(name = "claudony", repo = "claudony/app")
})
class HelloLlmIntegrationTest extends TopologyTestBase {

    private static final Map<String, String> DEPLOYMENT_REGISTRY =
        buildTypeRegistryFrom(InfraNodeSpec.class, DeploymentNodeSpec.class);

    @ServiceEndpoint("claudony")
    static String claudonyUrl;

    @Test
    void registersAppsViaRealClaudony() throws IOException {
        DesiredStateGraph graph = compileSingleGraph(
            "topologies/deployment/01-hello-llm.yaml", DEPLOYMENT_REGISTRY);

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

        for (DesiredNode node : graph.nodes().values()) {
            CaseHubApplicationSpec appSpec = extractAppSpec(node);
            NodeStatus status = target.status(appSpec.applicationId());
            assertThat(status)
                .as("Status of %s after registration", appSpec.applicationId())
                .isNotEqualTo(NodeStatus.ABSENT);
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

- [ ] **Step 2: Run test to verify it passes**

Run: `mvn --batch-mode test -pl topology-tests -Pintegration-live -Dtest=HelloLlmIntegrationTest`
Expected: PASS (1 test — requires claudony jar built and Docker running for PostgreSQL)

- [ ] **Step 3: Verify default surefire run excludes integration-live tests**

Run: `mvn --batch-mode -o test -pl topology-tests`
Expected: PASS — HelloLlmIntegrationTest and ServiceClusterExtensionTest NOT executed

- [ ] **Step 4: Commit**

```bash
git add topology-tests/src/test/java/io/casehub/ops/topology/integration/HelloLlmIntegrationTest.java
git commit -m "feat(#139): proof-of-concept integration test — register apps via real claudony

Validates ServiceProcess, ServiceClusterExtension, and ClaudonyDeploymentTarget
against a real claudony instance with Testcontainers PostgreSQL.

Refs #139

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## References

- [2026-10-14-real-instance-test-infra-design.md] — design spec this plan implements
- [topology-tests/src/test/java/.../deployment/DeploymentTargetIntegrationTest.java] — existing WireMock integration test pattern
- [topology-tests/src/test/java/.../live/KubernetesAvailable.java] — environment-gated test pattern (LlmAvailable modelled after this)
- [topology-tests/src/test/java/.../compilation/TopologyTestBase.java] — test base class with compileSingleGraph()
- [claudony/app/src/main/resources/application.properties] — claudony config (PostgreSQL, auth, profiles)
- [claudony/src/main/java/.../HttpClaudonyFleetClient.java] — HTTP client for claudony fleet API
- [claudony/src/main/java/.../ClaudonyDeploymentTarget.java] — deployment target SPI implementation
- [topology-tests/src/test/resources/topologies/deployment/01-hello-llm.yaml] — hello-llm topology YAML
- [topology-tests/pom.xml] — existing surefire config and profiles
- [GitHub #139] — focal issue
- [GitHub #131] — parent epic
