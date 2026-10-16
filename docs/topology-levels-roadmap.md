# Topology Levels Roadmap

Progressive tutorial + operational smoke test suite. Each level introduces one
platform capability, teaches how it works, and asserts it functions end-to-end.

## Structure

```
Epic #131: Deployment topology integration tests — progressive tutorial
|
+-- Infrastructure
|   +-- [x] #139  Real instance integration test infrastructure
|   +-- [ ] #150  Epic: Multi-environment deployment (see below)
|
+-- Basics — here's an app, here's how it deploys
|   +-- [x] #129  Level 01: Hello LLM — eidos + engine + LLM via claudony
|   +-- [x] #130  Level 02: Accountability — + ledger + work
|   +-- [x] #132  Level 03: Case management — + qhorus
|   +-- [~] #133  Level 04: Fleet scaling — 3 LLM instances (code done, issue open)
|       +-- [~] #143  End-to-end LLM identity test (code done, issue open)
|
+-- Operations — here's how it runs day-to-day
|   +-- [ ] #134  Level 05: Config drift — change config, only diff reconciled
|   +-- [ ] #135  Level 06: Lifecycle — suspend / resume
|   +-- [ ] #136  Level 07: Topology mutations — add / remove apps
|   +-- [ ] #137  Level 08: Fleet monitoring — + RAS
|
+-- Patterns — here's what you build with it
|   +-- [ ] #138  Level 09: Chat UI — standalone qhorus conversation viewer
|   +-- [ ] #146  Level 10: Pool deployment — managed claudony instance pool
|   +-- [ ] #147  Level 11: Blocks use case — logistics summariser
|   +-- [ ] #148  Level 12: Ad-hoc LLM deployer — runtime deployment via RAS input
|   +-- [ ] #149  Level 13: Agentic mesh — multi-agent delegation and communication
|
+-- Enterprise — resilience, tenancy, security, compliance
|   +-- [ ] #154  Level 14: Fault recovery — automatic detection and replacement
|   +-- [ ] #155  Level 15: Multi-tenancy — same topology, isolated tenants
|   +-- [ ] #156  Level 16: Security — service-to-service auth and credential injection
|   +-- [ ] #157  Level 17: Compliance posture — continuous monitoring and remediation
|
+-- Capstone
|   +-- [ ] #158  Level 18: Full platform — all modules, all services, everything installed
|
+-- Tooling
    +-- [ ] #159  Showcase UI — run levels from the ops app with live results


Epic #150: Multi-environment topology deployment — local, Kubernetes, Podman
|
+-- [ ] #151  Environment abstraction layer — target selection for topology tests
+-- [ ] #152  Kubernetes provisioner — deploy topology nodes to k8s via k3d/kind
+-- [ ] #153  Podman provisioner — deploy topology nodes to podman pods
+-- [ ] #160  Dual-runtime — deploy as Quarkus and Spring to prove framework neutrality
```

## Status key

- `[x]` — done and closed
- `[~]` — code complete on branch (not yet merged/closed)
- `[ ]` — not started

## Narrative arc

| Arc | Levels | What it teaches |
|-----|--------|----------------|
| Basics | 01-04 | Here's an app, here's how it deploys |
| Operations | 05-08 | Here's how it runs day-to-day |
| Patterns | 09-13 | Here's what you build with it |
| Enterprise | 14-17 | Resilience, tenancy, security, compliance |
| Capstone | 18 | Everything together |

## Operational assertions principle

Every level asserts its runtime capability WORKS, not just that deployment
succeeded. Each test is a lightweight operational smoke test:

1. **Deploy** — topology compiles, nodes provision, services register
2. **Exercise** — send real input through the deployed capability
3. **Assert outcome** — verify expected output, state change, or side effect
4. **Assert audit** — verify ledger/work/RAS recorded the operation

## Notes

- Level 09 (Chat UI) was flagged as a standalone app — may need reclassifying
- Multi-environment (#150) is cross-cutting — all levels should work on all targets
- Showcase UI (#159) runs levels from the ops app with live streaming results
- Dual-runtime (#160) proves framework neutrality — deploy same topology as Quarkus or
  Spring. Each build includes a framework-specific component proving it IS that runtime.
  If a CDI annotation leaked into core logic, the Spring build fails — that's the test.
