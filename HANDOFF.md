# Handoff — casehub-ops

## Active Work

**Branch:** `issue-194-acceptance-test-retrofit`
**Issues:** #194, #195, #196, #197, #198, #199 (epic #200)
**State:** Branch created, plan written, ready to execute.

## What to Do

Execute the plan at `wksp/plans/2026-10-26-acceptance-test-retrofit.md`.

Run `work continue` to pick up. The .plan has 6 issues queued (#194 active, #195-199 queued) with 7 tasks across 4 batches.

## Context

**The pattern:** All six tests follow the established `FleetLlmExerciser` pattern used by L01, L04, L05, L06, L07. Deploy topology via claudony Testcontainers, exercise deployed LLMs, assert level-specific outcomes.

**Tutorial output:** Tests must produce human-readable console output. These are learning content — a user running the test should see a narrative of what's happening (topology deployed, fleet exercised, capability verified), not just PASS/FAIL.

**Key reference files:**
- `HelloLlmIntegrationTest.java` — canonical example (simplest)
- `FleetScalingIntegrationTest.java` — fleet exercise pattern
- `TopologyMutationsIntegrationTest.java` — adaptation/situation pattern (needed for L12)
- `FleetLlmExerciser.java` — the shared exercise utility

**Complexity gradient:** L02/L03 are straightforward (deploy + exercise + assert). L08/L10 add fleet health and pool distribution. L12 involves RAS situation injection and adaptation rules. L14 requires fault injection and recovery verification.

**Read the topology YAML first** for each level before writing the test — it tells you what nodes exist and what the level claims to prove.

## Recently Completed

**Branch `issue-187-channel-patterns-topology`** landed on main (2026-10-25):
- #187: Channel protocol support (protocols, protocolParticipants, policyOverrides on ChannelNodeSpec)
- #188: Level 15 governed interaction topology + compilation tests
- Added `@NodeTypeId` to AgentNodeSpec and ChannelNodeSpec for graph-based YAML
- Code review caught policyOverrides merge-vs-replace semantics bug — fixed with `computePolicyOverrideDiff`
