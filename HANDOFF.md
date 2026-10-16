# Handover — Fleet scaling + LLM identity test complete

**Branch:** `issue-133-fleet-scaling-3-llm`
**Issues:** casehubio/casehub-ops#133 (done), casehubio/casehub-ops#143 (done)
**Status:** Both issues complete. Branch ready for work-end.
**Base:** rebased on 0.3 (upstream/main)

## What was done

### Issue #133 — Fleet scaling (Level 04)

Fleet topology with CSV data source + forEach expansion, deploying 3 differentiated LLM instances:

- `04-llm-fleet.yaml` — topology with `data` block + `forEach` expanding 3 nodes
- Parameterised eidos descriptors with `${config.*}` references
- 5 compilation tests (node count, types, IDs, dependencies, per-instance config)
- Integration test deploying 3 real instances to claudony
- Archetype, domain, and org identity fields in fleet descriptors

### Issue #143 — End-to-end LLM identity test

Proves the full parameter pipeline: topology config → `${config.*}` resolution → AgentDescriptor → briefing text → LLM response reflects identity.

- `LlmIdentityTest.java` — 7 tests (4 deterministic + 3 integration-live)
- Deterministic: analyst identity, coder identity, different briefings, no placeholders
- Integration-live: LLM response reflects analyst identity, coder identity, different responses
- All 3 integration-live tests pass against Ollama (gemma3:4b)
- Bug fix: separated `llm.model-tier` from `llm.capability` in descriptor YAML

### Planning — topology levels roadmap

Designed and filed the full 18-level progression plus infrastructure:

- **Roadmap doc:** `docs/topology-levels-roadmap.md` (full graph with status)
- **New level issues:** #146-149 (Levels 10-13), #154-158 (Levels 14-18)
- **Multi-environment epic:** #150 with children #151-153 (env abstraction, k8s, podman)
- **Dual-runtime:** #160 (deploy as Quarkus and Spring to prove framework neutrality)
- **Showcase UI:** #159 (run levels from ops app with live results)
- **Epic #131 updated:** 18 levels grouped by narrative arc, operational assertions principle

## Key files

| File | Purpose |
|------|---------|
| `topology-tests/src/test/resources/topologies/deployment/04-llm-fleet.yaml` | Fleet topology with CSV data + forEach |
| `topology-tests/src/test/resources/META-INF/eidos/descriptors.yaml` | Static + parameterised descriptors |
| `topology-tests/src/test/java/.../DeploymentCompilationTest.java` | Level 01-04 compilation tests |
| `topology-tests/src/test/java/.../FleetScalingIntegrationTest.java` | Real claudony deployment test |
| `topology-tests/src/test/java/.../LlmIdentityTest.java` | End-to-end identity pipeline test |

## What remains on this branch

Nothing. Both issues complete. Run `work end` to close.

## Next work

Continue through the levels. Candidates in priority order:
1. #134 Level 05: Config drift (builds on 04, natural next step)
2. #140 Claudony NodeProvisioner (infrastructure for later levels)
3. #146 Level 10: Pool deployment (new pattern)
