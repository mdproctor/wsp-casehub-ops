# Fleet Assertions — Effect-Based Test Verification

**Branch:** issue-212-simplify-llm-assertions
**Issues:** #212 (simplify LlmAssertions), #213 (use-case-level state observation)
**Scope:** #212 cleanup + #213 level 01 only. Subsequent levels are separate issues.

## Problem

`LlmAssertions` checks natural language quality — keyword regex matching, minimum
response length, content differentiation. These are non-deterministic, fragile across
model versions, and don't verify that the system did anything meaningful. The structured
response metadata (`backendKey`, `instanceId`, routing info) already carries the real
verification signal.

10 test classes across topology-tests use `LlmAssertions`. Every test follows the same
pattern: deploy topology → reconcile → audit assertions on provisioning → exercise LLM →
`assertDomainRelevant(response)`. The infrastructure assertions (provisioning, audit trail)
are solid. The LLM assertion is the weak link.

## Design

### Phase 1: Replace LlmAssertions with FleetAssertions (#212)

Delete `LlmAssertions` from `testing/`. Create `FleetAssertions` in its place.

**What moves:**
- `assertRouteConsistent(List<LlmResponse>)` — the one structural assertion. Verifies
  all fleet members route through the same backend. Moves verbatim.

**What's added:**
- `assertResponseWellFormed(LlmResponse)` — checks `backendKey` non-null, `instanceId`
  non-null, `content` non-blank. Deterministic proof that the agent responded through
  expected infrastructure.

**What's deleted:**
- `assertSubstantive` — prose length check on non-deterministic output
- `assertDomainRelevant` — keyword regex on natural language
- `assertDifferentiated` — checks distinct prose across fleet members
- `assertFleetOperational` — composite of the above
- `domainKeywords` — the regex map backing `assertDomainRelevant`

**Test migration:**
- `LlmAssertionsTest` → `FleetAssertionsTest` — tests for the two surviving/new methods
- All 10 topology test classes: update imports, replace `assertDomainRelevant` /
  `assertSubstantive` calls with `assertResponseWellFormed`, keep `assertRouteConsistent`
  where used

### Phase 2: Level 01 effect-based assertions (#213)

Level 01 (hello-llm) proves: "deploy an agent fleet and talk to it."

The effect to verify is that the agent is reachable, routes correctly, and returns a
well-formed structured response. At this level there are no deeper side effects (no
cases created, no work dispatched). The assertions are:

1. `FleetAssertions.assertResponseWellFormed(response)` — structured fields present
2. `FleetAssertions.assertRouteConsistent(responses)` — consistent backend routing

This replaces `assertDomainRelevant` in `HelloLlmIntegrationTest` and
`HelloLlmMultiEnvTest`.

### What this branch does NOT do

- Migrate levels 02+ — each level gets its own issue/branch
- Design a generic assertion framework — abstractions emerge from concrete level
  implementations, not upfront design
- Change exerciser classes (`HttpFleetExerciser`, `FleetLlmExerciser`) — invocation
  plumbing is unchanged
- Modify `LlmResponse` — the record is already structural and stays as-is

### Iterative approach for subsequent levels

Each level (02-accountability, 03-case-management, etc.) will:
1. Identify what "the system works" means for that use case
2. Determine what state to observe after exercising the agent
3. Add concrete assertion methods to `FleetAssertions` (or a level-specific utility
   if the assertion doesn't generalize)
4. Refactor and reconsolidate shared patterns as they emerge

## Files Changed

| File | Change |
|------|--------|
| `testing/.../LlmAssertions.java` | Delete |
| `testing/.../LlmAssertionsTest.java` | Delete |
| `testing/.../FleetAssertions.java` | Create — `assertRouteConsistent` + `assertResponseWellFormed` |
| `testing/.../FleetAssertionsTest.java` | Create — tests for both methods |
| `topology-tests/.../HelloLlmIntegrationTest.java` | Replace `assertDomainRelevant` → `assertResponseWellFormed` |
| `topology-tests/.../HelloLlmMultiEnvTest.java` | Replace `assertDomainRelevant` → `assertResponseWellFormed` |
| 8 other topology test classes | Replace `assertDomainRelevant`/`assertSubstantive`/`assertFleetOperational` → `assertResponseWellFormed` (+ `assertRouteConsistent` where `assertFleetOperational` was used) |

## References

- `testing/src/main/java/io/casehub/ops/testing/env/LlmAssertions.java` — current implementation
- `testing/src/main/java/io/casehub/ops/testing/env/LlmResponse.java` — response record (unchanged)
- `testing/src/main/java/io/casehub/ops/testing/env/HttpFleetExerciser.java` — HTTP exerciser
- `topology-tests/.../FleetLlmExerciser.java` — in-JVM exerciser
- Issue #207 — multi-env testing epic (parent context)
- Issue #208 — HTTP exerciser (invocation plumbing, landed)
- `decisions.md` — D1 (delete LlmAssertions), D2 (scope to level 01), D3 (assertResponseWellFormed)
