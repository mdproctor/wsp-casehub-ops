# Unified Service URL Resolution and Runtime Assertions

**Issue:** #206
**Date:** 2026-10-27
**Status:** Draft

## Summary

Unify the test assertion model across LOCAL, K8S, and PODMAN so all three environments follow the same verification flow: deploy → resolve service URL → wait for health → exercise functionally → assert behavior. Remove environment-specific assertion branches from `HelloLlmMultiEnvTest`.

## Motivation

Issue #205 established container image deployment to K8S and PODMAN. But only LOCAL runs functional assertions — K8S and PODMAN verify deployment exists without checking that services actually respond. The `HelloLlmMultiEnvTest` has environment-specific `if (env == LOCAL)` branches that should not exist.

## Scope

### In scope

- `serviceUrl(String instanceId)` default method on `ApplicationDeploymentTarget`
- K8S implementation via `kubectl port-forward` in k3s testcontainer
- Podman implementation via container port mapping + inspect
- `ServiceHealthWaiter` utility extracted from `ServiceProcess.waitForHealth()`
- `healthPath` field on `CaseHubApplicationSpec` (optional, defaults to `/q/health/ready`)
- `HelloLlmMultiEnvTest` rewritten with unified assertion flow
- `FleetLlmExerciser` retained as LOCAL-only additive assertion (in-JVM bonus)

### Out of scope

- Real images for eidos/engine/manifest (prerequisite: app modules in those repos)
- CI orchestration for cross-repo image builds
- Production service mesh or ingress configuration

## Architecture

### Component 1: `serviceUrl()` on `ApplicationDeploymentTarget`

Add a default method to the existing interface:

```java
default Optional<String> serviceUrl(String instanceId) {
    return Optional.empty();
}
```

Each implementation overrides:

**`ClaudonyDeploymentTarget`** (LOCAL): The claudony fleet URL is already known from the constructor — it's the same URL used for fleet API calls. Apps deployed via claudony are accessible through claudony's proxy at `<fleetUrl>/api/apps/<appId>/proxy/`.

**`KubernetesDeploymentTarget`** (K8S): After deploying, runs `kubectl port-forward svc/<instanceId> <localPort>:8080` inside the k3s container. The port-forward process binds to `0.0.0.0` so the host can reach it through the k3s container's mapped port. Returns `http://localhost:<hostPort>`.

Implementation detail: `KubernetesDeploymentTarget` gains a `Map<String, Integer> portForwards` field tracking active port-forward mappings per instanceId. The `serviceUrl()` call starts the port-forward if not already running. `undeploy()` cleans up the port-forward.

Port allocation: Use `ServerSocket(0)` on the host side to find a free port. Then exec into k3s: `kubectl port-forward --address 0.0.0.0 svc/<name> <port>:8080 -n casehub-test &`. Since the k3s container maps ports to the host, the test JVM accesses `http://<k3sContainerHost>:<port>`.

**`PodmanDeploymentTarget`** (PODMAN): Update `deploy()` to pass port mappings — map container port 8080 to host port 0 (dynamic assignment). After container starts, `serviceUrl()` inspects the container via `podman container inspect` to get the assigned host port. Returns `http://localhost:<hostPort>`.

Implementation: `PodmanDeploymentTarget.deploy()` currently creates containers with `Map.of()` for port mappings. Change to `Map.of(8080, 0)` — Podman assigns a random host port. Add a `containerInspect(String id)` method to `PodmanClient` that returns port mapping details. `serviceUrl()` calls inspect and extracts the host port.

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

`ServiceProcess.waitForHealth()` should be refactored to delegate to this utility, keeping the process-alive check as the only difference.

### Component 3: `healthPath` on `CaseHubApplicationSpec`

Add an optional `healthPath` field to `CaseHubApplicationSpec`:

```java
public record CaseHubApplicationSpec(
    String applicationId,
    String source,
    String image,
    RuntimeConfig runtime,
    Map<String, Object> config,
    List<String> dependencies,
    int replicas,
    String healthPath       // new — nullable, defaults to /q/health/ready
) implements DeploymentNodeSpec { ... }
```

The compact constructor defaults `healthPath` to `/q/health/ready` when null or blank.

Topology YAML example for a placeholder:
```yaml
manifest:
  type: casehub_application
  spec:
    applicationId: manifest
    image: nginx:alpine
    healthPath: /
```

### Component 4: `HelloLlmMultiEnvTest` rewrite

The unified flow:

```java
@Test
void deploysTopologyViaTransitionPlanner() {
    TargetEnvironment env = TargetEnvironment.current();
    DesiredStateGraph graph = compileGraph(...);
    ApplicationDeploymentTarget target = createTargetForEnv(env);

    // Deploy all nodes (via transition planner — existing)
    TransitionResult result = executeTransitions(graph, target);

    // Unified assertion — same for all environments
    for (DesiredNode node : graph.nodes().values()) {
        CaseHubApplicationSpec spec = extractAppSpec(node);
        Optional<String> url = target.serviceUrl(spec.applicationId());
        assertThat(url).as("Service URL for %s", spec.applicationId()).isPresent();

        String healthPath = spec.healthPath() != null ? spec.healthPath() : "/q/health/ready";
        ServiceHealthWaiter.waitForReady(url.get(), healthPath, Duration.ofSeconds(60));

        // Functional HTTP check — service responds
        assertHttpOk(url.get() + healthPath);
    }

    // LOCAL-only bonus: in-JVM LLM exercise
    if (env == TargetEnvironment.LOCAL) {
        FleetLlmExerciser.exercise("my-llm", ...);
    }

    undeploy(graph, target);
}
```

No environment branching in deploy/health/assert. The only `if` is the additive `FleetLlmExerciser` on LOCAL.

### Component 5: Podman port mapping

Update `PodmanDeploymentTarget.deploy()`:

```java
// Before: Map.of() — no ports exposed
var request = new CreateContainerRequest(name, image, Map.of(), env);

// After: expose port 8080 with dynamic host assignment
var request = new CreateContainerRequest(name, image, Map.of(8080, 0), env);
```

Add `containerInspect(String id)` to `PodmanClient` — calls `GET /containers/{id}/json` and extracts `HostConfig.PortBindings` to find the assigned host port.

`PodmanDeploymentTarget.serviceUrl()` calls inspect and returns `http://localhost:<hostPort>`.

## Test strategy

### Unit tests

- `ServiceHealthWaiterTest`: mock HTTP server, verify waits for 200, throws on timeout
- `KubernetesDeploymentTargetTest`: verify `serviceUrl()` returns port-forwarded URL
- `PodmanDeploymentTargetTest`: verify `serviceUrl()` returns inspected port URL
- `CaseHubApplicationSpecTest`: verify `healthPath` defaults to `/q/health/ready`

### Integration tests (integration-live)

- `HelloLlmMultiEnvTest` on all three environments: deploy → serviceUrl → health wait → assert
- `PodmanDeploymentTargetLiveTest`: verify port mapping and service URL resolution

## Trade-offs

- **healthPath widens CaseHubApplicationSpec:** Adds a field to a core record. Justified because health checking is fundamental to deployment verification, and different services have different health endpoints.
- **Port-forward complexity in K8S:** Adds background process management to `KubernetesDeploymentTarget`. Cleanup on undeploy is essential. Mitigated by tracking active port-forwards in a map.
- **Podman dynamic ports:** Host port assignment means tests can't predict the port. Mitigated by `serviceUrl()` resolving it at runtime.
- **FleetLlmExerciser remains LOCAL-only:** Accepted — it tests classpath routing, not service availability. The HTTP layer covers service availability uniformly.

## References

- `api/.../ApplicationDeploymentTarget.java:5-24` — interface to extend with serviceUrl()
- `api/.../CaseHubApplicationSpec.java:12-37` — record to extend with healthPath
- `api/.../HealthCheckResult.java:1-7` — existing health check contract
- `claudony/.../ClaudonyDeploymentTarget.java:18-128` — LOCAL target, fleet URL known
- `container/.../PodmanDeploymentTarget.java:14-75` — Podman target, no port mapping yet
- `container/.../PodmanClient.java:105-124` — CreateContainerRequest with portMappings support
- `topology-tests/.../KubernetesDeploymentTarget.java:20-117` — K8S target with imageOverride
- `topology-tests/.../HelloLlmMultiEnvTest.java` — test to rewrite
- `topology-tests/.../ServiceProcess.java:100-128` — waitForHealth() to extract
- `topology-tests/.../FleetLlmExerciser.java` — in-JVM LLM exercise (LOCAL-only)
- Decision review from #205 — R1-17 identified "local image store" ambiguity across environments
