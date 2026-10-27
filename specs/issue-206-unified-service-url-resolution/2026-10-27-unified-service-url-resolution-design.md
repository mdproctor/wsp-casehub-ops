# Unified Service URL Resolution and Runtime Assertions

**Issue:** #206
**Date:** 2026-10-27
**Status:** Revised (post-review)

## Summary

Unify the test assertion model across LOCAL, K8S, and PODMAN so all three environments follow the same verification flow: deploy → resolve service URL → wait for health → exercise functionally → assert behavior. Remove environment-specific assertion branches from `HelloLlmMultiEnvTest`.

## Motivation

Issue #205 established container image deployment to K8S and PODMAN. But only LOCAL runs functional assertions — K8S and PODMAN verify deployment exists without checking that services actually respond. The `HelloLlmMultiEnvTest` has environment-specific `if (env == LOCAL)` branches that should not exist.

## Scope

### In scope

- `serviceUrl(String instanceId)` default method on `ApplicationDeploymentTarget`
- K8S implementation via fabric8 `KubernetesClient.portForward()` API
- Podman implementation via container port mapping + inspect
- LOCAL implementation via URL tracking in `ClaudonyDeploymentTarget`
- `ServiceHealthWaiter` utility extracted from `ServiceProcess.waitForHealth()`
- `healthPath` field on `DeployConfig` (optional, defaults to `/q/health/ready`)
- `HelloLlmMultiEnvTest` rewritten with unified assertion flow
- `FleetLlmExerciser` retained as LOCAL-only additive assertion (in-JVM bonus — explicitly descoped from issue #206's deliverable list; see D1 rationale)
- `ApplicationDeploymentTarget` extends `AutoCloseable` for port-forward cleanup

### Out of scope

- Real images for eidos/engine/manifest (prerequisite: app modules in those repos)
- CI orchestration for cross-repo image builds
- Production service mesh or ingress configuration
- HTTP-based functional exerciser for K8S/PODMAN (separate future issue — replaces FleetLlmExerciser's in-JVM approach with HTTP)

## Architecture

### Component 1: `serviceUrl()` on `ApplicationDeploymentTarget`

Add a default method and extend `AutoCloseable`:

```java
public interface ApplicationDeploymentTarget extends AutoCloseable {

    default Optional<String> serviceUrl(String instanceId) {
        return Optional.empty();
    }

    @Override
    default void close() {}

    // ... existing methods unchanged
}
```

Each implementation overrides `serviceUrl()`:

**`ClaudonyDeploymentTarget`** (LOCAL): Claudony's fleet API registers apps as metadata — it does not proxy HTTP traffic to services. In the test, `ServiceClusterExtension` starts actual service processes via `ServiceProcess`, and their URLs come from `ServiceProcess.baseUrl()`. The test-created `ClaudonyDeploymentTarget` tracks a `Map<String, String> serviceUrls` map. After `ServiceClusterExtension` starts processes, it injects URLs into the target. `serviceUrl(instanceId)` reads from this map.

Implementation: add a `registerServiceUrl(String instanceId, String url)` method to `ClaudonyDeploymentTarget`. `ServiceClusterExtension` calls it after starting each `ServiceProcess`. For apps not managed by `ServiceClusterExtension` (registered via the topology planner), the URL is not available — `serviceUrl()` returns `Optional.empty()`.

**`KubernetesDeploymentTarget`** (K8S): Uses fabric8's `KubernetesClient.portForward()` API to tunnel through the K8s API server (already exposed from k3s to the host by Testcontainers):

```java
LocalPortForward pf = client.services()
    .inNamespace(namespace)
    .withName(instanceId)
    .portForward(8080);
int localPort = pf.getLocalPort();
// URL: http://localhost:<localPort>
```

This avoids shell access to the k3s container entirely. `LocalPortForward` implements `Closeable` — the target tracks active forwards in a `Map<String, LocalPortForward>` and closes them all in `close()`.

**`PodmanDeploymentTarget`** (PODMAN): Update `deploy()` to pass port mappings — map container port 8080 to host port 0 (dynamic assignment). Add `containerInspect(String id)` to `PodmanClient` that calls `GET /containers/{id}/json` and extracts the assigned host port from `HostConfig.PortBindings`. `serviceUrl()` calls inspect and returns `http://localhost:<hostPort>`.

### Component 2: `ServiceHealthWaiter`

New utility in `testing/src/main/java/io/casehub/ops/testing/env/ServiceHealthWaiter.java`.

Extracted from `ServiceProcess.waitForHealth()` (lines 100-128):

```java
public final class ServiceHealthWaiter {

    public static void waitForReady(String serviceUrl, String healthPath,
                                     Duration timeout) {
        String url = serviceUrl + healthPath;
        long deadline = System.currentTimeMillis() + timeout.toMillis();
        while (System.currentTimeMillis() < deadline) {
            try {
                HttpURLConnection conn = (HttpURLConnection)
                        URI.create(url).toURL().openConnection();
                conn.setConnectTimeout(500);
                conn.setReadTimeout(500);
                try {
                    if (conn.getResponseCode() == 200) return;
                } finally {
                    conn.disconnect();
                }
            } catch (IOException ignored) {}
            try { Thread.sleep(500); } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                throw new RuntimeException("Interrupted waiting for " + url);
            }
        }
        throw new IllegalStateException(
                serviceUrl + " did not become healthy within " + timeout
                + " (checked " + url + ")");
    }
}
```

`ServiceProcess.waitForHealth()` should be refactored to delegate to `ServiceHealthWaiter`, keeping the process-alive check as the only difference.

### Component 3: `healthPath` on `DeployConfig`

Add an optional `healthPath` field to `DeployConfig`:

```java
public record DeployConfig(
    String applicationId,
    int replicas,
    Map<String, Object> config,
    Map<String, String> env,
    String healthPath       // new — nullable, defaults to /q/health/ready
) {
    public DeployConfig {
        config = config != null ? Map.copyOf(config) : Map.of();
        env = env != null ? Map.copyOf(env) : Map.of();
        if (healthPath == null || healthPath.isBlank()) healthPath = "/q/health/ready";
    }

    // Backwards-compatible constructor for existing callers
    public DeployConfig(String applicationId, int replicas,
                        Map<String, Object> config, Map<String, String> env) {
        this(applicationId, replicas, config, env, null);
    }
}
```

Keeping it on `DeployConfig` rather than `CaseHubApplicationSpec` avoids putting test metadata on a production domain type. `DeployConfig` is already the deployment-time configuration bag — health path is a deployment concern.

### Component 4: `HelloLlmMultiEnvTest` rewrite

The unified flow:

```java
@Test
void deploysTopologyViaTransitionPlanner() {
    TargetEnvironment env = TargetEnvironment.current();
    DesiredStateGraph graph = compileGraph(...);
    ApplicationDeploymentTarget target = createTargetForEnv(env);

    try {
        // Deploy all nodes (via transition planner — existing)
        TransitionResult result = executeTransitions(graph, target);

        // Unified assertion — same for all environments
        for (DesiredNode node : graph.nodes().values()) {
            CaseHubApplicationSpec spec = extractAppSpec(node);
            Optional<String> url = target.serviceUrl(spec.applicationId());
            assertThat(url).as("Service URL for %s", spec.applicationId()).isPresent();

            ServiceHealthWaiter.waitForReady(url.get(), spec.healthPath(),
                    Duration.ofSeconds(60));

            assertHttpOk(url.get() + spec.healthPath());
        }

        // LOCAL-only bonus: in-JVM LLM exercise
        if (env == TargetEnvironment.LOCAL) {
            FleetLlmExerciser.exercise("my-llm", ...);
        }
    } finally {
        undeploy(graph, target);
        target.close();
    }
}
```

No environment branching in deploy/health/assert. The only `if` is the additive `FleetLlmExerciser` on LOCAL. The `finally` block ensures port-forward cleanup even on assertion failure.

### Component 5: Podman port mapping

Update `PodmanDeploymentTarget.deploy()`:

```java
// Before: Map.of() — no ports exposed
var request = new CreateContainerRequest(name, image, Map.of(), env);

// After: expose port 8080 with dynamic host assignment
var request = new CreateContainerRequest(name, image, Map.of(8080, 0), env);
```

Add `containerInspect(String id)` to `PodmanClient` — calls `GET /containers/{id}/json` and extracts the assigned host port from the port bindings response.

`PodmanDeploymentTarget.serviceUrl()` calls inspect and returns `http://localhost:<hostPort>`.

## Test strategy

### Unit tests

- `ServiceHealthWaiterTest`: mock HTTP server, verify waits for 200, throws on timeout
- `KubernetesDeploymentTargetTest`: verify `serviceUrl()` returns port-forwarded URL (mock fabric8 client)
- `PodmanDeploymentTargetTest`: verify `serviceUrl()` returns inspected port URL
- `DeployConfigTest`: verify `healthPath` defaults to `/q/health/ready`

### Integration tests (integration-live)

- `HelloLlmMultiEnvTest` on all three environments: deploy → serviceUrl → health wait → assert
- `PodmanDeploymentTargetLiveTest`: verify port mapping and service URL resolution

## Trade-offs

- **AutoCloseable on ApplicationDeploymentTarget:** Widens the interface contract. Justified because K8S port-forwards are stateful resources that need cleanup. Default `close()` is no-op — non-stateful targets are unaffected.
- **healthPath on DeployConfig, not CaseHubApplicationSpec:** Keeps the domain type clean but means the topology YAML can't declare health paths directly — the test harness must map from spec to deploy config. Accepted because health path is a deployment/test concern, not a domain concern.
- **LOCAL URL injection via registerServiceUrl():** Widens `ClaudonyDeploymentTarget` with a test-support method on a production class. Mitigated by making the method package-private or protected. The production CDI usage never calls it.
- **FleetLlmExerciser remains LOCAL-only:** Descoped from issue #206's deliverable. It tests in-JVM classpath routing, which is fundamentally different from HTTP service availability. An HTTP-based exerciser for K8S/PODMAN is a separate concern.
- **Port-forward complexity in K8S:** fabric8's `LocalPortForward` is well-tested API. Cleanup via `AutoCloseable.close()` is reliable. No shell exec needed.

## References

- `api/.../ApplicationDeploymentTarget.java:5-24` — interface to extend with serviceUrl() and AutoCloseable
- `api/.../DeployConfig.java:1-17` — record to extend with healthPath
- `api/.../HealthCheckResult.java:1-7` — existing health check contract
- `claudony/.../ClaudonyDeploymentTarget.java:18-128` — LOCAL target, fleet API metadata only
- `container/.../PodmanDeploymentTarget.java:14-75` — Podman target, needs port mapping
- `container/.../PodmanClient.java:105-124,185-189` — CreateContainerRequest with portMappings
- `topology-tests/.../KubernetesDeploymentTarget.java:20-117` — K8S target, needs fabric8 portForward
- `topology-tests/.../HelloLlmMultiEnvTest.java` — test to rewrite
- `topology-tests/.../ServiceProcess.java:100-128` — waitForHealth() to extract
- `topology-tests/.../ServiceClusterExtension.java:27-60` — manages ServiceProcess lifecycle, injects URLs
- `topology-tests/.../FleetLlmExerciser.java` — in-JVM LLM exercise (LOCAL-only)
- Design review `issue-206-service-url-20261027-005608` — R1-02 (proxy doesn't exist), R1-03 (fabric8 portForward), R1-08 (healthPath location)
