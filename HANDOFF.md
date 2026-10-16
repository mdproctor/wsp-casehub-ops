# Handover — Ready for Operations arc (Levels 05-08)

**Branch:** `main` (clean — previous branch landed)
**Closed this session:** #133 (Level 04 fleet scaling), #143 (LLM identity test)
**Next work:** Operations arc — levels 05-08 (#134, #135, #136, #137)

## What was done this session

### Implementation

1. **#143 — End-to-end LLM identity test** (completed and landed)
   - `LlmIdentityTest.java` — 7 tests proving config → descriptor → briefing → LLM identity
   - Added `casehub-eidos-core` test dependency
   - Bug fix: separated `llm.model-tier` from `llm.capability` in descriptor YAML
   - All 3 integration-live tests pass against Ollama

2. **Closed branch `issue-133-fleet-scaling-3-llm`** — squashed to 3 commits, landed on main

### Planning — topology levels roadmap

Designed the full 18-level progression and filed 17 new issues:

**New topology levels:**
- #146 Level 10: Pool deployment — managed claudony instance pool
- #147 Level 11: Blocks use case — logistics summariser
- #148 Level 12: Ad-hoc LLM deployer — runtime deployment via RAS input
- #149 Level 13: Agentic mesh — multi-agent delegation and communication
- #154 Level 14: Fault recovery — automatic detection and replacement
- #155 Level 15: Multi-tenancy — same topology, isolated tenants
- #156 Level 16: Security — service-to-service auth and credential injection
- #157 Level 17: Compliance posture — continuous monitoring and remediation
- #158 Level 18: Full platform — all modules, all services, everything installed

**Multi-environment epic #150:**
- #151 Environment abstraction layer
- #152 Kubernetes provisioner (k3d/kind)
- #153 Podman provisioner
- #160 Dual-runtime (deploy as Quarkus and Spring — proves framework neutrality)

**Tooling:**
- #159 Showcase UI — run levels from ops app with live results
- #161 Fleet observability UI — live monitoring of deployed topology

**Updated:**
- #131 epic — 18 levels grouped by narrative arc + operational assertions principle
- #137 Level 08 — expanded from "add RAS" to full observability (RAS, health, metrics, application signals)

### Key design decisions

- **Levels are both tutorial and smoke test** — each asserts its runtime capability works end-to-end (deploy → exercise → assert outcome → assert audit)
- **Infrastructure not a level** — dropped "Level 18: Infrastructure" since drift detection (05) and governance gates (02) already demonstrate the concept
- **Dual-runtime as a smoke test** — deploying same topology on Quarkus and Spring proves domain modules are framework-neutral. If CDI leaked into core logic, the Spring build fails — that IS the test
- **Level 09 (Chat UI) flagged** as a standalone app — may need reclassifying out of the level progression

## Roadmap doc

`docs/topology-levels-roadmap.md` in workspace — full graph with status, narrative arcs, operational assertions principle.

## Next session

Start the Operations arc (levels 05-08). Recommended order:

1. **#134 Level 05: Config drift** — S/Med, builds directly on Level 04 fleet
2. **#135 Level 06: Lifecycle** — S/Med, suspend/resume the fleet
3. **#136 Level 07: Topology mutations** — S/Med, add/remove apps
4. **#137 Level 08: Fleet observability** — M/Med, full signal chain

These can share a branch since they're an incremental progression on the same fleet. Create a new branch covering all four, or one per level — user's choice.
