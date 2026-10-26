## D1: Assertion model — HTTP-only vs in-JVM

**Choice:** HTTP assertions everywhere + FleetLlmExerciser as LOCAL-only bonus
**Alternatives:**
- HTTP-only unified — drops FleetLlmExerciser entirely, losing deeper in-JVM integration coverage on LOCAL
- Port-forward to enable in-JVM everywhere — complex wiring to make classpath-based test work with external containers
**Rationale:** All environments get identical HTTP-level assertions (health check + REST exercise). LOCAL additionally runs `FleetLlmExerciser` for in-JVM agent routing verification. Two assertion tiers: HTTP is the contract, in-JVM is a bonus.
**Trade-offs:** LOCAL still has an environment branch for the in-JVM exercise. Accepted because the in-JVM layer tests fundamentally different things (classpath routing) than the HTTP layer (service availability).
**Sources:** FleetLlmExerciser.java (in-JVM langchain4j), HelloLlmMultiEnvTest.java
**Exploration:** quick
**Status:** captured

## D2: Health check path configuration

**Choice:** Configurable health path per service
**Alternatives:**
- Try /q/health/ready then fall back to / — auto-detecting but adds latency per placeholder
- Skip health for placeholders — inconsistent treatment between real and placeholder services
**Rationale:** Real CaseHub services use `/q/health/ready` (Quarkus standard). Placeholder services (nginx:alpine) return 200 on `/`. The health path comes from the topology YAML spec (new optional `healthPath` field on `CaseHubApplicationSpec`) defaulting to `/q/health/ready`.
**Trade-offs:** Adds a field to `CaseHubApplicationSpec`. Topology YAML must be updated for placeholder nodes. Alternatively, the test harness could derive the health path from whether the image is real or fallback.
**Sources:** ServiceProcess.waitForHealth() (line 100-128), CaseHubApplicationSpec.java
**Exploration:** quick
**Status:** captured

## D3: serviceUrl() API location

**Choice:** Default method on `ApplicationDeploymentTarget`
**Alternatives:**
- Separate `ServiceEndpointResolver` interface — more flexible but adds a parallel interface hierarchy
- Return URL from `DeployResult.Success` — couples deploy and URL resolution
**Rationale:** `ApplicationDeploymentTarget` already has `deploy()`, `status()`, `healthCheck()`. Adding `serviceUrl(String instanceId)` keeps all service interaction in one place. Default implementation throws `UnsupportedOperationException`. Each target overrides.
**Trade-offs:** Widens an existing interface. Targets that don't support URL resolution need the default no-op.
**Sources:** ApplicationDeploymentTarget.java:5-24
**Exploration:** quick
**Status:** captured

## D4: K8S port resolution mechanism

**Choice:** kubectl port-forward via testcontainer exec
**Alternatives:**
- NodePort services — simpler but NodePort range is limited and requires container IP lookup
- ClusterIP from host network — complex networking setup
**Rationale:** Run `kubectl port-forward` inside the k3s container, mapping the K8s Service port to a host-accessible port. The test JVM connects to `localhost:<mapped-port>`. This is the standard testcontainers-k3s pattern for reaching services.
**Trade-offs:** Port-forward is a background process that needs cleanup. Adds complexity to K3dClusterExtension lifecycle. Port conflicts possible (mitigated by dynamic port allocation).
**Sources:** K3dClusterExtension.java, KubernetesDeploymentTarget.java
**Exploration:** quick
**Depends on:** D3 (serviceUrl returns the port-forwarded localhost URL)
**Status:** captured
