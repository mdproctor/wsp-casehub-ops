# Handover — Level 04 fleet scaling in progress

**Branch:** `issue-133-fleet-scaling-3-llm`
**Issues:** casehubio/casehub-ops#133, casehubio/casehub-ops#143
**Status:** Core fleet topology and tests complete. Org structure and end-to-end LLM identity test remaining.
**Base:** rebased on 0.3 (upstream/main)

## What was done

### 1. Fleet topology YAML with CSV data source

`topology-tests/src/test/resources/topologies/deployment/04-llm-fleet.yaml`

Builds on Level 01 (hello-llm). Same 3 platform services (manifest, eidos, engine), but replaces the single `my-llm` node with a parameterised fleet using `data` block + `forEach`:

```yaml
data:
  llm-fleet:
    inline: |
      name:STRING,role:STRING,capability:STRING,archetype:STRING,adjective:STRING,domain:STRING
      analyst,analyst,deep,investigator,methodical,analysis
      coder,developer,fast,engineer,pragmatic,software-engineering
      assistant,assistant,balanced,coordinator,versatile,general
```

ForEachExpander produces 3 nodes: `my-llm.analyst`, `my-llm.coder`, `my-llm.assistant`. Each gets a distinct `applicationId` and config values from the CSV row.

Per-instance config includes: `llm.agent-id`, `llm.name`, `llm.role`, `llm.capability`, `llm.archetype`, `llm.adjective`, `llm.domain`, `llm.identity-source`.

### 2. Parameterised eidos descriptor

`topology-tests/src/test/resources/META-INF/eidos/descriptors.yaml`

Replaces 3 static fleet descriptors with 1 parameterised entry using `${config.*}` references:

```yaml
- agentId: ${config.llm.agent-id:my-llm-fleet}
  name: ${config.llm.name:Fleet LLM}
  slot: ${config.llm.role:assistant}
  archetype: ${config.llm.archetype:assistant}
  archetypeAdjectives:
    - ${config.llm.adjective:versatile}
  domainVocabulary: ${config.llm.domain:general}
  ...
```

`DescriptorPreprocessor` in `eidos-core` resolves these via MicroProfile Config (Quarkus) or Spring Environment at app startup. No code needed — declarative.

The static `my-llm` descriptor for Level 01 is preserved alongside it.

### 3. Compilation tests (5 tests)

`topology-tests/src/test/java/io/casehub/ops/topology/compilation/DeploymentCompilationTest.java`

- `llmFleet_hasSixNodes` — 3 platform + 3 LLM instances
- `llmFleet_allNodesAreCasehubApplication` — all node types correct
- `llmFleet_threeExpandedLlmNodes` — node IDs: my-llm.analyst, my-llm.coder, my-llm.assistant
- `llmFleet_allLlmNodesDependOnEngine` — dependency edges correct
- `llmFleet_configValuesFlowPerInstance` — asserts per-instance config: analyst→deep/analyst, coder→fast/developer, assistant→balanced/assistant

### 4. Integration test

`topology-tests/src/test/java/io/casehub/ops/topology/integration/FleetScalingIntegrationTest.java`

- `deploysThreeFleetInstancesViaRealClaudony` — deploys all 6 nodes to real claudony, verifies deploy success and healthy status for all 3 LLM instances
- `fleetInstancesHaveDistinctApplicationIds` — verifies the 3 expanded nodes have distinct applicationIds

Tagged `@Tag("integration-live")` — requires claudony jar built (`mvn -pl app package -DskipTests` in claudony repo) and `-Pintegration-live` profile.

### 5. Version bump

Branch rebased on upstream/main which has the 0.3 release. `mvn install -DskipTests` completed — all 0.3 artifacts in local .m2.

## What remains

### A. Org structure in topology YAML

The 3 fleet agents need an organisational structure expressing their relationships. Now unblocked — `@NodeTypeId("org:unit")` and `@NodeTypeId("org:relationship")` were added in desiredstate#171 (landed in 0.3).

**What to do:**
1. Register `org:unit` and `org:relationship` in the type registry alongside `DeploymentNodeSpec` and `InfraNodeSpec`
2. Add nodes to `04-llm-fleet.yaml`:
   - One `org:unit` node for the fleet (unitId: llm-fleet, members: the 3 agents with roles)
   - `org:relationship` nodes for delegation between agents (e.g., analyst DELEGATES_TO coder for implementation, coder DELEGATES_TO analyst for investigation)
3. Add compilation tests asserting org nodes compile and have correct structure
4. The org node depends on the fleet members (the unit references their agentIds)

**Key types:**
- `OrgUnitNodeSpec(OrganizationalUnit unit)` — wraps an `OrganizationalUnit` with `unitId`, `name`, `kind`, `tenancyId`, `members` (list of `Membership(agentId, role, roleVocabulary)`)
- `OrgRelationshipNodeSpec(AgentRelationship relationship)` — wraps `AgentRelationship` with `sourceAgentId`, `targetAgentId`, `kind` (SUPERVISES, DELEGATES_TO, ESCALATES_TO, REPORTS_TO, BACKS_UP, EXTENDED), `scope`, `tenancyId`
- `OrgNodeTypes.UNIT` = `"org:unit"`, `OrgNodeTypes.RELATIONSHIP` = `"org:relationship"`

**Reference:** `desiredstate/eidos/src/test/java/io/casehub/eidos/org/runtime/OrgGoalCompilerTest.java` — shows how org units and relationships are constructed and what the compiled graph looks like.

### B. End-to-end LLM identity test (ops#143)

Prove the full parameter pipeline: topology CSV → forEach expansion → per-instance config → eidos descriptor resolution → rendered system prompt → LLM output reflects the agent's identity.

**Approach (from ops#143):**
1. Add `casehub-eidos-core` as a test dependency in topology-tests (needed for `DescriptorPreprocessor` and render pipeline)
2. Load the parameterised descriptor YAML
3. Resolve `${config.*}` with test config values using `DescriptorPreprocessor`
4. Render the system prompt via `DefaultPromptAssembler` or `EidosRenderPipeline`
5. Send system prompt + user prompt to LLM via `ManifestLlmClient` (Ollama/Claude CLI/simulation)
6. Assert response reflects the configured identity (role, archetype, domain)

**Key classes in eidos-core 0.3:**
- `io.casehub.eidos.core.yaml.DescriptorPreprocessor` — resolves `${config.*}`
- `io.casehub.eidos.core.renderer.DefaultPromptAssembler` — assembles system prompt from descriptor
- `io.casehub.eidos.core.renderer.EidosRenderPipeline` — full render pipeline
- `io.casehub.eidos.core.registrar.ClasspathYamlDescriptorRegistrar` — loads META-INF/eidos/descriptors.yaml

**Testing tiers:**
- Simulation: deterministic, always runs
- Ollama: `@OllamaAvailable` guard, gemma3:4b model
- Claude CLI: `@LlmAvailable` guard, Vertex auth via claude CLI

**Cross-platform note:** The `${config.*}` mechanism needs to work for both Quarkus (MicroProfile Config) and Spring (Environment) deployments — the ops examples demonstrate both "deploy as Quarkus" and "deploy as Spring". The preprocessor abstraction is an eidos concern, not ops.

## Design decisions (in workspace)

All captured in `specs/issue-133-fleet-scaling-3-llm/decisions.md`:
- D1: forEach + iterations pattern (not explicit nodes or replicas)
- D2: Single integration test class
- D3: CSV data source for per-instance parameterisation
- D4: Both levels — static descriptors work (Level A), topology params demonstrate config-driven flow (Level B)

## Key files

| File | Purpose |
|------|---------|
| `topology-tests/src/test/resources/topologies/deployment/04-llm-fleet.yaml` | Fleet topology with CSV data + forEach |
| `topology-tests/src/test/resources/META-INF/eidos/descriptors.yaml` | Static (Level 01) + parameterised (Level 04) descriptors |
| `topology-tests/src/test/java/io/casehub/ops/topology/compilation/DeploymentCompilationTest.java` | Level 01-04 compilation tests |
| `topology-tests/src/test/java/io/casehub/ops/topology/integration/FleetScalingIntegrationTest.java` | Real claudony deployment test |
| `topology-tests/src/test/java/io/casehub/ops/topology/integration/ManifestLlmClient.java` | LLM provider client (Ollama/Claude CLI/simulation) |
| `topology-tests/src/test/java/io/casehub/ops/topology/integration/HelloWorldLlmTest.java` | Per-provider LLM tests from #132 |

## Workspace files

| File | Purpose |
|------|---------|
| `specs/issue-133-fleet-scaling-3-llm/2026-10-15-fleet-scaling-design.md` | Design spec (needs updating for expanded scope) |
| `specs/issue-133-fleet-scaling-3-llm/decisions.md` | 4 captured decisions |
| `plans/2026-10-15-fleet-scaling.md` | Implementation plan (covers original scope, needs extending) |

## Future levels (noted during brainstorming, out of scope)

- **Pooling and scaling** — dynamic instance count based on load
- **LLM-as-deployer** — an LLM that ad-hoc deploys other LLMs
