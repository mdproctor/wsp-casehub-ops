# Real Instance Integration Test Infrastructure — Design Spec

**Issue:** casehubio/casehub-ops#139
**Branch:** issue-139-real-instance-test-infra
**Blocks:** #130, #132, #133, #134, #135, #136, #137, #138
**Epic:** #131

## Problem

Level 01 (#129) built a data-driven topology test framework that compiles YAML → DesiredStateGraph, plans transitions, and executes them using stubs and WireMock. It validates the YAML→compile→plan→provision pipeline but nothing actually runs. Subsequent levels (#130–#138) need infrastructure to deploy real CaseHub service instances and exercise end-to-end flows.

## Solution

Add integration test infrastructure to the `topology-tests` module that starts real claudony, eidos, and engine instances as separate JVM processes, manages their lifecycle via a JUnit 5 extension, and supports two modes: full (real LLM) and simulation (platform SimulatedAgentBackend with canned corpus responses).

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
- **Repo mapping:** annotation declares repo path (e.g., `claudony/app`, `eidos/app`, `engine/app`)
- **Missing jar:** `IllegalStateException` with actionable message naming the missing path and the build command to produce it.

### Process Lifecycle

**Startup:**
1. Extension reads `@ServiceCluster`, resolves jar paths
2. Starts services in declared order (first → last)
3. Each ServiceProcess launches `java -jar quarkus-run.jar` with system properties:
   - `-Dquarkus.http.port=<assigned>`
   - `-Dcasehub.simulation.profile=mock-llm` (when simulation mode)
   - Any per-service config overrides from the annotation
4. Health polling: `GET /q/health/ready` every 500ms, 60s timeout
5. On failure: stderr/stdout included in test failure message

**Shutdown:**
- `@AfterAll`: SIGTERM in reverse startup order
- 10s grace period, then `destroyForcibly()`
- Process output captured to `target/service-logs/<service-name>.log`

**Scope:** per test class, not per method.

### LLM Mode Detection

1. Check `System.getenv("ANTHROPIC_API_KEY")`
2. Present → `LlmMode.FULL` — services start normally
3. Absent → `LlmMode.SIMULATION` — services start with simulation profile

Future: Ollama detection as a third tier (local models for CI). For now, API key presence is the only signal.

### Simulation Corpus

- Canned responses as JSON files in `topology-tests/src/test/resources/simulation/corpora/`
- Each file maps to a test scenario (e.g., `hello-llm-corpus.json`)
- Simulation profile configures `SimulatedAgentBackend` (from `platform/agent-simulation-core`) with a `sequential` strategy returning corpus entries in order
- `LlmCorpusPopulator` (from `platform/simulation-testing`) available for recording real interactions to replay later

### Test Gating

- `@LlmAvailable` — ExecutionCondition, skips test when `LlmMode.SIMULATION`
- `@Tag("integration-live")` — new JUnit tag for all real-instance tests
- Surefire: excluded by default, activated via Maven profile `-Pintegration-live`
- Follows existing pattern: `reconciliation` and `infra-live` tags/profiles

### Test API

```java
@Tag("integration-live")
@ServiceCluster(services = {
    @Service(name = "claudony", repo = "claudony/app", port = 17700),
    @Service(name = "eidos", repo = "eidos/app", port = 17701),
    @Service(name = "engine", repo = "engine/app", port = 17702)
})
class HelloLlmIntegrationTest extends TopologyTestBase {

    @ServiceEndpoint("claudony")
    static String claudonyUrl;

    @Test
    void deploysAllServicesAndProcessesRequest() throws IOException {
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
    }

    @Test
    @LlmAvailable
    void fullModeExecutesLlmTask() {
        // Only runs when ANTHROPIC_API_KEY is set
    }
}
```

**Annotations:**
- `@ServiceCluster(services = {...})` — declares which services to start
- `@Service(name, repo, port)` — individual service definition
- `@ServiceEndpoint("name")` — injects base URL for a named service (static field injection)
- `@LlmAvailable` — gates tests requiring a real LLM

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
- `casehub-platform-simulation-api` (test scope) — SimulationCorpus, CorpusSeed
- `casehub-platform-agent-simulation-core` (test scope) — SimulatedAgentBackend

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

Test resources:
- `simulation/corpora/hello-llm-corpus.json` — canned LLM responses for hello-llm scenario

## Scope Boundaries

**In scope:**
- ServiceProcess and ServiceClusterExtension infrastructure
- LLM mode detection (API key)
- Simulation corpus for hello-llm scenario
- Maven profile for running integration-live tests
- One proof-of-concept integration test (hello-llm deployment against real claudony)

**Out of scope:**
- Docker/Testcontainers (later issue)
- Kubernetes deployment (later issue, `@KubernetesAvailable` pattern exists)
- Ollama/local model support (future extension to LlmMode)
- Per-service composable extensions (can extract from ServiceProcess later)
- CI pipeline configuration for integration-live profile

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
