# Multi-Environment Topology Deployment

**Issue:** casehubio/casehub-ops#150 (epic)
**Children:** #151, #152, #153
**Branch:** issue-150-multi-env-deployment
**Scale:** XL
**Complexity:** High
**Prerequisites:** #117 (deployment profiles), #83 (ApplicationDeploymentTarget SPI)

## Problem

The topology test suite deploys `casehub_application` nodes exclusively via the claudony fleet API. There is no mechanism to run the same topology YAML against different target environments (Kubernetes, Podman) without changing the YAML or the test code. The topology YAML should remain environment-agnostic; environment selection should be a runtime concern.

## Solution Overview

Introduce a `TargetEnvironment` abstraction in the `testing/` module that resolves to the correct `ApplicationDeploymentTarget` and cluster lifecycle based on a system property set by Maven profile. The abstraction reuses the `ApplicationDeploymentTarget` SPI from #117 — same interface, test-instantiated instead of CDI-discovered.

Three target environments:

| Environment | Maven Profile | System Property | Target | Cluster Lifecycle |
|-------------|--------------|-----------------|--------|-------------------|
| LOCAL | `-Pintegration-live` (default) | `casehub.topology.target-environment=local` | `ClaudonyDeploymentTarget` | `ServiceClusterExtension` (existing) |
| K8S | `-Ptarget-k8s` | `casehub.topology.target-environment=k8s` | `KubernetesDeploymentTarget` | `K3dClusterExtension` (new) |
| PODMAN | `-Ptarget-podman` | `casehub.topology.target-environment=podman` | `PodmanDeploymentTarget` | `PodmanEnvironmentExtension` (new) |

## Architecture

### TargetEnvironment Enum

```java
public enum TargetEnvironment {
    LOCAL, K8S, PODMAN;

    public static TargetEnvironment current() {
        String value = System.getProperty(
            "casehub.topology.target-environment", "local");
        return TargetEnvironment.valueOf(value.toUpperCase().replace("-", "_"));
    }
}
```

### TargetEnvironmentFactory

Creates the appropriate `ApplicationDeploymentTarget` for the current environment. Each target is constructed with its required dependencies (fleet client, K8s client, Podman client).

```java
public class TargetEnvironmentFactory {

    public static ApplicationDeploymentTarget createTarget(
            TargetEnvironment env, Map<String, String> endpoints) {
        return switch (env) {
            case LOCAL -> createClaudonyTarget(endpoints.get("claudony"));
            case K8S -> createK8sTarget(endpoints);
            case PODMAN -> createPodmanTarget(endpoints);
        };
    }
}
```

### NodeProvisioner Wiring

A single `EnvironmentAwareNodeProvisioner` in `testing/` handles `casehub_application` nodes by delegating to the environment's `ApplicationDeploymentTarget`:

```java
class EnvironmentAwareNodeProvisioner implements NodeProvisioner {

    private final ApplicationDeploymentTarget target;

    @Override
    public Set<NodeType> handledTypes() {
        return Set.of(NodeType.of("casehub_application"));
    }

    @Override
    public ProvisionResult provision(DesiredNode node, ProvisionContext context) {
        CaseHubApplicationSpec appSpec = extractAppSpec(node);
        DeployResult result = target.deploy(
            appSpec.image(),
            new DeployConfig(appSpec.applicationId(), appSpec.replicas(),
                appSpec.config(), Map.of()));
        return result instanceof DeployResult.Success
            ? new ProvisionResult.Success()
            : new ProvisionResult.Failed(((DeployResult.Failed) result).reason());
    }
}
```

This replaces the current `ClaudonyNodeProvisioner` in topology-tests. The existing class is a special case of this pattern (LOCAL environment only).

### Cluster Lifecycle Extensions

Each environment has a JUnit 5 extension managing infrastructure lifecycle:

**K3dClusterExtension (new):**
- `beforeAll`: Create k3d cluster, load application images, create namespace
- `afterAll`: Delete k3d cluster
- Image strategy: resolve `quarkus-run.jar` from Maven build (same as `ServiceProcess.resolveJar()`), build a minimal container image using k3d's built-in `k3d image import`
- Exports: `KubernetesClient` connected to the k3d cluster, endpoint map

**PodmanEnvironmentExtension (new):**
- `beforeAll`: Verify Podman socket accessible, create pod/network for test isolation
- `afterAll`: Clean up pods/networks/containers
- Uses existing `PodmanClient` from `container/` module
- Exports: `PodmanClient` instance, endpoint map

**ServiceClusterExtension (existing, unchanged):**
- Continues to manage LOCAL environment exactly as today
- Launches real Quarkus service JARs, PostgreSQL via Testcontainers

### Maven Profile Configuration

In `topology-tests/pom.xml`:

```xml
<profiles>
    <profile>
        <id>integration-live</id>
        <!-- existing profile, unchanged -->
        <properties>
            <casehub.topology.target-environment>local</casehub.topology.target-environment>
        </properties>
    </profile>
    <profile>
        <id>target-k8s</id>
        <properties>
            <casehub.topology.target-environment>k8s</casehub.topology.target-environment>
        </properties>
        <dependencies>
            <dependency>
                <groupId>io.fabric8</groupId>
                <artifactId>kubernetes-client</artifactId>
                <scope>test</scope>
            </dependency>
        </dependencies>
    </profile>
    <profile>
        <id>target-podman</id>
        <properties>
            <casehub.topology.target-environment>podman</casehub.topology.target-environment>
        </properties>
    </profile>
</profiles>
```

## Issue Decomposition

### #151: Environment Abstraction Layer (M, Med)

The foundation — introduces the abstraction without adding new environments:

1. `TargetEnvironment` enum in `testing/`
2. `TargetEnvironmentFactory` resolving environment → `ApplicationDeploymentTarget`
3. `EnvironmentAwareNodeProvisioner` in `testing/` — delegates to target
4. Maven profile `integration-live` sets `casehub.topology.target-environment=local`
5. Existing levels 01-04 pass on LOCAL with the new abstraction (regression — no behavior change)

The key constraint: existing tests must continue to work exactly as before. The `TargetEnvironment.current()` defaults to `LOCAL`. Tests that don't use the factory continue to work unchanged.

### #152: Kubernetes Provisioner (L, High)

Adds K8s as a target environment:

1. `KubernetesDeploymentTarget` implementing `ApplicationDeploymentTarget` — creates K8s Deployment + Service per application using fabric8 client
2. `K3dClusterExtension` — JUnit extension managing k3d cluster lifecycle (create/teardown)
3. Image loading strategy — resolve `quarkus-run.jar`, build minimal container, import into k3d
4. Maven profile `-Ptarget-k8s` with fabric8 dependency
5. At least levels 01 and 04 passing on K8s target

The `KubernetesDeploymentTarget` wraps the existing `Fabric8NodeProvisioner` patterns (already proven in `topology-tests/live/`). The key new piece is the `K3dClusterExtension` that automates cluster creation.

### #153: Podman Provisioner (M, Med)

Adds Podman as a target environment:

1. `PodmanDeploymentTarget` already exists in `container/` module with `targetId() = "podman"` — reuse it
2. `PodmanEnvironmentExtension` — JUnit extension managing Podman pod lifecycle
3. Maven profile `-Ptarget-podman`
4. At least levels 01 and 04 passing on Podman target

The `container/` module already provides the full Podman REST API client and a working `PodmanDeploymentTarget`. The primary work is the test lifecycle extension and wiring.

## Topology YAML — No Changes

The topology YAML stays exactly as-is. Environment selection is purely a runtime/config concern:

```yaml
# This YAML is identical across all environments
nodes:
  my-llm:
    type: casehub_application
    spec:
      applicationId: my-llm
      image: casehubio/my-llm-app:latest
      deploymentTarget: claudony  # ignored when env overrides target
      replicas: 1
```

The `deploymentTarget: claudony` field in the spec is used by the LOCAL environment. When K8S or PODMAN is selected, the `TargetEnvironmentFactory` provides a different `ApplicationDeploymentTarget` that overrides the per-node target. The precedence is: environment target > per-node `deploymentTarget`.

## Module Structure Changes

| Module | Changes |
|--------|---------|
| `testing/` | + `TargetEnvironment` enum, `TargetEnvironmentFactory`, `EnvironmentAwareNodeProvisioner`, `K3dClusterExtension`, `PodmanEnvironmentExtension` |
| `topology-tests/` | + Maven profiles (`target-k8s`, `target-podman`), + K8s/Podman integration tests, existing tests unchanged |
| `container/` | No changes — `PodmanDeploymentTarget` already exists |
| All other modules | No changes |

## Testing Strategy

1. **#151 regression:** All existing `@Tag("integration-live")` tests pass with the new abstraction on LOCAL. Zero behavior change.
2. **#152 K8s smoke:** Levels 01 and 04 deploy successfully to k3d cluster. Health checks pass. Deprovisioning cleans up K8s resources.
3. **#153 Podman smoke:** Levels 01 and 04 deploy successfully to Podman containers. Health checks pass. Container cleanup on teardown.
4. **Environment isolation:** K8s and Podman tests run in isolated namespaces/pods. No cross-test interference.

## Scope Boundaries

**In scope:**
- Environment abstraction layer (enum, factory, provisioner) in testing/
- K3dClusterExtension for K8s cluster lifecycle
- PodmanEnvironmentExtension for Podman lifecycle
- KubernetesDeploymentTarget (test-level, wrapping fabric8)
- Maven profiles for environment selection
- Levels 01 and 04 verified on all three environments

**Out of scope:**
- Production CDI wiring for deployment profiles (that's #117)
- Full level suite (01-14) on K8s/Podman (future work after foundation proves out)
- CI pipeline integration for K8s/Podman test environments
- Container registry / image publishing
- Network policy / security configuration for K8s

## References

- casehubio/casehub-ops#117 — deployment profiles and bootstrap model design spec
- `docs/specs/issue-117-deployment-profiles-bootstrap/2026-10-14-deployment-profiles-bootstrap-design.md` — full #117 spec
- `api/src/main/java/io/casehub/ops/api/deployment/ApplicationDeploymentTarget.java` — target SPI
- `container/src/main/java/io/casehub/ops/container/PodmanDeploymentTarget.java` — existing Podman target
- `topology-tests/src/test/java/io/casehub/ops/topology/live/Fabric8NodeProvisioner.java` — existing K8s provisioner patterns
- `topology-tests/src/test/java/io/casehub/ops/topology/live/KubernetesAvailable.java` — existing K8s availability condition
- `topology-tests/src/test/java/io/casehub/ops/topology/integration/ServiceClusterExtension.java` — existing LOCAL lifecycle
- `topology-tests/src/test/java/io/casehub/ops/topology/integration/ServiceProcess.java` — JAR resolution pattern
- `topology-tests/src/test/java/io/casehub/ops/topology/integration/ClaudonyNodeProvisioner.java` — current provisioner (replaced by EnvironmentAwareNodeProvisioner)
- ARC42STORIES.MD §4 — SPI quad pattern, CrossDomainCompositionEngine
- ARC42STORIES.MD §8 — CaseHub Application Node Type, ApplicationDeploymentTarget SPI
