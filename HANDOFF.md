# Handover — Level 03 landed, manifest-driven LLM routing

**Branch:** `main`
**Issue:** casehubio/casehub-ops#132 (closed)
**Status:** Level 03 complete. Ready for Level 04 (#133).

## What happened this session

### Level 03: Case Management (#132)

Added `03-case-management.yaml` — 7-node deployment topology adding qhorus:
`manifest → eidos → engine/ledger → work → qhorus → my-llm`

Qhorus depends on work + ledger. 4 compilation tests verify graph structure.

### Manifest-driven LLM routing

Replaced hardcoded `ANTHROPIC_API_KEY` detection with manifest-driven provider routing:

- **Topology YAMLs** use `llm.capability: fast` (tier-based) + `llm.identity-source: eidos`
- **ManifestLlmClient** reads the manifest provider chain: Ollama → Claude CLI → Simulation
- **LlmMode** reports FULL vs SIMULATED from the response, not as a gate
- **HelloWorldLlmTest** exercises all three providers individually + chain fallback
- **`@OllamaAvailable`** / **`@LlmAvailable`** condition annotations for per-provider gating

### Eidos agent descriptor

Added `META-INF/eidos/descriptors.yaml` with the my-llm agent identity (briefing, disposition, capability tier). Shows the complete stack: topology declares capability, eidos generates the brief, manifest resolves the provider.

### Platform issue filed and landed

**platform#623**: manifest-driven model chains — tier-based capability routing with provider fallback. Adds chain aliases in manifest, simulation backend, backendKey filter on ModelQuery.

## Key decisions

- **Apps request capability tiers, not providers** — `llm.capability: fast` instead of `llm.provider: anthropic`
- **Eidos generates the LLM brief** — `llm.identity-source: eidos`, not hardcoded system prompts
- **Simulation is a provider, not a mode** — always available as last-resort fallback, tests never skip
- **Vertex rawPredict endpoint not available** for this GCP project — Claude CLI handles Vertex auth natively

## Environment setup

`~/.casehub/agent-config.yaml` (machine-specific, not committed):
```yaml
providers:
  - vendor: ollama
    host: localhost:11434
  - vendor: vertex
    credential:
      project-id: env:ANTHROPIC_VERTEX_PROJECT_ID
      region: us-east5
  - vendor: simulation
aliases:
  fast:
    chain:
      - tier: FAST
        locality: LOCAL
        backend-key: ollama
      - tier: FAST
        locality: CLOUD
      - tier: FAST
        backend-key: simulation
defaults:
  backend: ollama
```

Ollama installed (`/opt/homebrew/bin/ollama`), gemma3:4b model pulled.

## Next work

**Level 04 (#133): Fleet scaling** — deploy 3 LLM instances. Scale: S, Complexity: Med.

## Key files

- `topology-tests/src/test/resources/topologies/deployment/03-case-management.yaml`
- `topology-tests/src/test/resources/META-INF/eidos/descriptors.yaml`
- `topology-tests/src/test/java/io/casehub/ops/topology/integration/ManifestLlmClient.java`
- `topology-tests/src/test/java/io/casehub/ops/topology/integration/HelloWorldLlmTest.java`
- `topology-tests/src/test/java/io/casehub/ops/topology/integration/OllamaAvailable.java`
- `topology-tests/src/test/java/io/casehub/ops/topology/compilation/DeploymentCompilationTest.java`
