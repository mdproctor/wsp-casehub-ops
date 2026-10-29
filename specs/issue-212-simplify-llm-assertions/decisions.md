## D1: Disposition of LlmAssertions class

**Choice:** Delete `LlmAssertions` entirely. Move `assertRouteConsistent` to a new `FleetAssertions` class. Remove all prose-checking methods (`assertSubstantive`, `assertDomainRelevant`, `assertDifferentiated`, `assertFleetOperational`).
**Alternatives:**
- Gut and rename in place — keeps git history but no practical benefit, same import churn
- Gut but keep the name — least churn but misleading name for what survives
**Rationale:** The class name reflects prose-checking, which is being removed. Every test file needs import updates anyway (removing prose assertion calls), so changing the import to a correct name costs nothing extra. `FleetAssertions` provides a natural home for effect-based assertions as they emerge level-by-level.
**Trade-offs:** Slightly more files touched than option C, but the name accuracy pays for itself immediately.
**Sources:** LlmAssertions.java, 10 test classes with ~20 call sites, issue #212 body
**Exploration:** quick
**Status:** captured

## D2: Scope of level-by-level migration in this branch

**Choice:** This branch covers level 01 (hello-llm) only. Subsequent levels are separate issues/branches.
**Alternatives:**
- Levels 01-03 in this branch — more coverage but risks mid-branch abstraction rework
- All levels in this branch — comprehensive but large change surface across 10 test classes
**Rationale:** Level 01 establishes the pattern and validates the approach. Each subsequent level is a focused diff. If level 02 forces a refactor, it's visible as its own change, not buried in a mega-branch.
**Trade-offs:** More branches/issues total, but each is small and reviewable. Slower overall throughput.
**Sources:** topology-tests directory listing, issue #213 body
**Exploration:** quick
**Status:** captured

## D3: Level 01 assertion replacement

**Choice:** Add `assertResponseWellFormed` to `FleetAssertions`. Checks structured fields: `backendKey` and `instanceId` non-null, `content` non-blank. Deterministic, proves the agent responded through expected infrastructure.
**Alternatives:**
- Remove assertion entirely, rely on exerciser not throwing — technically sufficient but loses explicit routing metadata verification
**Rationale:** Level 01 proves "deploy and talk to an agent." The meaningful signal is structured response fields, not prose quality. `assertResponseWellFormed` + `assertRouteConsistent` covers everything level 01 needs to prove.
**Trade-offs:** Marginally more code than option B, but makes the test's intent explicit.
**Depends on:** D1 (FleetAssertions as the home for these methods)
**Sources:** HelloLlmIntegrationTest.java, HelloLlmMultiEnvTest.java, LlmResponse.java
**Exploration:** quick
**Status:** captured
