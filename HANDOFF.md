# Handover — Level 12 ad-hoc LLM deployer complete

**Branch:** `main` (clean)
**Landed this session:** #148

## What was done

1. **#148** Level 12: Ad-hoc LLM Deployer — landed on main as `5ddfba5`. Topology YAML with deployer node + specialist catalog, `AdhocLlmDeployerRecompiler` (SituationRecompiler SPI implementation), 5 compilation tests, 6 unit tests, 4 integration tests. Tests cover: full lifecycle (deploy/remove), concurrent alerts, rejection, specialist health failure. Design review surfaced the existing adaptation rule system (`DeploymentAdaptiveSituationRecompiler`) — decision D3 chose to follow its pattern but implementation used direct `withNode()`/`withoutNode()` because `AddActionSpec` is coupled to `DeploymentGoals`.

2. **#189** filed — extend the adaptation rule framework for LLM-evaluated triggers and cross-domain add actions. This makes Level 12's deployer expressible as a pure YAML rule (matching all other levels). Key motivation: YAML-driven rules iterate faster (no compile cycle), work better with LLM coding assistants, and preserve audit/reproducibility.

## Key Design Insight

All other topology levels (L01-L14) are pure YAML — the framework Java handles everything. Level 12 is the first requiring per-level Java (`AdhocLlmDeployerRecompiler`). #189 fixes this by making LLM-evaluated triggers a framework capability. Once landed, Level 12's Java recompiler is replaced by a YAML rule and removed. Nothing in the ops domain benefits from compiled Java speed — all hot paths are I/O-bound (provisioning, LLM calls).

3. **Blog entry** written: "The LLM That Deploys LLMs" — covers the runtime proof, adaptation system discovery, YAML-first vs Java SPI contrast, and the path to #189. Two SVG diagrams (mutation flow, YAML-vs-Java comparison).

## Immediate Next Step

`work start #189` — extend adaptation rule framework. L/High. Two framework changes needed:
1. New `AdaptationTrigger` type: `llm-evaluated` (delegates to deployed LLM node)
2. Domain-agnostic `AddActionSpec` (currently coupled to `DeploymentGoals`/`DeploymentGoalCompiler`)

## References

- Level 12 topology: `topology-tests/src/test/resources/topologies/deployment/12-adhoc-llm-deployer/`
- Design spec: `specs/issue-148-adhoc-llm-deployer/2026-10-25-adhoc-llm-deployer-design.md` (in workspace)
- Existing adaptation system: `deployment/src/main/java/io/casehub/ops/deployment/adaptation/`
- Follow-up issue: casehubio/casehub-ops#189
