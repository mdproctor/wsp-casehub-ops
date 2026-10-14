## D1: Deployment surface

**Choice:** Separate JVM processes — each service runs in its own JVM via ProcessBuilder
**Alternatives:**
- Docker/Testcontainers — realistic but heavyweight, requires Docker daemon, overkill for this stage
- In-process classpath composition — fast but loses inter-service HTTP boundary testing
**Rationale:** Real HTTP boundaries between services without Docker overhead. K8s deferred to a later issue.
**Trade-offs:** Must manage process lifecycle, port allocation, and health-check polling manually
**Sources:** claudony/Dockerfile (shows Quarkus fast-jar layout), casehub-ops-app application.properties
**Exploration:** quick
**Status:** captured

## D2: Image source

**Choice:** Assume pre-built jars in sibling repo target/ directories
**Alternatives:**
- Build as needed — self-contained but much slower test startup
- Configurable — unnecessary complexity for now
**Rationale:** CI and developers already run `mvn install` on the parent. Tests locate jars by convention.
**Trade-offs:** Tests fail with a clear message if jars are missing rather than building on demand
**Sources:** claudony/app/target/quarkus-app/, eidos/app/target/quarkus-app/
**Exploration:** quick
**Status:** captured

## D3: Mock LLM approach

**Choice:** Platform simulation API — SimulatedAgentBackend with corpus-based canned responses
**Alternatives:**
- WireMock stubs — tests HTTP boundary but duplicates what simulation already does
- Purpose-built mock HTTP app — another artifact to maintain
**Rationale:** The platform already provides SimulatedAgentBackend, SimulationRuntime, Simulation.forTest(), corpus seeds, and LlmCorpusPopulator. Services use AgentBackend SPI — simulation profile activates the simulated backend. No new mock service needed.
**Trade-offs:** Doesn't test the actual LLM HTTP call path in mock mode (full mode still does)
**Sources:** platform/agent-simulation-core SimulatedAgentBackend.java, platform/simulation-api SimulationCorpus.java, platform/simulation-core SimulationRuntime.java, platform/docs/examples/simulation/tutorial/ExampleSimulationTest.java
**Exploration:** quick
**Status:** captured

## D4: LLM mode detection

**Choice:** API key presence — ANTHROPIC_API_KEY env var present → full mode, absent → mock/simulation mode
**Alternatives:**
- Explicit system property (-Dtest.llm.mode=full|mock) — more control but another config knob
- Layered (property overrides, key is default) — unnecessary complexity for now
**Rationale:** Matches CI/local dev reality. CI without API key runs simulation; developers with a key get full mode. Extensible later for Ollama (local model detection as a third tier).
**Trade-offs:** Can't force simulation mode when API key is present without unsetting the env var
**Sources:** Issue #139 design questions
**Exploration:** quick
**Status:** captured

## D5: Module location

**Choice:** New package in topology-tests — io.casehub.ops.topology.integration
**Alternatives:**
- Separate Maven module (integration-tests/) — cleaner boundaries but more build overhead
- Split infra/tests across modules — unnecessary fragmentation
**Rationale:** Keeps all topology testing together. Shares TopologyTestBase, ReconciliationTestBase. Follows existing package structure (compilation, reconciliation, deployment, live).
**Trade-offs:** topology-tests pom.xml gains new dependencies (simulation, potentially sibling app jars)
**Sources:** topology-tests module structure, existing package layout
**Exploration:** quick
**Status:** captured

## D6: Process lifecycle management

**Choice:** JUnit 5 extension — @ServiceCluster annotation with declarative service definitions
**Alternatives:**
- Test base class — simpler but less composable, inheritance hierarchy gets deep
- External setup (script/Maven plugin) — simplest tests but more build config
**Rationale:** Composable with other extensions. Follows existing pattern from @KubernetesAvailable. Manages process start, health-wait, shutdown declaratively.
**Trade-offs:** Extension complexity is higher upfront; pays off across many test classes
**Sources:** topology-tests/live/KubernetesAvailable.java pattern
**Exploration:** quick
**Status:** captured

## D7: Service cluster extension architecture

**Choice:** Monolithic @ServiceCluster annotation — one annotation declares all services to start
**Alternatives:**
- Per-service extensions — maximum composability but startup ordering is harder, more boilerplate
- Builder-based programmatic — most flexible but more code per test, manual lifecycle
**Rationale:** Most integration tests need the same set of services (claudony + eidos + engine). Annotation-driven keeps test classes clean. For subset tests, declare fewer services.
**Trade-offs:** Less composable than per-service extensions; rare partial-cluster tests need a different annotation set
**Sources:** @KubernetesAvailable pattern, DeploymentTargetIntegrationTest WireMock lifecycle
**Exploration:** quick
**Status:** captured

## D8: Deployability — eidos/engine app modules

**Choice:** Scope #139 to claudony + casehub-ops-app only. Eidos/engine app modules as a separate prerequisite issue.
**Alternatives:**
- Include eidos/engine app module creation in this issue — expands scope into cross-repo work
- Use scaffold as app wrapper — circular dependency (scaffold is a leaf, depends on engine)
**Rationale:** Eidos and engine are libraries with no standalone app modules. Only claudony and ops-app are deployable today. Creating app modules is cross-repo work that should be tracked separately. The app/ submodule pattern (following claudony) is recommended over scaffold — scaffold is a product, not an app-wrapper mechanism, and its position as a dependency leaf creates build ordering issues.
**Trade-offs:** Later levels (#130+) that need individual eidos/engine deployment are blocked until the app modules exist
**Depends on:** D1 (separate JVM processes)
**Sources:** eidos repo structure (no app/ module), engine repo structure (no app/ module), claudony/app/ (proven pattern), scaffold CLAUDE.md (leaf dependency position)
**Exploration:** quick
**Status:** captured
