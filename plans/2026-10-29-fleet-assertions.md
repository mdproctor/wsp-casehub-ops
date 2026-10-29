# Fleet Assertions Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #212 — refactor: simplify LlmAssertions to verify effects, not prose
**Issue group:** #212, #213

**Goal:** Replace prose-checking `LlmAssertions` with effect-based `FleetAssertions`, migrating level 01 tests as the first concrete implementation.

**Architecture:** Delete `LlmAssertions` (5 methods checking natural language output). Create `FleetAssertions` with two structural methods: `assertRouteConsistent` (moved verbatim) and `assertResponseWellFormed` (new — checks structured response fields). Update all 10 topology test classes to use the new class.

**Tech Stack:** Java 21, AssertJ, JUnit 5

## Global Constraints

- `FleetAssertions` lives in `testing/src/main/java/io/casehub/ops/testing/env/` (same package as `LlmAssertions`)
- `LlmResponse` record is unchanged
- No changes to exerciser classes (`HttpFleetExerciser`, `FleetLlmExerciser`)
- Build must pass: `mvn --batch-mode install`

---

## Batch 1: Create FleetAssertions and migrate all tests

### Task 1: Create FleetAssertions with tests

**Files:**
- Create: `testing/src/main/java/io/casehub/ops/testing/env/FleetAssertions.java`
- Create: `testing/src/test/java/io/casehub/ops/testing/env/FleetAssertionsTest.java`

**Interfaces:**
- Consumes: `LlmResponse(appId, role, domain, content, backendKey, instanceId)`
- Produces:
  - `FleetAssertions.assertResponseWellFormed(LlmResponse)` — asserts `backendKey` non-null, `instanceId` non-null, `content` non-blank
  - `FleetAssertions.assertRouteConsistent(List<LlmResponse>)` — asserts all responses share the same `backendKey`

- [ ] **Step 1: Write the failing test for assertResponseWellFormed**

```java
package io.casehub.ops.testing.env;

import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThatCode;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class FleetAssertionsTest {

    @Test
    void assertResponseWellFormed_passesForValidResponse() {
        var response = new LlmResponse("app-1", "analyst", "analysis",
                "Some substantive content here", "claude-3", "inst-1");
        assertThatCode(() -> FleetAssertions.assertResponseWellFormed(response))
                .doesNotThrowAnyException();
    }

    @Test
    void assertResponseWellFormed_failsForNullBackendKey() {
        var response = new LlmResponse("app-1", "analyst", "analysis",
                "Some content", null, "inst-1");
        assertThatThrownBy(() -> FleetAssertions.assertResponseWellFormed(response))
                .isInstanceOf(AssertionError.class);
    }

    @Test
    void assertResponseWellFormed_failsForNullInstanceId() {
        var response = new LlmResponse("app-1", "analyst", "analysis",
                "Some content", "claude-3", null);
        assertThatThrownBy(() -> FleetAssertions.assertResponseWellFormed(response))
                .isInstanceOf(AssertionError.class);
    }

    @Test
    void assertResponseWellFormed_failsForBlankContent() {
        var response = new LlmResponse("app-1", "analyst", "analysis",
                "   ", "claude-3", "inst-1");
        assertThatThrownBy(() -> FleetAssertions.assertResponseWellFormed(response))
                .isInstanceOf(AssertionError.class);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode -o test -pl testing -Dtest=FleetAssertionsTest`
Expected: FAIL — `FleetAssertions` does not exist

- [ ] **Step 3: Write FleetAssertions with assertResponseWellFormed**

```java
package io.casehub.ops.testing.env;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

public final class FleetAssertions {

    private FleetAssertions() {}

    public static void assertResponseWellFormed(LlmResponse response) {
        assertThat(response.backendKey())
                .as("LLM %s must have a backend key", response.appId())
                .isNotNull();
        assertThat(response.instanceId())
                .as("LLM %s must have an instance ID", response.appId())
                .isNotNull();
        assertThat(response.content())
                .as("LLM %s must produce non-blank content", response.appId())
                .isNotBlank();
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode -o test -pl testing -Dtest=FleetAssertionsTest`
Expected: PASS (4 tests)

- [ ] **Step 5: Add assertRouteConsistent tests**

Append to `FleetAssertionsTest`:

```java
    @Test
    void assertRouteConsistent_passesForSameBackend() {
        var r1 = new LlmResponse("app-1", "analyst", "analysis",
                "Content 1", "claude-3", "inst-1");
        var r2 = new LlmResponse("app-2", "coder", "software-engineering",
                "Content 2", "claude-3", "inst-2");
        assertThatCode(() -> FleetAssertions.assertRouteConsistent(List.of(r1, r2)))
                .doesNotThrowAnyException();
    }

    @Test
    void assertRouteConsistent_failsForDifferentBackends() {
        var r1 = new LlmResponse("app-1", "analyst", "analysis",
                "Content 1", "claude-3", "inst-1");
        var r2 = new LlmResponse("app-2", "coder", "software-engineering",
                "Content 2", "openai-4", "inst-2");
        assertThatThrownBy(() -> FleetAssertions.assertRouteConsistent(List.of(r1, r2)))
                .isInstanceOf(AssertionError.class);
    }
```

- [ ] **Step 6: Add assertRouteConsistent to FleetAssertions**

Append to `FleetAssertions`:

```java
    public static void assertRouteConsistent(List<LlmResponse> responses) {
        long distinctBackends = responses.stream()
                .map(LlmResponse::backendKey)
                .distinct()
                .count();
        assertThat(distinctBackends)
                .as("All fleet members should route through the same backend — got: %s",
                        responses.stream().map(r -> r.appId() + "→" + r.backendKey()).toList())
                .isEqualTo(1);
    }
```

- [ ] **Step 7: Run all FleetAssertions tests**

Run: `mvn --batch-mode -o test -pl testing -Dtest=FleetAssertionsTest`
Expected: PASS (6 tests)

- [ ] **Step 8: Commit**

```bash
git add testing/src/main/java/io/casehub/ops/testing/env/FleetAssertions.java testing/src/test/java/io/casehub/ops/testing/env/FleetAssertionsTest.java
git commit -m "feat(#212): add FleetAssertions with assertResponseWellFormed and assertRouteConsistent"
```

### Task 2: Delete LlmAssertions and migrate all test imports

**Files:**
- Delete: `testing/src/main/java/io/casehub/ops/testing/env/LlmAssertions.java` (use `ide_refactor_safe_delete`)
- Delete: `testing/src/test/java/io/casehub/ops/testing/env/LlmAssertionsTest.java` (use `ide_refactor_safe_delete`)
- Modify: `topology-tests/src/test/java/io/casehub/ops/topology/integration/HelloLlmIntegrationTest.java`
- Modify: `topology-tests/src/test/java/io/casehub/ops/topology/env/HelloLlmMultiEnvTest.java`
- Modify: `topology-tests/src/test/java/io/casehub/ops/topology/integration/AccountabilityAcceptanceTest.java`
- Modify: `topology-tests/src/test/java/io/casehub/ops/topology/integration/CaseManagementAcceptanceTest.java`
- Modify: `topology-tests/src/test/java/io/casehub/ops/topology/integration/ConfigDriftIntegrationTest.java`
- Modify: `topology-tests/src/test/java/io/casehub/ops/topology/integration/FleetObservabilityAcceptanceTest.java`
- Modify: `topology-tests/src/test/java/io/casehub/ops/topology/integration/FleetScalingIntegrationTest.java`
- Modify: `topology-tests/src/test/java/io/casehub/ops/topology/integration/LifecycleIntegrationTest.java`
- Modify: `topology-tests/src/test/java/io/casehub/ops/topology/integration/TopologyMutationsIntegrationTest.java`
- Modify: `topology-tests/src/test/java/io/casehub/ops/topology/integration/AdHocDeployerAcceptanceTest.java`

**Interfaces:**
- Consumes: `FleetAssertions.assertResponseWellFormed(LlmResponse)`, `FleetAssertions.assertRouteConsistent(List<LlmResponse>)` from Task 1

**Migration rules for each test file:**

| Old call | New call |
|----------|----------|
| `LlmAssertions.assertDomainRelevant(response)` | `FleetAssertions.assertResponseWellFormed(response)` |
| `LlmAssertions.assertSubstantive(response)` | `FleetAssertions.assertResponseWellFormed(response)` |
| `LlmAssertions.assertDifferentiated(responses)` | Remove (no structural replacement at this level) |
| `LlmAssertions.assertRouteConsistent(responses)` | `FleetAssertions.assertRouteConsistent(responses)` |
| `LlmAssertions.assertFleetOperational(responses)` | Replace with: `responses.forEach(FleetAssertions::assertResponseWellFormed);` then `FleetAssertions.assertRouteConsistent(responses);` |
| `import ...LlmAssertions` | `import ...FleetAssertions` |

- [ ] **Step 1: Migrate HelloLlmIntegrationTest (level 01)**

In `HelloLlmIntegrationTest.java`:
- Replace `import io.casehub.ops.testing.env.LlmAssertions;` with `import io.casehub.ops.testing.env.FleetAssertions;`
- Line 74: Replace `LlmAssertions.assertDomainRelevant(llmResponse);` with `FleetAssertions.assertResponseWellFormed(llmResponse);`
- Line 115: Replace `LlmAssertions.assertDomainRelevant(llmResponse);` with `FleetAssertions.assertResponseWellFormed(llmResponse);`

- [ ] **Step 2: Migrate HelloLlmMultiEnvTest (level 01 multi-env)**

In `HelloLlmMultiEnvTest.java`:
- Replace `import io.casehub.ops.testing.env.LlmAssertions;` with `import io.casehub.ops.testing.env.FleetAssertions;`
- Line 112: Replace `LlmAssertions.assertDomainRelevant(llmResponse);` with `FleetAssertions.assertResponseWellFormed(llmResponse);`

- [ ] **Step 3: Migrate AccountabilityAcceptanceTest**

- Replace import
- Line 112: `LlmAssertions.assertDomainRelevant(response);` → `FleetAssertions.assertResponseWellFormed(response);`

- [ ] **Step 4: Migrate CaseManagementAcceptanceTest**

- Replace import
- Line 103: `LlmAssertions.assertDomainRelevant(response);` → `FleetAssertions.assertResponseWellFormed(response);`

- [ ] **Step 5: Migrate ConfigDriftIntegrationTest**

- Replace import
- Line 82: `LlmAssertions.assertDomainRelevant(analystBeforeLlm);` → `FleetAssertions.assertResponseWellFormed(analystBeforeLlm);`
- Line 115: `LlmAssertions.assertDomainRelevant(analystAfterLlm);` → `FleetAssertions.assertResponseWellFormed(analystAfterLlm);`
- Line 121: `LlmAssertions.assertFleetOperational(allResponses);` → `allResponses.forEach(FleetAssertions::assertResponseWellFormed);` then `FleetAssertions.assertRouteConsistent(allResponses);`
- Line 163: `LlmAssertions.assertSubstantive(beforeDrift);` → `FleetAssertions.assertResponseWellFormed(beforeDrift);`
- Line 164: `LlmAssertions.assertSubstantive(afterDrift);` → `FleetAssertions.assertResponseWellFormed(afterDrift);`

- [ ] **Step 6: Migrate FleetObservabilityAcceptanceTest**

- Replace import
- Line 119: `LlmAssertions.assertDomainRelevant(response);` → `FleetAssertions.assertResponseWellFormed(response);`
- Line 124: `LlmAssertions.assertDifferentiated(responses);` → Remove this line (no structural replacement)

- [ ] **Step 7: Migrate FleetScalingIntegrationTest**

- Replace import
- Line 77: `LlmAssertions.assertFleetOperational(responses);` → `responses.forEach(FleetAssertions::assertResponseWellFormed);` then `FleetAssertions.assertRouteConsistent(responses);`
- Line 100: `LlmAssertions.assertFleetOperational(responses);` → `responses.forEach(FleetAssertions::assertResponseWellFormed);` then `FleetAssertions.assertRouteConsistent(responses);`

- [ ] **Step 8: Migrate LifecycleIntegrationTest**

- Replace import
- Line 82: `LlmAssertions.assertFleetOperational(preSuspend);` → `preSuspend.forEach(FleetAssertions::assertResponseWellFormed);` then `FleetAssertions.assertRouteConsistent(preSuspend);`
- Line 113: `LlmAssertions.assertFleetOperational(postResume);` → `postResume.forEach(FleetAssertions::assertResponseWellFormed);` then `FleetAssertions.assertRouteConsistent(postResume);`
- Line 145: `LlmAssertions.assertDomainRelevant(coderResponse);` → `FleetAssertions.assertResponseWellFormed(coderResponse);`

- [ ] **Step 9: Migrate TopologyMutationsIntegrationTest**

- Replace import
- Line 144: `LlmAssertions.assertFleetOperational(postMutation);` → `postMutation.forEach(FleetAssertions::assertResponseWellFormed);` then `FleetAssertions.assertRouteConsistent(postMutation);`
- Line 175: `LlmAssertions.assertDomainRelevant(researcherLlm);` → `FleetAssertions.assertResponseWellFormed(researcherLlm);`

- [ ] **Step 10: Migrate AdHocDeployerAcceptanceTest**

- Replace import
- Line 192: `LlmAssertions.assertSubstantive(response);` → `FleetAssertions.assertResponseWellFormed(response);`

- [ ] **Step 11: Delete LlmAssertions and LlmAssertionsTest**

Use `ide_refactor_safe_delete` on `LlmAssertions.java` — should have zero remaining references after migration.
Use `ide_refactor_safe_delete` on `LlmAssertionsTest.java`.

If IntelliJ is unavailable, verify no references remain with `grep -r "LlmAssertions" topology-tests/ testing/` then delete with `git rm`.

- [ ] **Step 12: Build and verify**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS — all modules compile, all unit tests pass

- [ ] **Step 13: Commit**

```bash
git add -A
git commit -m "refactor(#212): replace LlmAssertions with FleetAssertions across all topology tests"
```

## References

- [2026-10-29-fleet-assertions-design.md] — design spec this plan implements
- [testing/src/main/java/io/casehub/ops/testing/env/LlmAssertions.java] — class being replaced
- [testing/src/main/java/io/casehub/ops/testing/env/LlmResponse.java] — response record (unchanged)
- [testing/src/main/java/io/casehub/ops/testing/env/HttpFleetExerciser.java] — HTTP exerciser (unchanged)
- [topology-tests/src/test/java/io/casehub/ops/topology/integration/FleetLlmExerciser.java] — in-JVM exerciser (unchanged)
- [GitHub #212] — refactor: simplify LlmAssertions to verify effects, not prose
- [GitHub #213] — feat: use-case-level state observation assertions for multi-env topology tests
- [GitHub #207] — multi-env testing epic
