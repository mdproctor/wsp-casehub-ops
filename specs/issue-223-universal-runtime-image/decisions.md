# Decisions — Issue #223: Universal CaseHub Runtime Image

## D1: Module home

**Choice:** New `runtime/` module in casehub-ops
**Alternatives:**
- New top-level repo — clean separation but adds cross-repo CI dependency and maintenance burden
- In casehub-parent — parent is build infrastructure, not a deployable; conflates BOM/parent role with packaging
**Rationale:** Co-located with topology tests that consume it. ops already has a Quarkus shell (app/). CI builds the image from the ops build, tests verify it immediately.
**Trade-offs:** runtime/ couples the universal image to the ops repo. If scaffold or other consumers need to build independently, a separate repo might be needed later.
**Sources:** Issue #223 cross-repo section, ops CLAUDE.md module structure
**Exploration:** quick
**Status:** captured

## D2: Module activation mechanism

**Choice:** No explicit activation needed — Quarkus config properties via env vars control behavior, not CDI bean presence
**Alternatives:**
- @IfBuildProperty — build-time only, can't do same-image-different-config
- Runtime CDI @Startup + config check — unnecessary boilerplate given modules are already inert when unconfigured
- Quarkus profiles — limits to predefined combos
**Rationale:** CaseHub modules are Jandex libraries designed to be inert without goals. GoalCompiler produces nothing without declared entries, ReconciliationLoop is explicitly started, CrossDomainCompositionEngine handles coexistence. Config controls behavior (which goals to reconcile), not presence (which beans exist).
**Trade-offs:** No defense-in-depth against accidental activation. If a module has side effects at CDI startup (unlikely given current design), it would fire in all containers.
**Sources:** CaseHubApplicationSpec.java, RuntimeConfig.java, CrossDomainCompositionEngine (desiredstate#140), ARC42STORIES.MD §2 Constraints
**Exploration:** deep-analysis
**Depends on:** D1 (module structure)
**Status:** captured

## D3: Level 01 topology rewrite

**Choice:** Single casehub_application node with the universal image
**Alternatives:**
- Keep 4 nodes, same image — preserves decomposition narrative but artificial at Level 01
- 2 nodes (platform + app) — partial decomposition, still somewhat fictional
**Rationale:** Level 01 is "deploy your first app." A single node is simpler and more honest than 4 nodes pretending to be separate services. Later levels (03+) demonstrate real multi-node topologies.
**Trade-offs:** Loses the visual decomposition that the original 4-node topology provided. Level 01 walkthrough needs updating.
**Sources:** topology-tests/src/test/resources/topologies/deployment/01-hello-llm/, topology-metadata.yaml
**Exploration:** quick
**Status:** captured

## D4: Ops inclusion in the image

**Choice:** One fat image with everything (platform + ops modules), config controls what's active
**Alternatives:**
- Two image variants (platform-only + full) — smaller base but two builds, two pushes, two versions to track
- Platform-only, no ops — clean but later topology levels need ops in the image
**Rationale:** Later topology levels deploy containers that themselves run ops/desiredstate. Single image simplifies CI and reasoning. Unused modules are inert — disk cost only, no runtime overhead.
**Trade-offs:** Larger image size. All modules loaded into memory even when unused (CDI bean instances are lightweight).
**Sources:** User clarification: "later tests will have them deploying other systems"
**Exploration:** quick
**Status:** captured

## D5: CI scope

**Choice:** Local build only; CI/GHCR push is a follow-up issue
**Alternatives:**
- Full CI pipeline (Dockerfile + GitHub Actions + GHCR push) — complete but larger scope
- Jib (no Dockerfile) — zero Docker dependency but less flexible
**Rationale:** Gets real containers running fastest. CI is mostly boilerplate and can be added independently.
**Trade-offs:** Topology tests can't run in CI until GHCR push is set up (unless CI builds the image first).
**Sources:** Issue #223 sub-task 2
**Exploration:** quick
**Status:** captured

## D6: ImageResolver changes

**Choice:** Add universal image constant; keep per-service resolution for backward compat
**Alternatives:**
- Image ref in topology YAML only — simpler but ImageResolver becomes less central
- Replace per-service resolution entirely — breaking change for existing tests
**Rationale:** Additive change. New constant + overload for universal image. Existing per-service API stays for any tests that still need it.
**Trade-offs:** Two resolution paths in ImageResolver (per-service and universal). May need cleanup once all topologies migrate.
**Sources:** testing/src/main/java/io/casehub/ops/testing/env/ImageResolver.java
**Exploration:** quick
**Status:** captured
