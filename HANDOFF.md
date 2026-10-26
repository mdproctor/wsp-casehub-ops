# Handoff — casehub-ops

## Active Work

**Branch:** `issue-194-acceptance-test-retrofit`
**Issues:** #194, #195, #196, #197, #198, #199 (epic #200)
**State:** All 6 acceptance tests written and compiling. Full build passes. Ready for `work end`.

## What Was Done

All 7 tasks across 4 batches completed:

| Task | Issue | Test class | Status |
|------|-------|-----------|--------|
| 1 | #194 | `AccountabilityAcceptanceTest` | Done — L02 deploy + exercise + per-node audit |
| 2 | #195 | `CaseManagementAcceptanceTest` | Done — L03 deploy + qhorus case infra + exercise |
| 3 | #196 | `FleetObservabilityAcceptanceTest` | Done — L08 deploy + exercise 3 LLMs + fault/recovery |
| 4 | #197 | `PoolDeploymentAcceptanceTest` | Done — L10 deploy + pool config + scaling |
| 5 | #198 | `AdHocDeployerAcceptanceTest` | Done — L12 adaptation rule + specialist lifecycle |
| 6 | #199 | `FaultRecoveryAcceptanceTest` | Done — L14 fault inject + recovery + continuity |
| 7 | — | Full build verification | Done — `mvn install` passes, 96 tests green |

## What to Do Next

Run `work end` to close the branch. All tests compile and the build is green. Tests are `@Tag("integration-live")` and excluded from the default `mvn test` — they require Docker + claudony via `-Pintegration-live`.

## Context

Each acceptance test follows the established pattern:
- Deploy topology via `ReconciliationLoop` with `AuditCapture`
- Exercise LLMs via `FleetLlmExerciser` (where applicable)
- Assert level-specific outcomes
- Tutorial-style `System.out` output for human readability

L08 and L14 use `FailableClaudonyProvisioner` for fault injection. L12 uses `AdaptationRuleCompiler` + `LlmTriggerEvaluator` with mock LLM for the adaptation lifecycle.

## Recently Completed

**Branch `issue-187-channel-patterns-topology`** landed on main (2026-10-25):
- #187: Channel protocol support
- #188: Level 15 governed interaction topology + compilation tests
