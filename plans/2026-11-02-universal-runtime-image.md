# Universal CaseHub Runtime Image — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #223 — Universal CaseHub runtime image
**Issue group:** #223

**Goal:** Build a single universal CaseHub runtime image with all platform and ops modules, rewrite Level 01 topology to deploy a real CaseHub instance, and update ImageResolver to support the universal image.

**Architecture:** New `runtime/` module in casehub-ops — a Quarkus application aggregating all platform + ops deps into one fat image. No module activation mechanism needed — CaseHub modules are inert without declared goals. Level 01 becomes a single `casehub_application` node using `casehubio/casehub-runtime:latest`.

**Tech Stack:** Java 21, Quarkus (fast-jar), Maven, Docker, Podman REST API

## Global Constraints

- Java 21, Quarkus with fast-jar packaging
- Parent POM: `casehub-ops-parent` version `0.4-SNAPSHOT`
- Inter-module deps use `${project.version}` for ops modules
- Platform deps managed by `casehub-parent` BOM — no version tags needed
- Module type: Quarkus application (not Jandex library — uses `quarkus-maven-plugin`)
- Commit messages: `feat(#223): <description>`

---

## Batch 1: Runtime module + Dockerfile

### Task 1: Create runtime/ module with pom.xml

**Files:**
- Create: `runtime/pom.xml`
- Modify: `pom.xml` (parent — add `<module>runtime</module>`)

**Interfaces:**
- Consumes: `casehub-ops-service` (all platform + ops service logic), `casehub-ops-deployment`, `casehub-ops-compliance`, `casehub-ops-claudony`, `casehub-engine-rest`
- Produces: `casehub-ops-runtime` artifact (Quarkus fast-jar)

- [ ] **Step 1: Write a smoke test that verifies the Quarkus app starts**

Create `runtime/src/test/java/io/casehub/ops/runtime/RuntimeStartupTest.java`:

```java
package io.casehub.ops.runtime;

import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.enterprise.inject.Instance;
import io.casehub.desiredstate.api.GoalCompiler;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class RuntimeStartupTest {

    @Inject
    Instance<GoalCompiler<?>> goalCompilers;

    @Test
    void applicationStarts_andGoalCompilersAreDiscovered() {
        assertThat(goalCompilers.stream().count())
            .as("At least deployment + infra + compliance GoalCompilers should be discovered")
            .isGreaterThanOrEqualTo(3);
    }
}
```

- [ ] **Step 2: Run test to verify it fails (module doesn't exist yet)**

Run: `mvn --batch-mode test -pl runtime`
Expected: FAIL — module not found

- [ ] **Step 3: Create runtime/pom.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-ops-parent</artifactId>
        <version>0.4-SNAPSHOT</version>
    </parent>

    <artifactId>casehub-ops-runtime</artifactId>

    <name>CaseHub Ops :: Runtime — Universal Image</name>
    <description>Universal CaseHub runtime image. All platform and ops modules on
        the classpath, activated by runtime configuration. One image, many personalities.</description>

    <dependencies>
        <!-- Core: ops-service brings platform + engine + desiredstate + work -->
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-ops-service</artifactId>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-engine-rest</artifactId>
        </dependency>

        <!-- Ops domain modules -->
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-ops-deployment</artifactId>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-ops-compliance</artifactId>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-ops-claudony</artifactId>
        </dependency>

        <!-- Quarkus application packaging -->
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-smallrye-graphql</artifactId>
        </dependency>

        <!-- Test -->
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-junit</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-platform-testing</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-desiredstate-testing</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>io.quarkus</groupId>
                <artifactId>quarkus-maven-plugin</artifactId>
                <version>${quarkus.platform.version}</version>
                <extensions>true</extensions>
                <executions>
                    <execution>
                        <goals>
                            <goal>build</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>

    <profiles>
        <profile>
            <id>skip-quarkus-tests</id>
            <activation>
                <property>
                    <name>!runQuarkusTests</name>
                </property>
            </activation>
            <properties>
                <skipTests>true</skipTests>
            </properties>
        </profile>
    </profiles>
</project>
```

- [ ] **Step 4: Create minimal application.properties**

Create `runtime/src/main/resources/application.properties`:

```properties
quarkus.http.port=8080
quarkus.datasource.db-kind=h2
quarkus.datasource.jdbc.url=jdbc:h2:mem:casehub-runtime;DB_CLOSE_DELAY=-1
quarkus.hibernate-orm.database.generation=drop-and-create
quarkus.flyway.migrate-at-start=false
```

- [ ] **Step 5: Add runtime module to parent pom.xml**

In `pom.xml`, add `<module>runtime</module>` to the `<modules>` section (after `topology-tests`).

Use `ide_replace_text_in_file` or Edit tool on `pom.xml`:

```xml
        <module>topology-tests</module>
        <module>runtime</module>
        <module>examples/deployment-monitoring</module>
```

Also add the inter-module dependency management entry in `<dependencyManagement>`:

```xml
            <dependency>
                <groupId>io.casehub</groupId>
                <artifactId>casehub-ops-runtime</artifactId>
                <version>${project.version}</version>
            </dependency>
```

- [ ] **Step 6: Run test to verify the Quarkus app starts**

Run: `mvn --batch-mode test -pl runtime -am -DrunQuarkusTests`
Expected: PASS — Quarkus starts, 3+ GoalCompilers discovered (deployment, infra, compliance)

- [ ] **Step 7: Commit**

```bash
git add runtime/ pom.xml
git commit -m "feat(#223): add runtime/ module — universal CaseHub runtime image"
```

### Task 2: Add Dockerfile.jvm

**Files:**
- Create: `runtime/src/main/docker/Dockerfile.jvm`

**Interfaces:**
- Consumes: Quarkus fast-jar output in `runtime/target/quarkus-app/`
- Produces: `casehubio/casehub-runtime:latest` Docker image

- [ ] **Step 1: Write a test script that validates Docker image structure**

This is a manual verification step (not a JUnit test). The acceptance criteria: `docker build` succeeds and the image contains `quarkus-run.jar`.

- [ ] **Step 2: Create the Dockerfile**

Create `runtime/src/main/docker/Dockerfile.jvm`:

```dockerfile
FROM eclipse-temurin:21-jre-alpine

WORKDIR /app

COPY target/quarkus-app/lib/ /app/lib/
COPY target/quarkus-app/*.jar /app/
COPY target/quarkus-app/app/ /app/app/
COPY target/quarkus-app/quarkus/ /app/quarkus/

EXPOSE 8080

ENV JAVA_OPTS_APPEND=""

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS_APPEND -jar /app/quarkus-run.jar"]
```

- [ ] **Step 3: Build the fast-jar and Docker image**

Run:
```bash
mvn --batch-mode package -pl runtime -am -DskipTests
docker build -t casehubio/casehub-runtime:latest -f runtime/src/main/docker/Dockerfile.jvm runtime/
```
Expected: Image builds successfully.

- [ ] **Step 4: Verify the image starts**

Run:
```bash
docker run --rm -d --name casehub-runtime-test -p 8080:8080 casehubio/casehub-runtime:latest
sleep 5
curl -s http://localhost:8080/q/health/ready || true
docker stop casehub-runtime-test
```
Expected: Container starts (may not have health endpoint configured — that's OK, startup is the gate).

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/docker/Dockerfile.jvm
git commit -m "feat(#223): add Dockerfile.jvm for universal runtime image"
```

---

## Batch 2: ImageResolver + Level 01 topology rewrite

### Task 3: Add universal image support to ImageResolver

**Files:**
- Modify: `testing/src/main/java/io/casehub/ops/testing/env/ImageResolver.java`
- Modify: `testing/src/test/java/io/casehub/ops/testing/env/ImageResolverTest.java`

**Interfaces:**
- Consumes: Existing `ImageSource` enum, `resolve()` methods
- Produces: `UNIVERSAL_IMAGE_NAME` constant, `resolveUniversal(String version)` method, `resolveUniversal(String version, boolean allowFallback)` method

- [ ] **Step 1: Write failing tests for resolveUniversal**

Add to `ImageResolverTest.java`:

```java
@Test
void resolveUniversal_local_returnsUniversalImageRef() {
    // This test will fail if the image isn't built locally — that's expected
    // in CI without a prior build. Test the ref format, not existence.
    String ref = ImageResolver.imageRef(ImageResolver.UNIVERSAL_IMAGE_NAME, "latest");
    assertThat(ref).isEqualTo("casehubio/casehub-runtime:latest");
}

@Test
void resolveUniversal_fallback_returnsNginxAlpine() {
    String result = ImageResolver.resolveUniversal("latest", true);
    assertThat(result).isEqualTo(ImageResolver.FALLBACK_IMAGE);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn --batch-mode test -pl testing -Dtest=ImageResolverTest`
Expected: FAIL — `UNIVERSAL_IMAGE_NAME` not defined

- [ ] **Step 3: Add universal image constant and methods**

Use `ide_insert_member` to add to `ImageResolver.java` after the existing constants:

```java
public static final String UNIVERSAL_IMAGE_NAME = "casehub-runtime";

public static String resolveUniversal(String version) {
    return resolve(UNIVERSAL_IMAGE_NAME, version);
}

public static String resolveUniversal(String version, boolean allowFallback) {
    return resolve(UNIVERSAL_IMAGE_NAME, version, allowFallback);
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl testing -Dtest=ImageResolverTest`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add testing/src/main/java/io/casehub/ops/testing/env/ImageResolver.java
git add testing/src/test/java/io/casehub/ops/testing/env/ImageResolverTest.java
git commit -m "feat(#223): add universal image support to ImageResolver"
```

### Task 4: Rewrite Level 01 topology YAML

**Files:**
- Modify: `topology-tests/src/test/resources/topologies/deployment/01-hello-llm/01-hello-llm.yaml`
- Modify: `topology-tests/src/test/resources/topologies/deployment/01-hello-llm/topology-metadata.yaml`
- Modify: `topology-tests/src/test/resources/topologies/deployment/01-hello-llm/playbook.yaml`
- Modify: `service/src/main/resources/META-INF/desiredstate/topologies/01-hello-llm/01-hello-llm.yaml` (mirror copy)

**Interfaces:**
- Consumes: `casehub_application` node type, `CaseHubApplicationSpec` fields
- Produces: Updated Level 01 YAML with single `my-llm` node using universal image

- [ ] **Step 1: Verify existing compilation test reads this YAML**

Check that the topology compilation test compiles `01-hello-llm.yaml`. Find the test class:

```bash
ide_search_text query="01-hello-llm" filePattern="*Test.java"
```

Identify which test exercises Level 01 compilation so we know what to update.

- [ ] **Step 2: Rewrite 01-hello-llm.yaml (topology-tests copy)**

Replace the file content with:

```yaml
# Level 01: Hello LLM
# Methodology step: DEPLOY — First App
# Prerequisites: None

desiredState:
  namespace: casehub
  name: hello-llm

nodes:
  my-llm:
    type: casehub_application
    spec:
      applicationId: my-llm
      image: casehubio/casehub-runtime:latest
      deploymentTarget: claudony
      replicas: 1
      config:
        llm.capability: fast
```

- [ ] **Step 3: Update topology-metadata.yaml**

Replace content:

```yaml
description: "Deploy a CaseHub LLM agent"
methodologyStep: "DEPLOY — First App"
prerequisites: "None"
level: 1
```

- [ ] **Step 4: Update playbook.yaml**

Replace content:

```yaml
- step: "Provision my-llm agent"
  block:
    - provision-node:
        value: my-llm
        type: casehub_application
```

- [ ] **Step 5: Mirror the YAML to service module**

Copy the same `01-hello-llm.yaml` content to `service/src/main/resources/META-INF/desiredstate/topologies/01-hello-llm/01-hello-llm.yaml`.

- [ ] **Step 6: Update DeploymentTargetIntegrationTest**

The test at `topology-tests/src/test/java/io/casehub/ops/topology/deployment/DeploymentTargetIntegrationTest.java` currently:
- Stubs health for 4 apps (manifest, eidos, engine, my-llm)
- Verifies 4 POST requests to claudony
- Checks all 4 app statuses are PRESENT

Update to:
- Stub health only for `my-llm`
- Verify 1 POST request
- Check `my-llm` status is PRESENT

```java
@BeforeEach
void setUp() {
    wireMock = new WireMockServer(WireMockConfiguration.wireMockConfig().dynamicPort());
    wireMock.start();

    ClaudonyWireMockSupport.stubRegisterApp(wireMock);
    ClaudonyWireMockSupport.stubGetAppHealthy(wireMock, "my-llm");

    claudonyTarget = createClaudonyTarget(wireMock.baseUrl());
}
```

Update `helloLlm_deploysAllAppsViaClaudony`:

```java
@Test
void helloLlm_deploysAllAppsViaClaudony() throws IOException {
    DesiredStateGraph graph = compileSingleGraph(
            "topologies/deployment/01-hello-llm/01-hello-llm.yaml", DEPLOYMENT_REGISTRY);

    deployAllNodes(graph, claudonyTarget);

    wireMock.verify(1, postRequestedFor(urlEqualTo("/api/claudony/apps/")));
}
```

- [ ] **Step 7: Run topology compilation and deployment tests**

Run: `mvn --batch-mode -o test -pl topology-tests -Dtest="*Level01*,DeploymentTargetIntegrationTest"`

If no Level01-specific test class exists, run:
```bash
mvn --batch-mode -o test -pl topology-tests -Dtest=DeploymentTargetIntegrationTest
```
Expected: PASS — single node compiles and deploys

- [ ] **Step 8: Run full topology-tests to verify no regressions**

Run: `mvn --batch-mode -o test -pl topology-tests`
Expected: PASS — all existing tests pass (Level 02-15 unchanged)

- [ ] **Step 9: Commit**

```bash
git add topology-tests/src/test/resources/topologies/deployment/01-hello-llm/
git add service/src/main/resources/META-INF/desiredstate/topologies/01-hello-llm/
git add topology-tests/src/test/java/io/casehub/ops/topology/deployment/DeploymentTargetIntegrationTest.java
git commit -m "feat(#223): rewrite Level 01 topology — single node with universal runtime image"
```

---

## Batch 3: Update docs + verify full build

### Task 5: Update documentation and verify full build

**Files:**
- Modify: `docs/guides/consumer-guide.md` (update Module Structure table, add runtime module section)
- Modify: `CLAUDE.md` (add runtime module to Module Structure table)

**Interfaces:**
- Consumes: All changes from Tasks 1-4
- Produces: Updated docs, green full build

- [ ] **Step 1: Add runtime module to CLAUDE.md Module Structure table**

Add row after `app/`:

```
| `runtime/` | `casehub-ops-runtime` | `io.casehub.ops.runtime` | Universal CaseHub runtime image — all platform + ops modules, one image, config-driven behavior |
```

- [ ] **Step 2: Add runtime section to consumer-guide.md**

After the "App Module — Ops Console" section, add:

```markdown
---

## Runtime Module — Universal Image

### Purpose

Universal CaseHub runtime image with all platform and ops modules on the classpath. Designed for topology tests and Podman-based deployments where a single image serves all CaseHub personalities. Runtime behavior is controlled by the `config` map in `CaseHubApplicationSpec` — modules are inert without declared goals.

### Dependencies

All compile-scope dependencies from `casehub-ops-service` plus:

| Artifact | Nature |
|----------|--------|
| `casehub-ops-deployment` | Deployment domain |
| `casehub-ops-compliance` | Compliance domain |
| `casehub-ops-claudony` | Claudony fleet management |
| `casehub-engine-rest` | Engine REST endpoints |

### Building the Image

\`\`\`bash
mvn --batch-mode package -pl runtime -am -DskipTests
docker build -t casehubio/casehub-runtime:latest -f runtime/src/main/docker/Dockerfile.jvm runtime/
\`\`\`
```

- [ ] **Step 3: Update Build Commands in CLAUDE.md**

Add the runtime image build command:

```bash
mvn --batch-mode package -pl runtime -am -DskipTests   # build runtime fast-jar
docker build -t casehubio/casehub-runtime:latest -f runtime/src/main/docker/Dockerfile.jvm runtime/  # build image
```

- [ ] **Step 4: Run full Maven build**

Run: `mvn --batch-mode install`
Expected: PASS — all modules compile, all tests pass

- [ ] **Step 5: Commit**

```bash
git add docs/guides/consumer-guide.md CLAUDE.md
git commit -m "feat(#223): update docs — add runtime module to consumer guide and CLAUDE.md"
```

---

## References

- [2026-11-02-universal-runtime-image-design.md] — design spec this plan implements
- `api/src/main/java/io/casehub/ops/api/deployment/CaseHubApplicationSpec.java` — casehub_application node spec
- `api/src/main/java/io/casehub/ops/api/deployment/RuntimeConfig.java` — engine + modules
- `deployment/src/main/java/io/casehub/ops/deployment/handler/CaseHubAppProvisionHandler.java` — provisioning
- `container/src/main/java/io/casehub/ops/container/PodmanDeploymentTarget.java` — Podman target
- `testing/src/main/java/io/casehub/ops/testing/env/ImageResolver.java` — image resolution
- `topology-tests/src/test/resources/topologies/deployment/01-hello-llm/` — Level 01 topology
- `service/pom.xml` — existing platform deps (validates coexistence)
- `app/pom.xml` — existing Quarkus shell pattern
- GitHub #223 — focal issue
