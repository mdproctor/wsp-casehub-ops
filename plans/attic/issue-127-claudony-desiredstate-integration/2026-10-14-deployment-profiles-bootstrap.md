# Deployment Profiles and Bootstrap Model Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #117 — deployment profiles and bootstrap model
**Issue group:** #117

**Goal:** Enable the ops app to deploy and manage a full CaseHub installation from a single topology YAML declaration across three deployment profiles (local-dev, local-distributed, enterprise), with two-phase bootstrap and self-maintenance.

**Architecture:** Platform services (Qhorus, Eidos, Engine) are modeled as `CaseHubApplicationSpec` instances provisioned through `ApplicationDeploymentTarget` implementations selected by a CaseHub-level deployment profile config property. A `DeploymentApplicationNodeSpec` wrapper (mirroring `InfraDesiredNodeSpec`) carries per-node `deploymentTarget`/`existing` overrides extracted from the YAML spec map by `DeploymentNodeSpecFactoryProvider`. A `BootstrapDomainRegistrar` wires `CrossDomainCompositionEngine` at startup with profile-aware infra/deployment domain registration.

**Tech Stack:** Java 21, Quarkus (CDI, MicroProfile Config), casehub-desiredstate (GoalCompiler, NodeProvisioner, ActualStateAdapter, CrossDomainCompositionEngine), Podman REST API (via existing PodmanClient), fabric8 Kubernetes client.

## Global Constraints

- Java 21, Maven build (`mvn --batch-mode install`)
- Module type: Jandex library, CDI discovery, zero-config activation
- `tenancyId` propagated through all calls — bind in repository/adapter layer only
- Deployment profile is `casehub.ops.deployment-profile` config property, NOT a Quarkus profile
- `existing:` and `deploymentTarget:` live inside the YAML `spec:` block (following infra's `backendId` pattern)
- `DeploymentApplicationNodeSpec` implements `NodeSpec` (not `DeploymentNodeSpec`) — sits outside the sealed switch
- All `ApplicationDeploymentTarget` implementations are `@ApplicationScoped` CDI beans coexisting on classpath
- `NoOpDeploymentTarget` (`@DefaultBean`) unchanged — inherits default `targetId() = ""`

---

## Batch 1: Multi-Target Deployment Foundation

After this batch: `ApplicationDeploymentTarget` supports multi-target resolution by `targetId()`, `DeploymentApplicationNodeSpec` wrapper exists for per-node overrides, `DeploymentNodeSpecFactoryProvider` enables `casehub_application` nodes in `desiredState:` YAML format, and the provisioner dispatches to the correct target.

### Task 1: DeploymentApplicationNodeSpec wrapper + targetId() on SPI

**Files:**
- Modify: `api/src/main/java/io/casehub/ops/api/deployment/ApplicationDeploymentTarget.java`
- Create: `api/src/main/java/io/casehub/ops/api/deployment/DeploymentApplicationNodeSpec.java`
- Test: `api/src/test/java/io/casehub/ops/api/deployment/DeploymentApplicationNodeSpecTest.java`

**Interfaces:**
- Produces: `DeploymentApplicationNodeSpec(CaseHubApplicationSpec spec, String deploymentTargetId, String existingUrl)` implementing `NodeSpec`
- Produces: `ApplicationDeploymentTarget.targetId()` default method returning `""`

- [ ] **Step 1: Write the failing test for DeploymentApplicationNodeSpec**

```java
package io.casehub.ops.api.deployment;

import io.casehub.desiredstate.api.NodeSpec;
import io.casehub.desiredstate.api.NodeType;
import org.junit.jupiter.api.Test;
import java.util.Map;
import static org.junit.jupiter.api.Assertions.*;

class DeploymentApplicationNodeSpecTest {

    private static final CaseHubApplicationSpec APP_SPEC = new CaseHubApplicationSpec(
            "qhorus", null, "casehubio/qhorus:0.5", null, Map.of(), java.util.List.of(), 1);

    @Test
    void wrapsSpecWithTargetId() {
        var wrapper = new DeploymentApplicationNodeSpec(APP_SPEC, "podman", null);
        assertInstanceOf(NodeSpec.class, wrapper);
        assertEquals(APP_SPEC, wrapper.spec());
        assertEquals("podman", wrapper.deploymentTargetId());
        assertNull(wrapper.existingUrl());
        assertEquals("qhorus", wrapper.nodeId());
        assertEquals(NodeType.of("casehub_application"), wrapper.nodeType());
    }

    @Test
    void wrapsSpecWithExistingUrl() {
        var wrapper = new DeploymentApplicationNodeSpec(APP_SPEC, null, "https://qhorus.internal:8443");
        assertNull(wrapper.deploymentTargetId());
        assertEquals("https://qhorus.internal:8443", wrapper.existingUrl());
    }

    @Test
    void delegatesNodeIdAndNodeType() {
        var wrapper = new DeploymentApplicationNodeSpec(APP_SPEC, "podman", null);
        assertEquals(APP_SPEC.nodeId(), wrapper.nodeId());
        assertEquals(APP_SPEC.nodeType(), wrapper.nodeType());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode -o test -pl api -Dtest=DeploymentApplicationNodeSpecTest`
Expected: FAIL — `DeploymentApplicationNodeSpec` does not exist

- [ ] **Step 3: Create DeploymentApplicationNodeSpec and add targetId() to SPI**

`api/src/main/java/io/casehub/ops/api/deployment/DeploymentApplicationNodeSpec.java`:
```java
package io.casehub.ops.api.deployment;

import io.casehub.desiredstate.api.NodeSpec;
import io.casehub.desiredstate.api.NodeType;

public record DeploymentApplicationNodeSpec(
    CaseHubApplicationSpec spec,
    String deploymentTargetId,
    String existingUrl
) implements NodeSpec {

    @Override
    public String nodeId() { return spec.nodeId(); }

    @Override
    public NodeType nodeType() { return spec.nodeType(); }
}
```

Add to `ApplicationDeploymentTarget.java` (before `deploy` method):
```java
default String targetId() { return ""; }
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode -o test -pl api -Dtest=DeploymentApplicationNodeSpecTest`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add api/src/main/java/io/casehub/ops/api/deployment/DeploymentApplicationNodeSpec.java api/src/main/java/io/casehub/ops/api/deployment/ApplicationDeploymentTarget.java api/src/test/java/io/casehub/ops/api/deployment/DeploymentApplicationNodeSpecTest.java
git commit -m "feat(#117): add DeploymentApplicationNodeSpec wrapper and targetId() to ApplicationDeploymentTarget"
```

---

### Task 2: DeploymentNodeSpecFactoryProvider

**Files:**
- Create: `deployment/src/main/java/io/casehub/ops/deployment/DeploymentNodeSpecFactoryProvider.java`
- Test: `deployment/src/test/java/io/casehub/ops/deployment/DeploymentNodeSpecFactoryProviderTest.java`

**Interfaces:**
- Consumes: `DeploymentApplicationNodeSpec(CaseHubApplicationSpec, String, String)` from Task 1
- Consumes: `CaseHubApplicationSpec` constructor
- Produces: `DeploymentNodeSpecFactoryProvider implements NodeSpecFactoryProvider` — registers `casehub_application` type

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.ops.deployment;

import io.casehub.desiredstate.api.NodeSpec;
import io.casehub.ops.api.deployment.CaseHubApplicationSpec;
import io.casehub.ops.api.deployment.DeploymentApplicationNodeSpec;
import org.junit.jupiter.api.Test;
import java.util.LinkedHashMap;
import java.util.Map;
import static org.junit.jupiter.api.Assertions.*;

class DeploymentNodeSpecFactoryProviderTest {

    private final DeploymentNodeSpecFactoryProvider provider = new DeploymentNodeSpecFactoryProvider();

    @Test
    void registersCaseHubApplicationType() {
        var factories = provider.provide();
        assertTrue(factories.containsKey("casehub_application"));
    }

    @Test
    void createsBareCaseHubApplicationSpec() {
        var factory = provider.provide().get("casehub_application");
        Map<String, Object> specMap = new LinkedHashMap<>(Map.of(
                "applicationId", "qhorus",
                "image", "casehubio/qhorus:0.5",
                "replicas", 1));
        NodeSpec result = factory.create(specMap);
        assertInstanceOf(CaseHubApplicationSpec.class, result);
        assertEquals("qhorus", ((CaseHubApplicationSpec) result).applicationId());
    }

    @Test
    void wrapsWhenDeploymentTargetPresent() {
        var factory = provider.provide().get("casehub_application");
        Map<String, Object> specMap = new LinkedHashMap<>(Map.of(
                "applicationId", "qhorus",
                "image", "casehubio/qhorus:0.5",
                "replicas", 1,
                "deploymentTarget", "podman"));
        NodeSpec result = factory.create(specMap);
        assertInstanceOf(DeploymentApplicationNodeSpec.class, result);
        var wrapper = (DeploymentApplicationNodeSpec) result;
        assertEquals("podman", wrapper.deploymentTargetId());
        assertEquals("qhorus", wrapper.spec().applicationId());
    }

    @Test
    void wrapsWhenExistingPresent() {
        var factory = provider.provide().get("casehub_application");
        Map<String, Object> specMap = new LinkedHashMap<>(Map.of(
                "applicationId", "qhorus",
                "image", "casehubio/qhorus:0.5",
                "replicas", 1,
                "existing", "https://qhorus.internal:8443"));
        NodeSpec result = factory.create(specMap);
        assertInstanceOf(DeploymentApplicationNodeSpec.class, result);
        var wrapper = (DeploymentApplicationNodeSpec) result;
        assertEquals("https://qhorus.internal:8443", wrapper.existingUrl());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode -o test -pl deployment -Dtest=DeploymentNodeSpecFactoryProviderTest`
Expected: FAIL — class not found

- [ ] **Step 3: Implement DeploymentNodeSpecFactoryProvider**

`deployment/src/main/java/io/casehub/ops/deployment/DeploymentNodeSpecFactoryProvider.java`:
```java
package io.casehub.ops.deployment;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.desiredstate.api.NodeSpec;
import io.casehub.desiredstate.api.NodeSpecFactory;
import io.casehub.desiredstate.api.NodeSpecFactoryProvider;
import io.casehub.ops.api.deployment.CaseHubApplicationSpec;
import io.casehub.ops.api.deployment.DeploymentApplicationNodeSpec;

import java.util.LinkedHashMap;
import java.util.Map;

public class DeploymentNodeSpecFactoryProvider implements NodeSpecFactoryProvider {

    private static final ObjectMapper MAPPER = new ObjectMapper();

    @Override
    public Map<String, NodeSpecFactory> provide() {
        return Map.of("casehub_application", this::createCaseHubApplication);
    }

    private NodeSpec createCaseHubApplication(Map<String, Object> specMap) {
        Map<String, Object> fields = new LinkedHashMap<>(specMap);
        String deploymentTarget = (String) fields.remove("deploymentTarget");
        String existing = (String) fields.remove("existing");
        CaseHubApplicationSpec appSpec = MAPPER.convertValue(fields, CaseHubApplicationSpec.class);
        if (deploymentTarget != null || existing != null) {
            return new DeploymentApplicationNodeSpec(appSpec, deploymentTarget, existing);
        }
        return appSpec;
    }
}
```

Register as SPI: create `deployment/src/main/resources/META-INF/services/io.casehub.desiredstate.api.NodeSpecFactoryProvider`:
```
io.casehub.ops.deployment.DeploymentNodeSpecFactoryProvider
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode -o test -pl deployment -Dtest=DeploymentNodeSpecFactoryProviderTest`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add deployment/src/main/java/io/casehub/ops/deployment/DeploymentNodeSpecFactoryProvider.java deployment/src/test/java/io/casehub/ops/deployment/DeploymentNodeSpecFactoryProviderTest.java deployment/src/main/resources/META-INF/services/io.casehub.desiredstate.api.NodeSpecFactoryProvider
git commit -m "feat(#117): add DeploymentNodeSpecFactoryProvider for casehub_application YAML processing"
```

---

### Task 3: Update provisioner and handler for multi-target dispatch

**Files:**
- Modify: `deployment/src/main/java/io/casehub/ops/deployment/DeploymentNodeProvisioner.java`
- Modify: `deployment/src/main/java/io/casehub/ops/deployment/handler/CaseHubAppProvisionHandler.java`
- Modify: `deployment/src/main/java/io/casehub/ops/deployment/drift/CaseHubAppDriftChecker.java`
- Test: `deployment/src/test/java/io/casehub/ops/deployment/handler/CaseHubAppProvisionHandlerMultiTargetTest.java`

**Interfaces:**
- Consumes: `DeploymentApplicationNodeSpec` from Task 1
- Consumes: `ApplicationDeploymentTarget.targetId()` from Task 1
- Produces: `CaseHubAppProvisionHandler.provision(CaseHubApplicationSpec, ProvisionContext, String deploymentTargetId, String existingUrl)` overload

- [ ] **Step 1: Write the failing test for multi-target handler**

```java
package io.casehub.ops.deployment.handler;

import io.casehub.desiredstate.api.NodeStatus;
import io.casehub.desiredstate.api.ProvisionContext;
import io.casehub.desiredstate.api.ProvisionResult;
import io.casehub.ops.api.deployment.*;
import io.casehub.ops.deployment.InstanceIdStore;
import jakarta.enterprise.inject.Instance;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import java.util.stream.Stream;
import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

class CaseHubAppProvisionHandlerMultiTargetTest {

    private static final CaseHubApplicationSpec SPEC = new CaseHubApplicationSpec(
            "qhorus", null, "casehubio/qhorus:0.5", null, Map.of(), List.of(), 1);

    @Test
    void resolvesTargetByIdWhenDeploymentTargetIdProvided() {
        var podmanTarget = createTarget("podman");
        var inProcessTarget = createTarget("in-process");
        var defaultTarget = createTarget("");

        @SuppressWarnings("unchecked")
        Instance<ApplicationDeploymentTarget> targets = mock(Instance.class);
        when(targets.stream()).thenReturn(Stream.of(podmanTarget, inProcessTarget, defaultTarget));

        var handler = new CaseHubAppProvisionHandler(
                new PrebuiltImagePipeline(), defaultTarget, new InstanceIdStore(), targets);

        var ctx = mock(ProvisionContext.class);
        when(ctx.tenancyId()).thenReturn("default");

        var result = handler.provision(SPEC, ctx, "podman", null);
        assertInstanceOf(ProvisionResult.Success.class, result);
        verify(podmanTarget).deploy(eq("casehubio/qhorus:0.5"), any());
        verify(inProcessTarget, never()).deploy(any(), any());
    }

    @Test
    void skipsProvisioningWhenExistingUrlSet() {
        var defaultTarget = createTarget("");

        @SuppressWarnings("unchecked")
        Instance<ApplicationDeploymentTarget> targets = mock(Instance.class);
        when(targets.stream()).thenReturn(Stream.of(defaultTarget));

        var handler = new CaseHubAppProvisionHandler(
                new PrebuiltImagePipeline(), defaultTarget, new InstanceIdStore(), targets);

        var ctx = mock(ProvisionContext.class);
        when(ctx.tenancyId()).thenReturn("default");

        var result = handler.provision(SPEC, ctx, null, "https://qhorus.internal:8443");
        assertInstanceOf(ProvisionResult.Success.class, result);
        verify(defaultTarget, never()).deploy(any(), any());
    }

    private ApplicationDeploymentTarget createTarget(String targetId) {
        var target = mock(ApplicationDeploymentTarget.class);
        when(target.targetId()).thenReturn(targetId);
        when(target.deploy(any(), any())).thenReturn(new DeployResult.Success("instance-1"));
        when(target.healthCheck(any())).thenReturn(new HealthCheckResult.Healthy("instance-1"));
        return target;
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode -o test -pl deployment -Dtest=CaseHubAppProvisionHandlerMultiTargetTest`
Expected: FAIL — constructor overload and method overload don't exist

- [ ] **Step 3: Update CaseHubAppProvisionHandler**

Add `Instance<ApplicationDeploymentTarget>` injection and multi-target overload:

```java
// New field
private final Instance<ApplicationDeploymentTarget> allTargets;

// Updated constructor — add Instance<ApplicationDeploymentTarget> parameter
@Inject
public CaseHubAppProvisionHandler(
        ApplicationBuildPipeline buildPipeline,
        ApplicationDeploymentTarget deployTarget,
        InstanceIdStore instanceIdStore,
        @Any Instance<ApplicationDeploymentTarget> allTargets) {
    this.buildPipeline   = buildPipeline;
    this.deployTarget    = deployTarget;
    this.instanceIdStore = instanceIdStore;
    this.allTargets      = allTargets;
}

// New overload for multi-target dispatch
public ProvisionResult provision(CaseHubApplicationSpec spec, ProvisionContext context,
                                  String deploymentTargetId, String existingUrl) {
    if (existingUrl != null) {
        return new ProvisionResult.Success();
    }
    ApplicationDeploymentTarget target = resolveTarget(deploymentTargetId);
    return provisionWith(spec, context, target);
}

private ApplicationDeploymentTarget resolveTarget(String targetId) {
    if (targetId == null || targetId.isEmpty()) return deployTarget;
    return allTargets.stream()
            .filter(t -> targetId.equals(t.targetId()))
            .findFirst()
            .orElse(deployTarget);
}

private ProvisionResult provisionWith(CaseHubApplicationSpec spec, ProvisionContext context,
                                       ApplicationDeploymentTarget target) {
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
    var deployResult = target.deploy(imageRef, deployConfig);

    if (deployResult instanceof DeployResult.Failed f) {
        return new ProvisionResult.Failed("Deploy failed: " + f.reason());
    }

    var instanceId = ((DeployResult.Success) deployResult).instanceId();
    var health     = target.healthCheck(instanceId);
    if (health instanceof HealthCheckResult.Unhealthy u) {
        return new ProvisionResult.Failed("Health check failed: " + u.reason());
    }

    instanceIdStore.record(NodeId.of(spec.applicationId()), instanceId);
    return new ProvisionResult.Success();
}
```

Refactor existing `provision(CaseHubApplicationSpec, ProvisionContext)` to delegate to `provisionWith`:
```java
public ProvisionResult provision(CaseHubApplicationSpec spec, ProvisionContext context) {
    return provisionWith(spec, context, deployTarget);
}
```

- [ ] **Step 4: Update DeploymentNodeProvisioner to handle DeploymentApplicationNodeSpec**

In `DeploymentNodeProvisioner.provision()`, before the existing `DeploymentNodeSpec` check (line 71), add:

```java
if (node.spec() instanceof DeploymentApplicationNodeSpec wrapper) {
    return appHandler.provision(wrapper.spec(), context, wrapper.deploymentTargetId(), wrapper.existingUrl());
}
```

Same pattern for `deprovision()`.

- [ ] **Step 5: Update CaseHubAppDriftChecker for existing URL probing**

In `CaseHubAppDriftChecker.check()`, add handling for `DeploymentApplicationNodeSpec`:

```java
@Override
public NodeStatus check(NodeSpec spec, String tenancyId) {
    if (spec instanceof DeploymentApplicationNodeSpec wrapper) {
        if (wrapper.existingUrl() != null) {
            return probeExistingUrl(wrapper.existingUrl());
        }
        return checkAppStatus(wrapper.spec());
    }
    if (!(spec instanceof CaseHubApplicationSpec appSpec)) {
        return NodeStatus.UNKNOWN;
    }
    return checkAppStatus(appSpec);
}

private NodeStatus checkAppStatus(CaseHubApplicationSpec appSpec) {
    var nodeId     = NodeId.of(appSpec.applicationId());
    var instanceId = instanceIdStore.get(nodeId).orElse(appSpec.applicationId() + "-instance");
    return deployTarget.status(instanceId);
}

private NodeStatus probeExistingUrl(String url) {
    try {
        var conn = (java.net.HttpURLConnection) java.net.URI.create(url).toURL().openConnection();
        conn.setRequestMethod("GET");
        conn.setConnectTimeout(5000);
        conn.setReadTimeout(5000);
        int code = conn.getResponseCode();
        return (code >= 200 && code < 400) ? NodeStatus.PRESENT : NodeStatus.DRIFTED;
    } catch (Exception e) {
        return NodeStatus.ABSENT;
    }
}
```

- [ ] **Step 6: Run all deployment module tests**

Run: `mvn --batch-mode -o test -pl deployment`
Expected: PASS — existing tests still pass, new multi-target tests pass

- [ ] **Step 7: Commit**

```bash
git add deployment/src/main/java/io/casehub/ops/deployment/DeploymentNodeProvisioner.java deployment/src/main/java/io/casehub/ops/deployment/handler/CaseHubAppProvisionHandler.java deployment/src/main/java/io/casehub/ops/deployment/drift/CaseHubAppDriftChecker.java deployment/src/test/java/io/casehub/ops/deployment/handler/CaseHubAppProvisionHandlerMultiTargetTest.java
git commit -m "feat(#117): multi-target dispatch in provisioner and handler, existing URL probing in drift checker"
```

---

## Batch 2: Deployment Target Implementations

After this batch: `InProcessDeploymentTarget` verifies CDI bean availability (local-dev), `PodmanDeploymentTarget` deploys via Podman containers (local-distributed). Both coexist on classpath, resolved by `targetId()`.

### Task 4: InProcessDeploymentTarget

**Files:**
- Create: `deployment/src/main/java/io/casehub/ops/deployment/target/InProcessDeploymentTarget.java`
- Test: `deployment/src/test/java/io/casehub/ops/deployment/target/InProcessDeploymentTargetTest.java`

**Interfaces:**
- Consumes: `ApplicationDeploymentTarget` SPI with `targetId()`
- Produces: `InProcessDeploymentTarget` with `targetId() = "in-process"`

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.ops.deployment.target;

import io.casehub.desiredstate.api.NodeStatus;
import io.casehub.ops.api.deployment.*;
import jakarta.enterprise.inject.Instance;
import org.junit.jupiter.api.Test;
import java.util.Map;
import java.util.stream.Stream;
import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

class InProcessDeploymentTargetTest {

    @Test
    void targetIdIsInProcess() {
        var target = new InProcessDeploymentTarget(mockBeanRegistry(true));
        assertEquals("in-process", target.targetId());
    }

    @Test
    void deploySucceedsWhenBeanResolvable() {
        var target = new InProcessDeploymentTarget(mockBeanRegistry(true));
        var config = new DeployConfig("qhorus", 1, Map.of(), Map.of());
        var result = target.deploy("casehubio/qhorus:0.5", config);
        assertInstanceOf(DeployResult.Success.class, result);
        assertEquals("qhorus", ((DeployResult.Success) result).instanceId());
    }

    @Test
    void deployFailsWhenBeanNotResolvable() {
        var target = new InProcessDeploymentTarget(mockBeanRegistry(false));
        var config = new DeployConfig("qhorus", 1, Map.of(), Map.of());
        var result = target.deploy("casehubio/qhorus:0.5", config);
        assertInstanceOf(DeployResult.Failed.class, result);
    }

    @Test
    void statusReturnsPresentWhenBeanResolvable() {
        var target = new InProcessDeploymentTarget(mockBeanRegistry(true));
        assertEquals(NodeStatus.PRESENT, target.status("qhorus"));
    }

    @Test
    void statusReturnsAbsentWhenBeanNotResolvable() {
        var target = new InProcessDeploymentTarget(mockBeanRegistry(false));
        assertEquals(NodeStatus.ABSENT, target.status("unknown-app"));
    }

    @Test
    void undeployIsNoOp() {
        var target = new InProcessDeploymentTarget(mockBeanRegistry(true));
        var result = target.undeploy("qhorus");
        assertInstanceOf(DeployResult.Success.class, result);
    }

    private InProcessDeploymentTarget.BeanRegistry mockBeanRegistry(boolean resolvable) {
        return applicationId -> resolvable;
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode -o test -pl deployment -Dtest=InProcessDeploymentTargetTest`
Expected: FAIL — class not found

- [ ] **Step 3: Implement InProcessDeploymentTarget**

```java
package io.casehub.ops.deployment.target;

import io.casehub.desiredstate.api.NodeStatus;
import io.casehub.ops.api.deployment.*;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

@ApplicationScoped
public class InProcessDeploymentTarget implements ApplicationDeploymentTarget {

    private final BeanRegistry beanRegistry;

    @Inject
    public InProcessDeploymentTarget(BeanRegistry beanRegistry) {
        this.beanRegistry = beanRegistry;
    }

    @Override
    public String targetId() { return "in-process"; }

    @Override
    public DeployResult deploy(String imageRef, DeployConfig config) {
        if (!beanRegistry.isResolvable(config.applicationId())) {
            return new DeployResult.Failed(
                "In-process deployment failed: no CDI beans found for applicationId '" 
                + config.applicationId() + "'. Ensure the service JAR is on the classpath.");
        }
        return new DeployResult.Success(config.applicationId());
    }

    @Override
    public DeployResult undeploy(String instanceId) {
        return new DeployResult.Success(instanceId);
    }

    @Override
    public NodeStatus status(String instanceId) {
        return beanRegistry.isResolvable(instanceId) ? NodeStatus.PRESENT : NodeStatus.ABSENT;
    }

    @Override
    public HealthCheckResult healthCheck(String instanceId) {
        return beanRegistry.isResolvable(instanceId)
            ? new HealthCheckResult.Healthy(instanceId)
            : new HealthCheckResult.Unhealthy(instanceId, "CDI beans not resolvable");
    }

    @FunctionalInterface
    public interface BeanRegistry {
        boolean isResolvable(String applicationId);
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode -o test -pl deployment -Dtest=InProcessDeploymentTargetTest`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add deployment/src/main/java/io/casehub/ops/deployment/target/InProcessDeploymentTarget.java deployment/src/test/java/io/casehub/ops/deployment/target/InProcessDeploymentTargetTest.java
git commit -m "feat(#117): InProcessDeploymentTarget — CDI bean verification for local-dev profile"
```

---

### Task 5: PodmanDeploymentTarget

**Files:**
- Create: `deployment/src/main/java/io/casehub/ops/deployment/target/PodmanDeploymentTarget.java`
- Test: `deployment/src/test/java/io/casehub/ops/deployment/target/PodmanDeploymentTargetTest.java`

**Interfaces:**
- Consumes: `ApplicationDeploymentTarget` SPI with `targetId()`
- Consumes: `PodmanClient` from container module
- Produces: `PodmanDeploymentTarget` with `targetId() = "podman"`

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.ops.deployment.target;

import io.casehub.desiredstate.api.NodeStatus;
import io.casehub.ops.api.deployment.*;
import io.casehub.ops.container.podman.PodmanClient;
import io.casehub.ops.container.podman.PodmanContainer;
import io.smallrye.mutiny.Uni;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

class PodmanDeploymentTargetTest {

    @Test
    void targetIdIsPodman() {
        var client = mock(PodmanClient.class);
        var target = new PodmanDeploymentTarget(client);
        assertEquals("podman", target.targetId());
    }

    @Test
    void deployPullsCreatesAndStarts() {
        var client = mock(PodmanClient.class);
        when(client.pullImage(any())).thenReturn(Uni.createFrom().voidItem());
        when(client.createContainer(any())).thenReturn(Uni.createFrom().item("container-123"));
        when(client.startContainer(any())).thenReturn(Uni.createFrom().voidItem());
        var target = new PodmanDeploymentTarget(client);

        var config = new DeployConfig("qhorus", 1, Map.of("quarkus.http.port", "8081"), Map.of());
        var result = target.deploy("casehubio/qhorus:0.5", config);

        assertInstanceOf(DeployResult.Success.class, result);
        assertEquals("container-123", ((DeployResult.Success) result).instanceId());
        verify(client).pullImage("casehubio/qhorus:0.5");
        verify(client).createContainer(any());
        verify(client).startContainer("container-123");
    }

    @Test
    void statusReturnsPresentWhenContainerRunning() {
        var client = mock(PodmanClient.class);
        when(client.listContainers()).thenReturn(Uni.createFrom().item(
            List.of(new PodmanContainer("c1", List.of("qhorus"), "running", "casehubio/qhorus:0.5"))));
        var target = new PodmanDeploymentTarget(client);

        assertEquals(NodeStatus.PRESENT, target.status("qhorus"));
    }

    @Test
    void statusReturnsAbsentWhenNoContainer() {
        var client = mock(PodmanClient.class);
        when(client.listContainers()).thenReturn(Uni.createFrom().item(List.of()));
        var target = new PodmanDeploymentTarget(client);

        assertEquals(NodeStatus.ABSENT, target.status("qhorus"));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode -o test -pl deployment -Dtest=PodmanDeploymentTargetTest`
Expected: FAIL — class not found

- [ ] **Step 3: Implement PodmanDeploymentTarget**

```java
package io.casehub.ops.deployment.target;

import io.casehub.desiredstate.api.NodeStatus;
import io.casehub.ops.api.deployment.*;
import io.casehub.ops.container.podman.PodmanClient;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.util.HashMap;
import java.util.Map;

@ApplicationScoped
public class PodmanDeploymentTarget implements ApplicationDeploymentTarget {

    private final PodmanClient podmanClient;

    @Inject
    public PodmanDeploymentTarget(PodmanClient podmanClient) {
        this.podmanClient = podmanClient;
    }

    @Override
    public String targetId() { return "podman"; }

    @Override
    public DeployResult deploy(String imageRef, DeployConfig config) {
        try {
            podmanClient.pullImage(imageRef).await().indefinitely();
            Map<String, String> env = new HashMap<>(config.config());
            var request = new PodmanClient.CreateContainerRequest(
                    config.applicationId(), imageRef, Map.of(), env);
            var containerId = podmanClient.createContainer(request).await().indefinitely();
            podmanClient.startContainer(containerId).await().indefinitely();
            return new DeployResult.Success(containerId);
        } catch (Exception e) {
            return new DeployResult.Failed("Podman deploy failed: " + e.getMessage());
        }
    }

    @Override
    public DeployResult undeploy(String instanceId) {
        try {
            podmanClient.stopContainer(instanceId).await().indefinitely();
            podmanClient.removeContainer(instanceId).await().indefinitely();
            return new DeployResult.Success(instanceId);
        } catch (Exception e) {
            return new DeployResult.Failed("Podman undeploy failed: " + e.getMessage());
        }
    }

    @Override
    public NodeStatus status(String instanceId) {
        try {
            var containers = podmanClient.listContainers().await().indefinitely();
            for (var c : containers) {
                if (c.names().contains(instanceId) || c.id().equals(instanceId)) {
                    return "running".equals(c.state()) ? NodeStatus.PRESENT : NodeStatus.DRIFTED;
                }
            }
            return NodeStatus.ABSENT;
        } catch (Exception e) {
            return NodeStatus.UNKNOWN;
        }
    }

    @Override
    public HealthCheckResult healthCheck(String instanceId) {
        var nodeStatus = status(instanceId);
        return nodeStatus == NodeStatus.PRESENT
            ? new HealthCheckResult.Healthy(instanceId)
            : new HealthCheckResult.Unhealthy(instanceId, "Container not running");
    }
}
```

Note: `deployment/pom.xml` needs a dependency on `casehub-ops-container` (provided scope) for `PodmanClient`. If the dependency is not appropriate (circular), extract `PodmanDeploymentTarget` into the `container/` module instead.

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode -o test -pl deployment -Dtest=PodmanDeploymentTargetTest`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add deployment/src/main/java/io/casehub/ops/deployment/target/PodmanDeploymentTarget.java deployment/src/test/java/io/casehub/ops/deployment/target/PodmanDeploymentTargetTest.java
git commit -m "feat(#117): PodmanDeploymentTarget — container-based deployment for local-distributed profile"
```

---

## Batch 3: Topology YAML and Bootstrap Wiring

After this batch: topology YAML files exist at `META-INF/desiredstate/`, `BootstrapDomainRegistrar` wires `CrossDomainCompositionEngine` with profile-aware domain registration, and the deployment profile config property is active.

### Task 6: Topology YAML files and configuration

**Files:**
- Create: `app/src/main/resources/META-INF/desiredstate/casehub-topology.yaml`
- Create: `app/src/main/resources/META-INF/desiredstate/modules/casehub-infra-local-distributed.yaml`
- Modify: `app/src/main/resources/application.properties` — add deployment profile config
- Test: `app/src/test/java/io/casehub/ops/service/bootstrap/TopologyYamlCompilationTest.java`

**Interfaces:**
- Produces: topology YAML at standard classpath location
- Produces: `casehub.ops.deployment-profile` config property

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.ops.service.bootstrap;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import org.junit.jupiter.api.Test;
import java.io.InputStream;
import static org.junit.jupiter.api.Assertions.*;

class TopologyYamlCompilationTest {

    @Test
    void topologyYamlExistsOnClasspath() {
        InputStream is = getClass().getClassLoader()
                .getResourceAsStream("META-INF/desiredstate/casehub-topology.yaml");
        assertNotNull(is, "casehub-topology.yaml must exist on classpath");
    }

    @Test
    void topologyYamlParsesAsValidYaml() throws Exception {
        InputStream is = getClass().getClassLoader()
                .getResourceAsStream("META-INF/desiredstate/casehub-topology.yaml");
        var mapper = new ObjectMapper(new YAMLFactory());
        var tree = mapper.readTree(is);
        assertNotNull(tree.get("desiredState"), "Must have desiredState root");
        assertNotNull(tree.get("nodes"), "Must have nodes section");
        assertTrue(tree.get("nodes").has("qhorus"), "Must declare qhorus");
        assertTrue(tree.get("nodes").has("eidos"), "Must declare eidos");
        assertTrue(tree.get("nodes").has("engine"), "Must declare engine");
        assertTrue(tree.get("nodes").has("ops-app"), "Must declare ops-app");
    }

    @Test
    void infraModuleLocalDistributedExistsOnClasspath() {
        InputStream is = getClass().getClassLoader()
                .getResourceAsStream("META-INF/desiredstate/modules/casehub-infra-local-distributed.yaml");
        assertNotNull(is, "casehub-infra-local-distributed.yaml must exist on classpath");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode -o test -pl app -Dtest=TopologyYamlCompilationTest`
Expected: FAIL — YAML files don't exist

- [ ] **Step 3: Create topology YAML files**

`app/src/main/resources/META-INF/desiredstate/casehub-topology.yaml`:
```yaml
desiredState:
  namespace: casehub
  name: casehub-topology

nodes:
  qhorus:
    type: casehub_application
    spec:
      applicationId: qhorus
      image: casehubio/qhorus:0.5-SNAPSHOT
      replicas: 1
      config:
        quarkus.http.port: "8081"

  eidos:
    type: casehub_application
    spec:
      applicationId: eidos
      image: casehubio/eidos:0.3-SNAPSHOT
      replicas: 1

  engine:
    type: casehub_application
    spec:
      applicationId: engine
      image: casehubio/engine:0.4-SNAPSHOT
      replicas: 1

  ops-app:
    type: casehub_application
    dependsOn: [qhorus, eidos, engine]
    spec:
      applicationId: ops-app
      image: casehubio/casehub-ops-app:latest
      replicas: 1
```

`app/src/main/resources/META-INF/desiredstate/modules/casehub-infra-local-distributed.yaml`:
```yaml
module:
  name: casehub-infra-local-distributed
  parameters: {}

nodes:
  ops-postgres:
    type: container:database
    spec:
      name: ops-postgres
      image: "postgres:16"
      databaseName: casehub_ops
      username: casehub
      password: casehub
      dataVolume: ops-postgres-data

  ops-network:
    type: network
    spec:
      name: casehub-net
```

Add to `app/src/main/resources/application.properties`:
```properties
# Deployment profile
casehub.ops.deployment-profile=local-dev
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode -o test -pl app -Dtest=TopologyYamlCompilationTest`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add app/src/main/resources/META-INF/desiredstate/casehub-topology.yaml app/src/main/resources/META-INF/desiredstate/modules/casehub-infra-local-distributed.yaml app/src/main/resources/application.properties app/src/test/java/io/casehub/ops/service/bootstrap/TopologyYamlCompilationTest.java
git commit -m "feat(#117): topology YAML files and deployment profile config property"
```

---

### Task 7: BootstrapDomainRegistrar

**Files:**
- Create: `service/src/main/java/io/casehub/ops/service/bootstrap/BootstrapDomainRegistrar.java`
- Test: `service/src/test/java/io/casehub/ops/service/bootstrap/BootstrapDomainRegistrarTest.java`

**Interfaces:**
- Consumes: `CrossDomainCompositionEngine.registerDomain(DomainRegistration)` from casehub-desiredstate
- Consumes: `casehub.ops.deployment-profile` config property
- Produces: profile-aware domain registration at startup

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.ops.service.bootstrap;

import io.casehub.desiredstate.runtime.composition.CrossDomainCompositionEngine;
import io.casehub.desiredstate.runtime.composition.DomainRegistration;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

class BootstrapDomainRegistrarTest {

    @Test
    void localDevSkipsInfraRegistration() {
        var engine = mock(CrossDomainCompositionEngine.class);
        var registrar = new BootstrapDomainRegistrar(engine, "local-dev");
        registrar.onStartup();

        verify(engine, times(1)).registerDomain(argThat(reg ->
            reg.domainId().value().equals("deployment")));
        verify(engine, never()).registerDomain(argThat(reg ->
            reg.domainId().value().equals("infra")));
    }

    @Test
    void localDistributedRegistersInfraDomain() {
        var engine = mock(CrossDomainCompositionEngine.class);
        var registrar = new BootstrapDomainRegistrar(engine, "local-distributed");
        registrar.onStartup();

        verify(engine, times(2)).registerDomain(any());
        verify(engine).registerDomain(argThat(reg ->
            reg.domainId().value().equals("infra")));
        verify(engine).registerDomain(argThat(reg ->
            reg.domainId().value().equals("deployment")));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode -o test -pl service -Dtest=BootstrapDomainRegistrarTest`
Expected: FAIL — class not found

- [ ] **Step 3: Implement BootstrapDomainRegistrar**

```java
package io.casehub.ops.service.bootstrap;

import io.casehub.desiredstate.api.NodeType;
import io.casehub.desiredstate.runtime.composition.CrossDomainCompositionEngine;
import io.casehub.desiredstate.runtime.composition.DomainId;
import io.casehub.desiredstate.runtime.composition.DomainRegistration;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.eclipse.microprofile.config.inject.ConfigProperty;

import java.util.Set;

@ApplicationScoped
public class BootstrapDomainRegistrar {

    private final CrossDomainCompositionEngine engine;
    private final String deploymentProfile;

    @Inject
    public BootstrapDomainRegistrar(
            CrossDomainCompositionEngine engine,
            @ConfigProperty(name = "casehub.ops.deployment-profile", defaultValue = "local-dev")
            String deploymentProfile) {
        this.engine = engine;
        this.deploymentProfile = deploymentProfile;
    }

    public void onStartup() {
        Set<NodeType> infraProvides = Set.of();

        if (!"local-dev".equals(deploymentProfile)) {
            infraProvides = registerInfraDomain();
        }

        registerDeploymentDomain(infraProvides);
    }

    private Set<NodeType> registerInfraDomain() {
        Set<NodeType> provides = Set.of(
            NodeType.of("container:database"),
            NodeType.of("network"),
            NodeType.of("volume"));
        engine.registerDomain(DomainRegistration.builder(DomainId.of("infra"))
            .provides(provides)
            .build());
        return provides;
    }

    private void registerDeploymentDomain(Set<NodeType> infraRequires) {
        engine.registerDomain(DomainRegistration.builder(DomainId.of("deployment"))
            .provides(Set.of(NodeType.of("casehub_application")))
            .requires(infraRequires)
            .build());
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode -o test -pl service -Dtest=BootstrapDomainRegistrarTest`
Expected: PASS

- [ ] **Step 5: Run full build**

Run: `mvn --batch-mode install`
Expected: PASS — all modules compile and tests pass

- [ ] **Step 6: Commit**

```bash
git add service/src/main/java/io/casehub/ops/service/bootstrap/BootstrapDomainRegistrar.java service/src/test/java/io/casehub/ops/service/bootstrap/BootstrapDomainRegistrarTest.java
git commit -m "feat(#117): BootstrapDomainRegistrar — profile-aware CrossDomainCompositionEngine wiring"
```

---

## Batch 4: Bootstrap Phase Coordinator and Self-Maintenance

After this batch: the ops app has a two-phase bootstrap lifecycle (seed → self-aware), a self-maintenance monitor that activates after dependencies are healthy, and stub KubernetesDeploymentTarget for enterprise profile.

### Task 8: Bootstrap phase coordinator

**Files:**
- Create: `service/src/main/java/io/casehub/ops/service/bootstrap/BootstrapPhaseCoordinator.java`
- Create: `service/src/main/java/io/casehub/ops/service/bootstrap/BootstrapPhase.java`
- Test: `service/src/test/java/io/casehub/ops/service/bootstrap/BootstrapPhaseCoordinatorTest.java`

**Interfaces:**
- Consumes: `BootstrapDomainRegistrar` from Task 7
- Produces: `BootstrapPhaseCoordinator` managing SEED → SELF_AWARE transition
- Produces: `BootstrapPhase` enum: `SEED`, `SELF_AWARE`

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.ops.service.bootstrap;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class BootstrapPhaseCoordinatorTest {

    @Test
    void startsInSeedPhase() {
        var coordinator = new BootstrapPhaseCoordinator();
        assertEquals(BootstrapPhase.SEED, coordinator.currentPhase());
    }

    @Test
    void transitionsToSelfAwareWhenAllDependenciesHealthy() {
        var coordinator = new BootstrapPhaseCoordinator();
        coordinator.markDependencyHealthy("qhorus");
        coordinator.markDependencyHealthy("eidos");
        coordinator.markDependencyHealthy("engine");
        assertEquals(BootstrapPhase.SEED, coordinator.currentPhase());

        coordinator.evaluatePhaseTransition(java.util.Set.of("qhorus", "eidos", "engine"));
        assertEquals(BootstrapPhase.SELF_AWARE, coordinator.currentPhase());
    }

    @Test
    void doesNotTransitionWhenDependenciesMissing() {
        var coordinator = new BootstrapPhaseCoordinator();
        coordinator.markDependencyHealthy("qhorus");
        coordinator.evaluatePhaseTransition(java.util.Set.of("qhorus", "eidos", "engine"));
        assertEquals(BootstrapPhase.SEED, coordinator.currentPhase());
    }

    @Test
    void transitionIsIdempotent() {
        var coordinator = new BootstrapPhaseCoordinator();
        coordinator.markDependencyHealthy("qhorus");
        coordinator.markDependencyHealthy("eidos");
        coordinator.markDependencyHealthy("engine");
        coordinator.evaluatePhaseTransition(java.util.Set.of("qhorus", "eidos", "engine"));
        coordinator.evaluatePhaseTransition(java.util.Set.of("qhorus", "eidos", "engine"));
        assertEquals(BootstrapPhase.SELF_AWARE, coordinator.currentPhase());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode -o test -pl service -Dtest=BootstrapPhaseCoordinatorTest`
Expected: FAIL

- [ ] **Step 3: Implement BootstrapPhase and BootstrapPhaseCoordinator**

`service/src/main/java/io/casehub/ops/service/bootstrap/BootstrapPhase.java`:
```java
package io.casehub.ops.service.bootstrap;

public enum BootstrapPhase {
    SEED,
    SELF_AWARE
}
```

`service/src/main/java/io/casehub/ops/service/bootstrap/BootstrapPhaseCoordinator.java`:
```java
package io.casehub.ops.service.bootstrap;

import jakarta.enterprise.context.ApplicationScoped;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;

@ApplicationScoped
public class BootstrapPhaseCoordinator {

    private volatile BootstrapPhase phase = BootstrapPhase.SEED;
    private final Set<String> healthyDependencies = ConcurrentHashMap.newKeySet();

    public BootstrapPhase currentPhase() {
        return phase;
    }

    public void markDependencyHealthy(String applicationId) {
        healthyDependencies.add(applicationId);
    }

    public void markDependencyUnhealthy(String applicationId) {
        healthyDependencies.remove(applicationId);
    }

    public void evaluatePhaseTransition(Set<String> requiredDependencies) {
        if (phase == BootstrapPhase.SELF_AWARE) return;
        if (healthyDependencies.containsAll(requiredDependencies)) {
            phase = BootstrapPhase.SELF_AWARE;
        }
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode -o test -pl service -Dtest=BootstrapPhaseCoordinatorTest`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add service/src/main/java/io/casehub/ops/service/bootstrap/BootstrapPhase.java service/src/main/java/io/casehub/ops/service/bootstrap/BootstrapPhaseCoordinator.java service/src/test/java/io/casehub/ops/service/bootstrap/BootstrapPhaseCoordinatorTest.java
git commit -m "feat(#117): BootstrapPhaseCoordinator — SEED to SELF_AWARE phase transition"
```

---

### Task 9: SelfMaintenanceMonitor

**Files:**
- Create: `service/src/main/java/io/casehub/ops/service/bootstrap/SelfMaintenanceMonitor.java`
- Test: `service/src/test/java/io/casehub/ops/service/bootstrap/SelfMaintenanceMonitorTest.java`

**Interfaces:**
- Consumes: `BootstrapPhaseCoordinator` from Task 8
- Consumes: `ApplicationDeploymentTarget` for provisioning replacement
- Produces: `SelfMaintenanceMonitor` that activates when phase is SELF_AWARE

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.ops.service.bootstrap;

import io.casehub.desiredstate.api.NodeStatus;
import io.casehub.ops.api.deployment.*;
import org.junit.jupiter.api.Test;
import java.util.Map;
import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

class SelfMaintenanceMonitorTest {

    @Test
    void doesNotActivateInSeedPhase() {
        var coordinator = new BootstrapPhaseCoordinator();
        var target = mock(ApplicationDeploymentTarget.class);
        var monitor = new SelfMaintenanceMonitor(coordinator, target, true);

        assertFalse(monitor.isActive());
        monitor.check("ops-app", "casehubio/casehub-ops-app:2.0");
        verify(target, never()).deploy(any(), any());
    }

    @Test
    void activatesInSelfAwarePhase() {
        var coordinator = new BootstrapPhaseCoordinator();
        coordinator.markDependencyHealthy("qhorus");
        coordinator.markDependencyHealthy("eidos");
        coordinator.markDependencyHealthy("engine");
        coordinator.evaluatePhaseTransition(java.util.Set.of("qhorus", "eidos", "engine"));
        var target = mock(ApplicationDeploymentTarget.class);
        var monitor = new SelfMaintenanceMonitor(coordinator, target, true);

        assertTrue(monitor.isActive());
    }

    @Test
    void disabledWhenConfigFalse() {
        var coordinator = new BootstrapPhaseCoordinator();
        coordinator.markDependencyHealthy("qhorus");
        coordinator.markDependencyHealthy("eidos");
        coordinator.markDependencyHealthy("engine");
        coordinator.evaluatePhaseTransition(java.util.Set.of("qhorus", "eidos", "engine"));
        var target = mock(ApplicationDeploymentTarget.class);
        var monitor = new SelfMaintenanceMonitor(coordinator, target, false);

        assertFalse(monitor.isActive());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode -o test -pl service -Dtest=SelfMaintenanceMonitorTest`
Expected: FAIL

- [ ] **Step 3: Implement SelfMaintenanceMonitor**

```java
package io.casehub.ops.service.bootstrap;

import io.casehub.ops.api.deployment.ApplicationDeploymentTarget;
import io.casehub.ops.api.deployment.DeployConfig;
import io.casehub.ops.api.deployment.DeployResult;
import io.casehub.ops.api.deployment.HealthCheckResult;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.eclipse.microprofile.config.inject.ConfigProperty;
import org.jboss.logging.Logger;

import java.util.Map;

@ApplicationScoped
public class SelfMaintenanceMonitor {

    private static final Logger LOG = Logger.getLogger(SelfMaintenanceMonitor.class);

    private final BootstrapPhaseCoordinator coordinator;
    private final ApplicationDeploymentTarget deployTarget;
    private final boolean enabled;

    @Inject
    public SelfMaintenanceMonitor(
            BootstrapPhaseCoordinator coordinator,
            ApplicationDeploymentTarget deployTarget,
            @ConfigProperty(name = "casehub.ops.self-maintenance.enabled", defaultValue = "true")
            boolean enabled) {
        this.coordinator = coordinator;
        this.deployTarget = deployTarget;
        this.enabled = enabled;
    }

    public boolean isActive() {
        return enabled && coordinator.currentPhase() == BootstrapPhase.SELF_AWARE;
    }

    public void check(String applicationId, String newImageRef) {
        if (!isActive()) return;

        LOG.infof("Self-maintenance: upgrading %s to %s", applicationId, newImageRef);
        var config = new DeployConfig(applicationId + "-next", 1, Map.of(), Map.of());
        var result = deployTarget.deploy(newImageRef, config);

        if (result instanceof DeployResult.Failed f) {
            LOG.errorf("Self-maintenance: deploy failed for %s — %s", applicationId, f.reason());
            return;
        }

        var instanceId = ((DeployResult.Success) result).instanceId();
        var health = deployTarget.healthCheck(instanceId);

        if (health instanceof HealthCheckResult.Unhealthy u) {
            LOG.errorf("Self-maintenance: health check failed for %s — %s. Rolling back.", applicationId, u.reason());
            deployTarget.undeploy(instanceId);
            return;
        }

        LOG.infof("Self-maintenance: replacement healthy. Old instance should deactivate gracefully.");
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode -o test -pl service -Dtest=SelfMaintenanceMonitorTest`
Expected: PASS

- [ ] **Step 5: Add self-maintenance config to application.properties**

Append to `app/src/main/resources/application.properties`:
```properties
# Self-maintenance
casehub.ops.self-maintenance.enabled=true
casehub.ops.self-maintenance.health-check-timeout=30s
```

- [ ] **Step 6: Run full build**

Run: `mvn --batch-mode install`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add service/src/main/java/io/casehub/ops/service/bootstrap/SelfMaintenanceMonitor.java service/src/test/java/io/casehub/ops/service/bootstrap/SelfMaintenanceMonitorTest.java app/src/main/resources/application.properties
git commit -m "feat(#117): SelfMaintenanceMonitor — activates in SELF_AWARE phase for ops app upgrades"
```

---

### Task 10: KubernetesDeploymentTarget (stub)

**Files:**
- Create: `app/src/main/java/io/casehub/ops/service/target/KubernetesDeploymentTarget.java`
- Test: `app/src/test/java/io/casehub/ops/service/target/KubernetesDeploymentTargetTest.java`

**Interfaces:**
- Consumes: `ApplicationDeploymentTarget` SPI with `targetId()`
- Produces: `KubernetesDeploymentTarget` with `targetId() = "kubernetes"`

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.ops.service.target;

import io.casehub.ops.api.deployment.*;
import org.junit.jupiter.api.Test;
import java.util.Map;
import static org.junit.jupiter.api.Assertions.*;

class KubernetesDeploymentTargetTest {

    @Test
    void targetIdIsKubernetes() {
        var target = new KubernetesDeploymentTarget();
        assertEquals("kubernetes", target.targetId());
    }

    @Test
    void deployReturnsNotImplementedUntilFabric8Wired() {
        var target = new KubernetesDeploymentTarget();
        var config = new DeployConfig("qhorus", 1, Map.of(), Map.of());
        var result = target.deploy("casehubio/qhorus:0.5", config);
        assertInstanceOf(DeployResult.Failed.class, result);
        assertTrue(((DeployResult.Failed) result).reason().contains("not yet implemented"));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode -o test -pl app -Dtest=KubernetesDeploymentTargetTest`
Expected: FAIL

- [ ] **Step 3: Implement KubernetesDeploymentTarget stub**

```java
package io.casehub.ops.service.target;

import io.casehub.desiredstate.api.NodeStatus;
import io.casehub.ops.api.deployment.*;
import jakarta.enterprise.context.ApplicationScoped;

@ApplicationScoped
public class KubernetesDeploymentTarget implements ApplicationDeploymentTarget {

    @Override
    public String targetId() { return "kubernetes"; }

    @Override
    public DeployResult deploy(String imageRef, DeployConfig config) {
        return new DeployResult.Failed("Kubernetes deployment target not yet implemented — requires fabric8 client wiring (#126)");
    }

    @Override
    public DeployResult undeploy(String instanceId) {
        return new DeployResult.Failed("Kubernetes deployment target not yet implemented");
    }

    @Override
    public NodeStatus status(String instanceId) {
        return NodeStatus.UNKNOWN;
    }

    @Override
    public HealthCheckResult healthCheck(String instanceId) {
        return new HealthCheckResult.Unknown(instanceId);
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode -o test -pl app -Dtest=KubernetesDeploymentTargetTest`
Expected: PASS

- [ ] **Step 5: Run full build and all tests**

Run: `mvn --batch-mode install`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add app/src/main/java/io/casehub/ops/service/target/KubernetesDeploymentTarget.java app/src/test/java/io/casehub/ops/service/target/KubernetesDeploymentTargetTest.java
git commit -m "feat(#117): KubernetesDeploymentTarget stub — enterprise profile placeholder (#126)"
```

---

## References

- [2026-10-14-deployment-profiles-bootstrap-design.md] — design spec this plan implements
- `api/src/main/java/io/casehub/ops/api/deployment/ApplicationDeploymentTarget.java` — deployment target SPI
- `api/src/main/java/io/casehub/ops/api/deployment/CaseHubApplicationSpec.java` — platform service node type
- `api/src/main/java/io/casehub/ops/api/deployment/NoOpDeploymentTarget.java` — existing @DefaultBean
- `deployment/src/main/java/io/casehub/ops/deployment/DeploymentNodeProvisioner.java:70-86` — provision dispatch
- `deployment/src/main/java/io/casehub/ops/deployment/handler/CaseHubAppProvisionHandler.java` — build→deploy→health orchestration
- `deployment/src/main/java/io/casehub/ops/deployment/drift/CaseHubAppDriftChecker.java` — casehub_application drift detection
- `infra/src/main/java/io/casehub/ops/infra/InfraWrappingFactory.java` — backendId extraction pattern (model for deploymentTarget extraction)
- `infra/src/main/java/io/casehub/ops/infra/InfraNodeSpecFactoryProvider.java` — existing NodeSpecFactoryProvider pattern
- `container/src/main/java/io/casehub/ops/container/ContainerNodeSpecFactoryProvider.java` — existing NodeSpecFactoryProvider
- `container/src/main/java/io/casehub/ops/container/podman/PodmanClient.java` — Podman REST API client
- `app/src/main/resources/application.properties` — existing config
- GitHub #117 — parent issue
- GitHub #123 — self-maintenance detailed protocol (deferred)
- GitHub #126 — CI/CD pipeline integration for enterprise (deferred)
