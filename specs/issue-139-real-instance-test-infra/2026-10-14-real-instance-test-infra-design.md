# Real Instance Integration Test Infrastructure — Design Spec

**Issue:** casehubio/casehub-ops#139
**Branch:** issue-139-real-instance-test-infra
**Blocks:** #130, #132, #133, #134, #135, #136, #137, #138
**Epic:** #131

## Problem

Level 01 (#129) built a data-driven topology test framework that compiles YAML → DesiredStateGraph, plans transitions, and executes them using stubs and WireMock. It validates the YAML→compile→plan→provision pipeline but nothing actually runs. Subsequent levels (#130–#138) need infrastructure to deploy real CaseHub service instances and exercise end-to-end flows.

## Solution

Add integration test infrastructure to the `topology-tests` module that starts real CaseHub service instances as separate JVM processes and manages their lifecycle via a JUnit 5 extension. The proof-of-concept validates that the infrastructure can start claudony and ops-app, and that the deployment API works against a real claudony instance. LLM simulation and end-to-end app deployment are deferred until eidos/engine have app modules.

### Deployability Constraint

Eidos and engine are libraries/Quarkus extensions — they have no standalone `app/` modules and cannot be started as separate processes today. Only claudony (which has `claudony/app/`) and `casehub-ops-app` are deployable Quarkus apps.

**Recommendation for eidos/engine:** add `app/` submodules to each repo (following claudony's pattern), rather than using scaffold as an app-wrapper. Scaffold is a leaf in the dependency graph (depends on engine) — using it to wrap engine would create a circular build dependency. App folders keep the build DAG clean and the packaging collocated with the library.

**For this issue:** scope to claudony + casehub-ops-app (both deployable today). Eidos/engine app modules are a prerequisite for later levels (#130+) and should be filed as a separate issue.

## Architecture

### Components

Three new components in `topology-tests`, package `io.casehub.ops.topology.integration`:

1. **`ServiceProcess`** — wraps ProcessBuilder for a Quarkus fast-jar. Handles jar location, process start, health polling, log capture, and shutdown.

2. **`@ServiceCluster` + `ServiceClusterExtension`** — JUnit 5 extension that reads the annotation, starts services in declared order, waits for health, injects service URLs, detects LLM mode, and tears down after the test class.

3. **`@LlmAvailable`** — JUnit 5 ExecutionCondition that disables tests when no API key is present. Modelled after the existing `@KubernetesAvailable`.

### Service Discovery & Jar Location

Jars are located by convention — no build step in tests:

- **Family root:** `../../` from the ops project root (the casehub family directory). Overridable via `casehub.family.root` system property.
- **Jar path pattern:** `<family-root>/<repo>/target/quarkus-app/quarkus-run.jar`
- **Repo mapping:** annotation declares repo path (e.g., `claudony/app` for claudony, `ops/app` for casehub-ops-app). Future: `eidos/app`, `engine/app` once those repos add app modules.
- **Missing jar:** `IllegalStateException` with actionable message naming the missing path and the build command to produce it.

### Database Provisioning

Claudony requires PostgreSQL (`quarkus.datasource.qhorus.db-kind=postgresql`, reactive). Quarkus Dev Services auto-start a container in dev/test profiles but do NOT activate when running a pre-built jar via ProcessBuilder.

**Solution:** Testcontainers starts a PostgreSQL container in the test JVM. The container's JDBC URL is passed to claudony via system property overrides:
- `-Dquarkus.datasource.qhorus.reactive.url=postgresql://<host>:<port>/claudony`
- `-Dquarkus.datasource.qhorus.jdbc.url=jdbc:postgresql://<host>:<port>/claudony`

Ops-app uses H2 in test profile (`%test.quarkus.datasource.db-kind=h2`), so no PostgreSQL needed for it.

The `ServiceClusterExtension` manages the Testcontainers PostgreSQL lifecycle — started in `beforeAll`, shared across all services that need it, stopped in `afterAll`.

### Authentication

Claudony's REST API requires authentication (`/api/*` → `authenticated` policy). The test profile provides a fixed fleet key (`test-fleet-key-do-not-use-in-prod`).

**Solution:** Start claudony with `-Dquarkus.profile=test`. `HttpClaudonyFleetClient` needs a fleet-key header added, or the extension overrides the auth policy: `-Dquarkus.http.auth.permission.protected.policy=permit`. The latter is simpler for testing — no client changes needed.

### Process Lifecycle

**Port allocation:** Dynamic ports via `ServerSocket(0)` — find a free port, close the socket, pass the port to the service via `-Dquarkus.http.port=<port>`. Avoids hardcoded port collisions (R1-05).

**Startup:**
1. Extension reads `@ServiceCluster`, resolves jar paths
2. Starts Testcontainers PostgreSQL (if any service needs it)
3. Allocates dynamic ports for each service
4. Starts services in declared order (first → last)
5. Each ServiceProcess launches `java -jar quarkus-run.jar` with system properties:
   - `-Dquarkus.http.port=<dynamic>`
   - `-Dquarkus.profile=test` (activates test-profile config: H2 for ops-app, test fleet key for claudony)
   - Database URL overrides (pointing to Testcontainers PostgreSQL)
   - `-Dquarkus.http.auth.permission.protected.policy=permit` (disable auth for testing)
   - Any per-service config overrides from the annotation
6. Health polling: `GET /q/health/ready` every 500ms, 60s timeout
7. On failure: stderr/stdout included in test failure message

**Shutdown:**
- `@AfterAll`: SIGTERM in reverse startup order
- 10s grace period, then `destroyForcibly()`
- Process output captured to `target/service-logs/<service-name>.log`
- Testcontainers PostgreSQL stopped

**Scope:** per test class, not per method.

### LLM Mode Detection (deferred)

The proof-of-concept scope (claudony + ops-app) does not involve LLM calls — those happen inside the deployed apps (eidos/engine/my-llm), which are not yet deployable. LLM mode detection and simulation activation are deferred until eidos/engine have app modules.

The infrastructure includes `LlmMode` enum and `@LlmAvailable` annotation as extension points — they're wired but not exercised until later levels.

**Future mode detection:**
1. Check `System.getenv("ANTHROPIC_API_KEY")` → `LlmMode.FULL`
2. Absent → `LlmMode.SIMULATION`
3. Later: Ollama detection as a third tier for CI

**Future simulation mechanism:**
The platform provides `SimulatedAgentBackend` and `SimulationRuntime`, but activating them inside service JVMs requires either:
- A runtime-switchable `AgentBackend` factory (config property selects implementation)
- Or `@LookupIfProperty`-annotated CDI alternatives in the platform

This is a platform-level change and will be designed when eidos/engine app modules are created. The `SimulatedAgentBackend` → `Simulation.forTest()` → corpus seed pattern is validated by existing platform tests (`ExampleSimulationTest`).

### Test Gating

- `@LlmAvailable` — ExecutionCondition, skips test when `LlmMode.SIMULATION`
- `@Tag("integration-live")` — new JUnit tag for all real-instance tests
- Surefire: excluded by default, activated via Maven profile `-Pintegration-live`
- Follows existing pattern: `reconciliation` and `infra-live` tags/profiles

### Test API

```java
@Tag("integration-live")
@ServiceCluster(services = {
    @Service(name = "claudony", repo = "claudony/app"),
    @Service(name = "ops-app", repo = "ops/app")
})
class HelloLlmIntegrationTest extends TopologyTestBase {

    @ServiceEndpoint("claudony")
    static String claudonyUrl;

    @ServiceEndpoint("ops-app")
    static String opsAppUrl;

    @Test
    void registersAppsViaRealClaudony() throws IOException {
        // Proves: ServiceCluster starts real processes, ClaudonyDeploymentTarget
        // works against a real claudony instance (not WireMock)
        DesiredStateGraph graph = compileSingleGraph(
            "topologies/deployment/01-hello-llm.yaml", DEPLOYMENT_REGISTRY);

        HttpClaudonyFleetClient client = new HttpClaudonyFleetClient(
            claudonyUrl, new ObjectMapper().registerModule(new JavaTimeModule()));
        ClaudonyDeploymentTarget target = new ClaudonyDeploymentTarget(client);

        for (DesiredNode node : graph.nodes().values()) {
            CaseHubApplicationSpec spec = extractAppSpec(node);
            DeployResult result = target.deploy(spec.image(),
                new DeployConfig(spec.applicationId(), spec.replicas(),
                    spec.config(), Map.of()));
            assertThat(result).isInstanceOf(DeployResult.Success.class);
        }

        // Verify apps are registered in claudony's registry
        for (DesiredNode node : graph.nodes().values()) {
            CaseHubApplicationSpec spec = extractAppSpec(node);
            NodeStatus status = target.status(spec.applicationId());
            assertThat(status).isNotEqualTo(NodeStatus.ABSENT);
        }
    }
}
```

**What the proof-of-concept validates:**
1. `ServiceProcess` can locate, start, and manage real Quarkus JVM processes
2. `ServiceClusterExtension` orchestrates multiple services with database provisioning (Testcontainers PostgreSQL), dynamic port allocation, health-check polling, and teardown
3. `ClaudonyDeploymentTarget` can register apps against a real claudony instance (not WireMock), verifying the full REST call path including serialization, HTTP transport, and server-side processing
4. The `integration-live` tag/profile gating works correctly

**What it does NOT validate (deferred):**
- End-to-end deployment of running eidos/engine/my-llm instances (requires app modules)
- LLM simulation via SimulatedAgentBackend (requires services that call the LLM to be running)
- Real LLM task execution (requires both of the above)

**Annotations:**
- `@ServiceCluster(services = {...})` — declares which services to start
- `@Service(name, repo)` — individual service definition (ports allocated dynamically)
- `@ServiceEndpoint("name")` — injects base URL for a named service (static field injection)
- `@LlmAvailable` — gates tests requiring a real LLM (wired but not exercised in proof-of-concept)

### Maven Configuration

New profile in `topology-tests/pom.xml`:

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

Default surefire exclusion updated to include `integration-live`:

```xml
<excludedGroups>reconciliation,infra-live,integration-live</excludedGroups>
```

### New Dependencies

topology-tests pom.xml additions:
- `org.testcontainers:postgresql` (test scope) — PostgreSQL container for claudony
- `org.testcontainers:junit-jupiter` (test scope) — Testcontainers JUnit 5 integration

Deferred dependencies (needed when simulation is activated):
- `casehub-platform-simulation-api` — SimulationCorpus, CorpusSeed (in service JVMs, not test JVM)
- `casehub-platform-agent-simulation-core` — SimulatedAgentBackend (in service JVMs, not test JVM)

## File Inventory

| File | Purpose |
|------|---------|
| `ServiceProcess.java` | Process lifecycle: start, health-poll, stop, log capture |
| `ServiceCluster.java` | Annotation: declares services to start |
| `Service.java` | Annotation: individual service definition |
| `ServiceEndpoint.java` | Annotation: injects service URL into test field |
| `ServiceClusterExtension.java` | JUnit 5 extension: orchestrates ServiceProcess instances |
| `LlmMode.java` | Enum: FULL, SIMULATION |
| `LlmAvailable.java` | Annotation + ExecutionCondition: gates full-mode tests |
| `ServiceEndpoints.java` | Record holding name→URL mappings, resolved by extension |

All in `topology-tests/src/test/java/io/casehub/ops/topology/integration/`.

## Scope Boundaries

**In scope:**
- ServiceProcess and ServiceClusterExtension infrastructure
- Testcontainers PostgreSQL provisioning for claudony
- Dynamic port allocation
- Authentication bypass for testing
- LlmMode enum and @LlmAvailable annotation (wired, not exercised)
- Maven profile for running integration-live tests
- One proof-of-concept integration test (app registration via real claudony)

**Out of scope:**
- LLM simulation activation mechanism — deferred until services that call the LLM are deployable. Requires platform-level CDI changes (runtime-switchable AgentBackend factory or @LookupIfProperty).
- Kubernetes deployment (later issue, `@KubernetesAvailable` pattern exists)
- Ollama/local model support (future extension to LlmMode)
- Per-service composable extensions (can extract from ServiceProcess later)
- CI pipeline configuration for integration-live profile
- Eidos/engine app modules — separate issue, prerequisite for later levels (#130+). Recommendation: `app/` submodules in each repo (not scaffold), following claudony's pattern. Scaffold is a leaf in the dependency graph and cannot wrap upstream libraries without circular build dependencies.

## References

- `topology-tests/src/test/java/.../deployment/DeploymentTargetIntegrationTest.java` — existing WireMock-based integration test pattern
- `topology-tests/src/test/java/.../live/KubernetesAvailable.java` — environment-gated test pattern
- `topology-tests/src/test/java/.../reconciliation/ReconciliationTestBase.java` — test base class pattern
- `platform/agent-simulation-core/.../SimulatedAgentBackend.java` — mock LLM backend
- `platform/simulation-core/.../SimulationRuntime.java` — simulation orchestration
- `platform/simulation-testing/.../LlmCorpusPopulator.java` — corpus generation from real LLM
- `platform/docs/examples/simulation/tutorial/ExampleSimulationTest.java` — simulation API usage
- `claudony/Dockerfile` — Quarkus fast-jar layout, port 7777
- `claudony/docker-compose.yml` — fleet configuration pattern
- `claudony/src/main/java/.../ClaudonyDeploymentTarget.java` — deployment target SPI implementation
- `ops/api/src/main/java/.../ApplicationDeploymentTarget.java` — deployment target interface
- `ops/deployment/src/main/java/.../handler/CaseHubAppProvisionHandler.java` — multi-target provisioning
- `topology-tests/src/test/resources/topologies/deployment/01-hello-llm.yaml` — hello-llm topology
