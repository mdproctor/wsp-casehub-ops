# Handoff

## Last Session — 2026-11-03/04

Completed **#227** (Level 03 topology rewrite) and **#236** (qhorus runtime integration + e2e behavior test). Filed **#237** (templated e2e test dimensions).

### What was built

**#227** — Rewrote Level 03 topology from 7 fictional nodes to 3 real nodes (platform, ledger, qhorus) using the universal runtime image. Compilation tests (5), reconciliation tests (4), acceptance test updated. Landed as `6b50fc8` on main.

**#236** — Added qhorus modules (`casehub-qhorus` + `casehub-qhorus-persistence-memory`) to the universal runtime image. Fixed multi-persistence-unit CDI wiring. Wrote `QhorusPodmanE2eTest` — deploys a container via Podman and exercises real application logic: platform CRUD (create application, verify persistence) + qhorus channels (create channel, post COMMAND message, verify round-trip). Wrote `docs/guides/topology-levels-strategy.md` — the living strategy doc for levels, testing dimensions, and interactive tutorials. Landed as `52c49a7` on main.

### Blog entries

- `blog/2026-11-03-mdp01-one-image-to-rule-them-all.md` — universal runtime image (Level 01/02, from prior session)
- `blog/2026-11-04-mdp01-the-test-is-the-tutorial.md` — test dimensions, runtime proof, tutorial vision

## Recommended Next Work

### 1. #237 — Templated e2e test dimensions (START HERE)

Design and build the reusable dimension framework so each topology level declares which capabilities it tests. This is the foundation everything else depends on.

**Key artifacts:**
- Strategy doc: `docs/guides/topology-levels-strategy.md` (read this first — has dimension contract, YAML descriptor format, and phase progression)
- Reference test: `topology-tests/src/test/java/io/casehub/ops/topology/env/QhorusPodmanE2eTest.java` (Level 03 behavior test to extract dimensions from)
- Reference test: `topology-tests/src/test/java/io/casehub/ops/topology/env/HelloLlmPodmanE2eTest.java` (Level 01 behavior test)

**What to build:**
- `TestDimension` interface — name, requires, execute(urls, state)
- `ServiceUrls` abstraction — target-agnostic URL provider (Podman, K8s, local)
- Concrete dimensions: Deploy, Platform, LLM, Channels
- Refactor QhorusPodmanE2eTest and HelloLlmPodmanE2eTest to use dimensions

### 2. #228 — Level 04 topology (LLM Fleet)

Next level in #226 epic. Adds the Fleet dimension (multiple LLM instances, per-instance routing). Follow the same pattern: YAML rewrite + compilation + reconciliation + e2e behavior test using the new dimension framework from #237.

### 3. Strategy doc refinement

`docs/guides/topology-levels-strategy.md` covers the four-phase arc:
1. Java dimension tests (current) → 2. YAML descriptors → 3. Scaffold UI tutorials → 4. Continuous verification

The YAML descriptor format and scaffold UI integration sections need design work once #237's dimension framework is built.

## Epic Status: #226 — Rewrite all topology levels to universal runtime

| # | Title | Status |
|---|-------|--------|
| #223 | Level 01: Hello LLM | Done |
| #225 | Level 02: Accountability | Done |
| #227 | Level 03: Case Management | Done |
| #236 | Level 03: Qhorus runtime + e2e | Done |
| #228 | Level 04: LLM Fleet | Open — next |
| #229 | Level 05: Config Drift | Open |
| #230 | Level 06: Lifecycle | Open |
| #231 | Level 07: Topology Mutations | Open |
| #232 | Level 08: Fleet Observability | Open |
| #233 | Level 09: Chat UI | Open |
| #234 | Levels 10, 12, 14: Claudony domain | Open |
| #235 | Level 15: Governed Interaction | Open |

## Related Issues

- **#237** — Templated e2e test dimensions (foundational — do before more levels)
- **#200** — Epic: Retrofit system-level acceptance tests across all topology levels (overlaps with #237)

## Key Design Decisions

- **Service boundaries by state independence** — co-locate what's coupled (engine+work+eidos+manifest → platform), separate what manages independent state (ledger = audit, qhorus = protocol/channels)
- **Tests verify runtime behavior** — not just deployment. Every level must exercise application logic through REST APIs.
- **Same tests become tutorials** — dimension framework → YAML descriptors → scaffold UI interactive tutorials with visual feedback
- **Incremental module addition** — add dependencies to the universal runtime as each level needs them (qhorus added for Level 03, next modules for Level 04+)
