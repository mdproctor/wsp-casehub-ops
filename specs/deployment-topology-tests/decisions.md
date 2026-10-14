# Deployment Topology Integration Tests — Decisions

## D1: Test module location

**Choice:** Extend existing `topology-tests` module
**Alternatives:**
- New `app-topology-tests` module — cleaner separation but duplicates test infrastructure
**Rationale:** topology-tests already has the data-driven pattern (TopologyTestBase, ReconciliationTestBase, auto-discovery). Adding deployment-domain node types is additive.
**Trade-offs:** topology-tests gains a second concern (infra + deployment), may need restructuring later
**Sources:** topology-tests/src/test/ existing pattern
**Exploration:** quick
**Status:** captured

## D2: Deployment target

**Choice:** Claudony from the start — it's the deployment target throughout
**Alternatives:**
- StubDeploymentTarget only — validates reconciliation but nothing actually deploys
- Podman — heavier than needed for early levels
- InProcessDeploymentTarget — not meaningfully different from "run this main"
**Rationale:** Without claudony, desiredstate adds nothing over just running the app. Claudony is what makes it fleet deployment. WireMock simulates claudony's REST API in tests.
**Trade-offs:** Requires WireMock setup for claudony fleet API
**Sources:** ClaudonyDeploymentTarget, HttpClaudonyFleetClient
**Exploration:** quick
**Status:** captured

## D3: Minimum viable hello world

**Choice:** Manifest + Eidos + Engine + Blocks + LLM app, all deployed via claudony
**Alternatives:**
- Single agent + channel — not meaningful, just metadata registration
- Single app without engine — deployed but can't do anything
- Without eidos — LLM has no context to reason about
**Rationale:** This is the minimum honest CaseHub app. Anything less and something can't function. Five services but that IS the platform.
**Trade-offs:** Hello world is non-trivial, but accurately reflects the platform
**Sources:** User discussion on what "actually doing something" means
**Exploration:** deep-analysis
**Status:** captured

## D4: Test discovery mechanism

**Choice:** Auto-discover YAML files from `topologies/deployment/` directories
**Alternatives:**
- Explicit JUnit @ParameterizedTest with named list — more control but requires Java changes per topology
**Rationale:** Adding a topology = dropping a YAML file. No Java changes needed. Matches existing infra topology pattern.
**Trade-offs:** Less control over test ordering, harder to skip individual topologies
**Sources:** topology-tests existing discovery pattern
**Exploration:** quick
**Status:** captured

## D5: Tutorial framing

**Choice:** Tests double as an incremental tutorial. YAML header comments explain each level. INDEX.md lists all topologies with reading order.
**Alternatives:**
- Companion markdown per level — richer but more to maintain
- Generated docs — polished but infrastructure overhead upfront
**Rationale:** The YAML IS the tutorial. Each file is self-documenting. More infrastructure later.
**Trade-offs:** Less depth per concept than dedicated docs
**Sources:** User requirement
**Exploration:** quick
**Status:** captured

## D6: Progression — operational capabilities layered on core

**Choice:** Nine levels, iterative with reassessment after each:
1. Core (manifest + eidos + engine + blocks + LLM via claudony)
2. + ledger + work (accountability)
3. + qhorus (case management)
4. Fleet scaling (3 LLMs)
5. Config change (drift reconciliation)
6. Suspend/resume (lifecycle)
7. Add/remove apps (topology mutations)
8. + RAS (fleet monitoring)
9. + chat-app (standalone qhorus UI)
**Alternatives:** None — this emerged from iterative discussion
**Rationale:** Each level adds a platform capability on top of a working core. Demonstrates progressive value. Each level is a separate issue/branch.
**Trade-offs:** Later levels depend on services (RAS, chat-app) that may not exist yet
**Sources:** User requirements, iterative discussion
**Exploration:** deep-analysis
**Status:** captured

## D7: Iterative execution

**Choice:** Each level is a separate issue and branch, reassessed after completion
**Alternatives:** Big-bang implementation of all levels at once
**Rationale:** User preference for baby steps. Each level may change assumptions about subsequent levels.
**Trade-offs:** More overhead per level, but safer
**Sources:** User requirement
**Exploration:** quick
**Status:** captured
