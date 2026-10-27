# Handoff — casehub-ops

## Active Work

**Branch:** `issue-207-multi-env-deploy-testing` (paused)
**Epic:** #207 — Harden multi-environment deployment testing (LOCAL/K8S/PODMAN parity)
**State:** design complete, implementation plan written, ready to execute

## What to Do

`work resume` → then start executing the plan at `/Users/mdproctor/claude/public/casehub-ops/plans/2026-10-27-multi-env-deploy-testing.md`

Batches 1-2 and 4 can start immediately. Batch 3 is blocked on cross-repo prerequisites (platform#666, eidos#208).

## Cross-Repo Issues Filed

- **casehubio/platform#666** — `agent-rest` module: structured HTTP agent invocation endpoint + `AgentResolver` SPI. Prerequisite for ops #208.
- **casehubio/eidos#208** — implement `AgentResolver` SPI using `SystemPromptRenderer`. Depends on platform#666.

## Architecture Decisions

1. **K8S/PODMAN deploy path**: direct `target.deploy()` (not TransitionPlanner). Runtime validated by reconciliation tests.
2. **HTTP agent invocation**: platform `POST /api/agent/chat` with AgentResolver SPI (eidos implements).
3. **ImageResolver**: context-based mutually exclusive tiers (LOCAL/REGISTRY/FALLBACK) — not a fallback chain.
4. **Shared assertion contract**: `LlmResponse` + `LlmAssertions` migrate from topology-tests to testing/ module.
5. **Production networking**: tiered — NodePort for test/dev, Ingress for production (deferred).
6. **Sequencing**: #209 → platform#666 + eidos#208 → ops#208 → #210 → #211

## Plan Summary (6 tasks, 6 batches)

1. **ImageResolver Enhancement** (#209) — `ImageSource` enum, context-based tiers
2. **Shared Assertion Contract** (#208) — extract `LlmResponse` + `LlmAssertions` to testing/
3. **HTTP Fleet Exerciser** (#208) — `HttpFleetExerciser` + wire into HelloLlmMultiEnvTest *(blocked on platform#666)*
4. **K8S Target Consolidation** (#211) — move to deployment/, delete stub
5. **NodePort Networking** (#211) — replace port-forwarding, K3d port mapping
6. **CI Pipeline** (#210) — GitHub Actions dispatch workflow

## Key Files

- `specs/issue-207-multi-env-deploy-testing/2026-10-27-multi-env-deploy-testing-design.md` — design spec
- `specs/issue-207-multi-env-deploy-testing/decisions.md` — 6 captured decisions
- `plans/2026-10-27-multi-env-deploy-testing.md` — implementation plan
- `topology-tests/src/test/java/.../env/HelloLlmMultiEnvTest.java` — unified multi-env test
- `topology-tests/src/test/java/.../integration/FleetLlmExerciser.java` — in-JVM exerciser
- `testing/src/main/java/.../env/ImageResolver.java` — image resolution
