# fsitrading Adaptive Ops Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #25 — feat: fsitrading adaptive ops — first real consumer of deployment desired-state reconciliation
**Issue group:** #25

**Goal:** Full-stack adaptive topology management: market signals → RAS situation detection → topology adaptation → reconciliation, spanning casehub-ops (recompiler) and casehub-fsitrading (situations + deployment YAML).

**Architecture:** `DeploymentAdaptiveSituationRecompiler` implements the `SituationRecompiler` SPI from desiredstate-api. It receives `ActiveSituation` pushes from the runtime, tracks all active situations per tenant, and recompiles from the base `DeploymentGoals` every time — applying YAML-defined adaptation rules (scale, add, update) for all currently active situations. On the fsitrading side, a RAS summarisation pipeline processes market events into phase-detect signals, and Ganglion detectors raise situations that the recompiler consumes.

**Tech Stack:** Java 21+, Quarkus 3.x, casehub-desiredstate-api 0.2-SNAPSHOT (SituationRecompiler SPI), casehub-ras-api 0.2-SNAPSHOT (ActiveSituation, SituationDefinitionProvider, GanglionDescriptor), Jackson (YAML/tree-merge), JUnit 5/Awaitility (tests)

## Global Constraints

- `SituationRecompiler` SPI exists in desiredstate-api — do not modify the interface (except the proposed `situationResolved()` default method, which is a separate cross-repo change)
- `AdaptationRuleSpec`, `AdaptationRule`, `DeploymentGoals.adaptations`, `AgentNodeSpec.withAgentId()` all exist — do not recreate
- `TenantAdaptationState` is package-private — keep it that way
- Follow `deployment-spec-tenant-scoped` protocol — `tenancyId` required on all deployment specs
- Follow `engine-spi-noops-defaultbean` protocol — `@DefaultBean` for SPI stubs
- Deployment YAML namespace: pipeline input = `io.casehub.fsitrading.market.*`, pipeline output = `io.casehub.fsitrading.situation.*`, security = `io.casehub.fsitrading.security.*`
- `ChainMode.Count` (not `ChainMode.Single` — does not exist), `TriggerAction.NotifyOnly` (not `Noop`), `TriggerMode.Repeating` (not `Continuous`)
- All commits reference `Refs #25`

---

## Batch 1: Ops — Adaptive Situation Recompiler

The core new component: `DeploymentAdaptiveSituationRecompiler` implementing `SituationRecompiler`, plus `TenantAdaptationState` modifications for situation tracking. After this batch, the recompiler is wired and testable in isolation.

### Task 1: Extend TenantAdaptationState with situation tracking

**Files:**
- Modify: `deployment/src/main/java/io/casehub/ops/deployment/adaptation/TenantAdaptationState.java`
- Test: `deployment/src/test/java/io/casehub/ops/deployment/adaptation/TenantAdaptationStateTest.java`

**Interfaces:**
- Consumes: `ActiveSituation` (from ras-api), `AdaptationRule` (existing)
- Produces: `updateSituation(ActiveSituation)`, `activeSituationFor(AdaptationRule) → Optional<ActiveSituation>`, `clearSituation(String) → boolean`, `clearAbsentSituations()` (replaces `clearAbsentSituations(Set<String>)`)

- [ ] **Step 1: Write failing test for `updateSituation` and `activeSituationFor`**

```java
@Test
void updateSituationTracksAndRetrievesBySituationId() {
    var state = new TenantAdaptationState(goals, rules,
        Map.of("volatility-spike", Duration.ofMinutes(30)));
    var situation = new ActiveSituation("volatility-spike", "key1", "t1",
        0.85, Map.of(), Instant.now(), Instant.now(), 1);

    state.updateSituation(situation);

    Optional<ActiveSituation> found = state.activeSituationFor(scaleRule);
    assertThat(found).isPresent();
    assertThat(found.get().situationId()).isEqualTo("volatility-spike");
    assertThat(found.get().confidence()).isEqualTo(0.85);
}
```

- [ ] **Step 2: Run test — verify it fails (method does not exist)**

Run: `mvn --batch-mode -o test -pl deployment -Dtest=TenantAdaptationStateTest#updateSituationTracksAndRetrievesBySituationId`
Expected: compilation error

- [ ] **Step 3: Add new fields and constructor parameter**

Add to `TenantAdaptationState`:

```java
private final Map<String, ActiveSituation> trackedSituations = new HashMap<>();
private final Map<String, Duration> situationClearanceWindows;

TenantAdaptationState(DeploymentGoals goals, List<AdaptationRule> rules,
                      Map<String, Duration> situationClearanceWindows) {
    this.goals = Objects.requireNonNull(goals, "goals");
    this.rules = List.copyOf(Objects.requireNonNull(rules, "rules"));
    this.situationClearanceWindows = Map.copyOf(
        Objects.requireNonNull(situationClearanceWindows, "situationClearanceWindows"));
}
```

Add methods:

```java
void updateSituation(ActiveSituation situation) {
    trackedSituations.put(situation.situationId(), situation);
}

Optional<ActiveSituation> activeSituationFor(AdaptationRule rule) {
    return Optional.ofNullable(
        trackedSituations.get(rule.trigger().situation()));
}
```

- [ ] **Step 4: Run test — verify it passes**

Run: `mvn --batch-mode -o test -pl deployment -Dtest=TenantAdaptationStateTest#updateSituationTracksAndRetrievesBySituationId`
Expected: PASS

- [ ] **Step 5: Write failing test for `clearSituation`**

```java
@Test
void clearSituationRemovesTrackedSituationAndResetsRuleState() {
    var state = new TenantAdaptationState(goals, rules,
        Map.of("volatility-spike", Duration.ofMinutes(30)));
    var situation = new ActiveSituation("volatility-spike", "key1", "t1",
        0.85, Map.of(), Instant.now(), Instant.now(), 1);

    state.updateSituation(situation);
    state.shouldActivate(scaleRule, situation);

    boolean cleared = state.clearSituation("volatility-spike");
    assertThat(cleared).isTrue();
    assertThat(state.activeSituationFor(scaleRule)).isEmpty();

    boolean clearedAgain = state.clearSituation("volatility-spike");
    assertThat(clearedAgain).isFalse();
}
```

- [ ] **Step 6: Implement `clearSituation`**

```java
boolean clearSituation(String situationId) {
    ActiveSituation removed = trackedSituations.remove(situationId);
    if (removed == null) return false;
    for (AdaptationRule rule : rules) {
        if (rule.trigger().situation().equals(situationId)) {
            activePerRule.put(rule.name(), false);
            lastChangePerRule.put(rule.name(), Instant.now());
        }
    }
    return true;
}
```

- [ ] **Step 7: Run test — verify it passes**

- [ ] **Step 8: Write failing test for timestamp-based `clearAbsentSituations()`**

```java
@Test
void clearAbsentSituationsRemovesStaleSituations() {
    var state = new TenantAdaptationState(goals, rules,
        Map.of("volatility-spike", Duration.ofSeconds(1)));
    var stale = new ActiveSituation("volatility-spike", "key1", "t1",
        0.85, Map.of(), Instant.now().minusSeconds(10),
        Instant.now().minusSeconds(5), 1);

    state.updateSituation(stale);
    state.shouldActivate(scaleRule, stale);

    state.clearAbsentSituations();
    assertThat(state.activeSituationFor(scaleRule)).isEmpty();
}
```

- [ ] **Step 9: Replace `clearAbsentSituations(Set<String>)` with timestamp-based version**

```java
void clearAbsentSituations() {
    Instant now = Instant.now();
    var iter = trackedSituations.entrySet().iterator();
    while (iter.hasNext()) {
        var entry = iter.next();
        String sitId = entry.getKey();
        ActiveSituation sit = entry.getValue();
        Duration clearanceWindow = situationClearanceWindows.getOrDefault(
            sitId, Duration.ofMinutes(5));
        if (Duration.between(sit.lastSignal(), now).compareTo(clearanceWindow) > 0) {
            for (AdaptationRule rule : rules) {
                if (rule.trigger().situation().equals(sitId)) {
                    Boolean wasActive = activePerRule.get(rule.name());
                    if (wasActive != null && wasActive) {
                        Duration cooldown = rule.trigger().effectiveCooldown();
                        if (!cooldown.isZero()) {
                            Instant lastChange = lastChangePerRule.get(rule.name());
                            if (lastChange != null &&
                                Duration.between(lastChange, now).compareTo(cooldown) < 0) {
                                continue;
                            }
                        }
                        activePerRule.put(rule.name(), false);
                        lastChangePerRule.put(rule.name(), now);
                    }
                }
            }
            iter.remove();
        }
    }
}
```

- [ ] **Step 10: Run all TenantAdaptationState tests — verify pass**

Run: `mvn --batch-mode -o test -pl deployment -Dtest=TenantAdaptationStateTest`
Expected: PASS

- [ ] **Step 11: Fix callers of old `clearAbsentSituations(Set<String>)` signature**

`AdaptiveTopologyManager.compileAdapted()` calls `state.clearAbsentSituations(activeSituationIds)` — update to `state.clearAbsentSituations()`. Also update the old 2-arg constructor call sites to pass `Map.of()` for `situationClearanceWindows`.

- [ ] **Step 12: Run full deployment module tests**

Run: `mvn --batch-mode -o test -pl deployment`
Expected: PASS

- [ ] **Step 13: Commit**

```bash
git add deployment/
git commit -m "feat(#25): extend TenantAdaptationState with situation tracking

Add updateSituation(), activeSituationFor(), clearSituation(),
replace clearAbsentSituations(Set) with timestamp-based clearing.

Refs #25

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 2: Implement DeploymentAdaptiveSituationRecompiler

**Files:**
- Create: `deployment/src/main/java/io/casehub/ops/deployment/adaptation/DeploymentAdaptiveSituationRecompiler.java`
- Test: `deployment/src/test/java/io/casehub/ops/deployment/adaptation/DeploymentAdaptiveSituationRecompilerTest.java`

**Interfaces:**
- Consumes: `SituationRecompiler` (desiredstate-api), `TenantAdaptationState`, `AdaptationRule`, `DeploymentGoalCompiler`, `ActiveSituation`
- Produces: `register(String tenancyId, DeploymentGoals goals, Map<String, Duration> situationClearanceWindows)`, `recompile(...)` → `Optional<CompilationResult>`

- [ ] **Step 1: Write failing test — recompile returns empty for unregistered tenant**

```java
@Test
void recompileReturnsEmptyForUnregisteredTenant() {
    var recompiler = new DeploymentAdaptiveSituationRecompiler();
    recompiler.compiler = mockCompiler;
    recompiler.mapper = mapper;

    var result = recompiler.recompile("unknown", graph, actualState,
        situation, graphFactory);
    assertThat(result).isEmpty();
}
```

- [ ] **Step 2: Run test — verify compile error**

- [ ] **Step 3: Implement `DeploymentAdaptiveSituationRecompiler`**

Create the class implementing `SituationRecompiler` per the spec's §Component 2 code. Key features:
- `@ApplicationScoped`, `priority() = 100`
- `ConcurrentHashMap<String, TenantAdaptationState>` per-tenant state
- `register()` method for bootstrap
- `recompile()` — updates tracked situation, recompiles from base with all active situations, structural graph comparison
- Note: `situationResolved()` is deferred — requires cross-repo `casehub-desiredstate-api` change first (new default method). Implement after that lands.
- `graphsEqual()` — compares `nodes()` and `dependencies()` maps

- [ ] **Step 4: Run test — verify pass**

- [ ] **Step 5: Write test — single situation triggers scale adaptation**

```java
@Test
void recompileScalesRiskAgentOnVolatilitySpike() {
    recompiler.register("t1", goalsWithAdaptations,
        Map.of("volatility-spike", Duration.ofMinutes(30)));

    var situation = new ActiveSituation("volatility-spike", "k1", "t1",
        0.85, Map.of(), Instant.now(), Instant.now(), 1);

    var result = recompiler.recompile("t1", baseGraph, actualState,
        situation, graphFactory);

    assertThat(result).isPresent();
    var adapted = ((CompilationResult.Single) result.get()).graph();
    assertThat(adapted.nodes()).containsKey(NodeId.of("risk-agent~2"));
    assertThat(adapted.nodes()).containsKey(NodeId.of("risk-agent~3"));
}
```

- [ ] **Step 6: Run test — verify pass (implementation already handles this via AdaptationRule.apply)**

- [ ] **Step 7: Write test — multiple situations compose correctly**

```java
@Test
void recompileComposesMultipleActiveSituations() {
    recompiler.register("t1", goalsWithAdaptations,
        Map.of("volatility-spike", Duration.ofMinutes(30),
               "market-anomaly", Duration.ofMinutes(15)));

    var volatility = new ActiveSituation("volatility-spike", "k1", "t1",
        0.85, Map.of(), Instant.now(), Instant.now(), 1);
    recompiler.recompile("t1", baseGraph, actualState, volatility, graphFactory);

    var anomaly = new ActiveSituation("market-anomaly", "k2", "t1",
        0.7, Map.of(), Instant.now(), Instant.now(), 1);
    var result = recompiler.recompile("t1", baseGraph, actualState,
        anomaly, graphFactory);

    assertThat(result).isPresent();
    var adapted = ((CompilationResult.Single) result.get()).graph();
    // Scale rule applied
    assertThat(adapted.nodes()).containsKey(NodeId.of("risk-agent~2"));
    // Update rule applied — trust threshold changed
    var trustNode = adapted.nodes().get(NodeId.of("trade-execution"));
    assertThat(trustNode).isNotNull();
}
```

- [ ] **Step 8: Run test — verify pass**

- [ ] **Step 9: Write test — identical graph returns empty (no unnecessary updates)**

```java
@Test
void recompileReturnsEmptyWhenGraphUnchanged() {
    recompiler.register("t1", goalsWithoutAdaptations, Map.of());

    var situation = new ActiveSituation("unrelated-situation", "k1", "t1",
        0.9, Map.of(), Instant.now(), Instant.now(), 1);
    var result = recompiler.recompile("t1", baseGraph, actualState,
        situation, graphFactory);

    assertThat(result).isEmpty();
}
```

- [ ] **Step 10: Run all recompiler tests**

Run: `mvn --batch-mode -o test -pl deployment -Dtest=DeploymentAdaptiveSituationRecompilerTest`
Expected: PASS

- [ ] **Step 11: Commit**

```bash
git add deployment/
git commit -m "feat(#25): implement DeploymentAdaptiveSituationRecompiler

Implements SituationRecompiler SPI at priority 100. Tracks active
situations per tenant, recompiles from base goals with all active
situations on each invocation. Structural graph comparison prevents
unnecessary updates.

Refs #25

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 3: Remove AdaptiveTopologyManager + StubSituationSource

**Files:**
- Delete: `deployment/src/main/java/io/casehub/ops/deployment/adaptation/AdaptiveTopologyManager.java`
- Delete: `deployment/src/test/java/io/casehub/ops/deployment/adaptation/AdaptiveTopologyManagerTest.java`
- Delete: `deployment/src/test/java/io/casehub/ops/deployment/adaptation/AdaptiveTopologyIntegrationTest.java`
- Delete: `app/src/main/java/io/casehub/ops/app/spi/StubSituationSource.java`
- Modify: any files that reference `AdaptiveTopologyManager` or `StubSituationSource`

**Interfaces:**
- Consumes: nothing — this is deletion
- Produces: clean compilation across deployment + app modules

- [ ] **Step 1: Find all references to `AdaptiveTopologyManager`**

Use `ide_find_references` on `AdaptiveTopologyManager` to identify all usages beyond the class itself and its tests.

- [ ] **Step 2: Find all references to `StubSituationSource`**

Use `ide_find_references` on `StubSituationSource` in `app/src/main/java/`.

- [ ] **Step 3: Update or remove each reference**

For each reference found:
- If it's a test using `AdaptiveTopologyManager` → delete the test or migrate to use `DeploymentAdaptiveSituationRecompiler`
- If it's app bootstrap code → update to use `DeploymentAdaptiveSituationRecompiler.register()`
- If it's documentation → update reference

- [ ] **Step 4: Delete the files**

Use `ide_refactor_safe_delete` for each file:
- `AdaptiveTopologyManager.java`
- `AdaptiveTopologyManagerTest.java`
- `AdaptiveTopologyIntegrationTest.java`
- `StubSituationSource.java`

- [ ] **Step 5: Build deployment + app modules**

Run: `mvn --batch-mode -o compile -pl deployment,app`
Expected: BUILD SUCCESS

- [ ] **Step 6: Run full test suite**

Run: `mvn --batch-mode -o test -pl deployment,app`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add deployment/ app/
git commit -m "refactor(#25): remove AdaptiveTopologyManager + StubSituationSource

Replaced by DeploymentAdaptiveSituationRecompiler which implements the
SituationRecompiler SPI directly. No CDI event observation, no periodic
polling — the runtime pushes situations to the recompiler.

Refs #25

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 2: fsitrading — Domain Types + Event Bridge

Standalone fsitrading domain types and the CloudEvent bridge from the existing MarketPulse pipeline. After this batch, fsitrading emits CloudEvents for market conditions and security alerts.

### Task 4: FsiTradingEventTypes + FsiTradingSecurityEvent + SecurityAlertBridge

**Files:**
- Create: `fsitrading/app/src/main/java/io/casehub/fsitrading/app/situation/FsiTradingEventTypes.java`
- Create: `fsitrading/api/src/main/java/io/casehub/fsitrading/model/FsiTradingSecurityEvent.java`
- Create: `fsitrading/app/src/main/java/io/casehub/fsitrading/app/situation/SecurityAlertBridge.java`
- Test: `fsitrading/app/src/test/java/io/casehub/fsitrading/app/situation/SecurityAlertBridgeTest.java`

**Interfaces:**
- Consumes: `FsiTradingSecurityEvent` (CDI event), `CloudEventEmitter` (from ras/platform)
- Produces: `FsiTradingEventTypes.SIGNAL`, `.CONDITION`, `.SECURITY` constants; `io.casehub.fsitrading.security.alert` CloudEvents on CRITICAL severity

- [ ] **Step 1: Create `FsiTradingEventTypes`**

```java
package io.casehub.fsitrading.app.situation;

public final class FsiTradingEventTypes {
    public static final String SIGNAL = "io.casehub.fsitrading.situation.signal";
    public static final String CONDITION = "io.casehub.fsitrading.situation.condition";
    public static final String SECURITY = "io.casehub.fsitrading.security.alert";
    private FsiTradingEventTypes() {}
}
```

- [ ] **Step 2: Create `FsiTradingSecurityEvent`**

```java
package io.casehub.fsitrading.model;

import java.time.Instant;

public record FsiTradingSecurityEvent(
    Severity severity,
    String source,
    String detail,
    Instant timestamp
) {
    public enum Severity { INFORMATIONAL, WARNING, CRITICAL }
}
```

- [ ] **Step 3: Write failing test for `SecurityAlertBridge`**

```java
@Test
void onlyCriticalEventsEmitCloudEvent() {
    var bridge = new SecurityAlertBridge();
    bridge.emitter = mockEmitter;

    bridge.onSecurityEvent(new FsiTradingSecurityEvent(
        FsiTradingSecurityEvent.Severity.WARNING,
        "test", "warning detail", Instant.now()));
    verify(mockEmitter, never()).emit(any());

    bridge.onSecurityEvent(new FsiTradingSecurityEvent(
        FsiTradingSecurityEvent.Severity.CRITICAL,
        "trust-violation", "agent exceeded threshold", Instant.now()));
    verify(mockEmitter).emit(argThat(ce ->
        ce.getType().equals(FsiTradingEventTypes.SECURITY)));
}
```

- [ ] **Step 4: Implement `SecurityAlertBridge`**

Per spec §Component 4 code — observes `FsiTradingSecurityEvent` via `@ObservesAsync`, emits CloudEvent with `category: "BREACH"` for CRITICAL severity.

- [ ] **Step 5: Run tests**

Run: `mvn --batch-mode -o test -pl fsitrading/app -Dtest=SecurityAlertBridgeTest`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/193/fsitrading add .
git -C /Users/mdproctor/claude/casehub/slots/193/fsitrading commit -m "feat(#25): add FsiTradingEventTypes, SecurityEvent, SecurityAlertBridge

Domain event types for market conditions and security alerts.
SecurityAlertBridge emits CloudEvents on CRITICAL severity.

Refs casehubio/casehub-ops#25

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 5: MarketConditionCloudEventPublisher

**Files:**
- Create: `fsitrading/app/src/main/java/io/casehub/fsitrading/app/situation/MarketConditionCloudEventPublisher.java`
- Test: `fsitrading/app/src/test/java/io/casehub/fsitrading/app/situation/MarketConditionCloudEventPublisherTest.java`

**Interfaces:**
- Consumes: `RegimeAssessment` (from MarketPulse L3 output via EventStreamBus), `CloudEventEmitter`
- Produces: `io.casehub.fsitrading.market.assessment` CloudEvents

- [ ] **Step 1: Write failing test**

```java
@Test
void publishesRegimeAssessmentAsCloudEvent() {
    var publisher = new MarketConditionCloudEventPublisher();
    publisher.emitter = mockEmitter;

    var assessment = new RegimeAssessment("BTCUSD",
        MarketRegime.HIGH_VOLATILITY, 0.85, "price spike",
        Instant.now());

    publisher.onRegimeAssessment(assessment);

    verify(mockEmitter).emit(argThat(ce ->
        ce.getType().equals("io.casehub.fsitrading.market.assessment")
        && ce.getSource().toString().equals("/fsitrading/market-pulse")));
}
```

- [ ] **Step 2: Implement `MarketConditionCloudEventPublisher`**

Per spec §Component 4 code — subscribes to MarketPulse L3 output, emits `io.casehub.fsitrading.market.assessment` CloudEvents.

- [ ] **Step 3: Run test — verify pass**

- [ ] **Step 4: Wire into `MarketPulseConfiguration`**

Add subscription to L3 bus output that calls `publisher.onRegimeAssessment()`. Use `@Inject` and `@PostConstruct` or `@Observes StartupEvent` to wire the subscription.

- [ ] **Step 5: Run full fsitrading tests**

Run: `mvn --batch-mode -o test -pl fsitrading/app`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/193/fsitrading add .
git -C /Users/mdproctor/claude/casehub/slots/193/fsitrading commit -m "feat(#25): add MarketConditionCloudEventPublisher

Bridges MarketPulse L3 RegimeAssessment output to RAS-consumable
CloudEvents under io.casehub.fsitrading.market.* namespace.

Refs casehubio/casehub-ops#25

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 3: fsitrading — RAS Pipeline + Situation Definitions

The summarisation pipeline YAML and Ganglion situation definitions. After this batch, fsitrading can detect market situations (volatility-spike, market-anomaly, active-breach) from CloudEvents.

### Task 6: Summarisation Pipeline YAML

**Files:**
- Create: `fsitrading/app/src/main/resources/META-INF/summarisation/fsitrading-market-monitoring.yaml`

**Interfaces:**
- Consumes: `io.casehub.fsitrading.market.*` CloudEvents (from MarketConditionCloudEventPublisher)
- Produces: `io.casehub.fsitrading.situation.signal` (L1) and `io.casehub.fsitrading.situation.condition` (L2) CloudEvents

- [ ] **Step 1: Create the pipeline YAML**

Per spec §Component 4 — full YAML with:
- L1 `threshold-classify`: price-spike, volume-surge, spread-widening, order-imbalance, normal-tick
- L2 `phase-detect`: STABLE → VOLATILE → ANOMALOUS with transitions including STABLE → ANOMALOUS direct path

- [ ] **Step 2: Write pipeline loading test**

```java
@Test
void pipelineYamlParsesCorrectly() {
    var yaml = new Yaml();
    var config = yaml.load(getClass().getResourceAsStream(
        "/META-INF/summarisation/fsitrading-market-monitoring.yaml"));
    assertThat(config).isNotNull();
    // Verify structure
    var pipeline = ((Map<String, Object>) config).get("pipeline");
    assertThat(pipeline).isNotNull();
}
```

- [ ] **Step 3: Run test — verify pass**

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/193/fsitrading add .
git -C /Users/mdproctor/claude/casehub/slots/193/fsitrading commit -m "feat(#25): add fsitrading market monitoring summarisation pipeline

L1 threshold-classify (5 rules: price-spike, volume-surge,
spread-widening, order-imbalance, normal-tick).
L2 phase-detect (STABLE/VOLATILE/ANOMALOUS with direct transitions).

Refs casehubio/casehub-ops#25

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 7: FsiTradingSituationDefinitionProvider

**Files:**
- Create: `fsitrading/app/src/main/java/io/casehub/fsitrading/app/situation/FsiTradingSituationDefinitionProvider.java`
- Test: `fsitrading/app/src/test/java/io/casehub/fsitrading/app/situation/FsiTradingSituationDefinitionProviderTest.java`

**Interfaces:**
- Consumes: `SituationDefinitionProvider` (ras-api), `GanglionDescriptor`, `SituationRegistration`, `SituationDefinition`, `ChainMode.Count`, `TriggerAction.NotifyOnly`, `TriggerAction.CreateCase`, `TriggerMode.Repeating`, `TriggerMode.FireOnce`
- Produces: 4 ganglia (`volatile-detected`, `anomalous-detected`, `stable-detected`, `breach-signal`) + 3 situation definitions (`volatility-spike`, `market-anomaly`, `active-breach`)

- [ ] **Step 1: Write failing test — correct number of ganglia and registrations**

```java
@Test
void providesCorrectGanglionAndRegistrationCounts() {
    var provider = new FsiTradingSituationDefinitionProvider();
    assertThat(provider.ganglionDescriptors()).hasSize(4);
    assertThat(provider.registrations()).hasSize(3);
}
```

- [ ] **Step 2: Implement `FsiTradingSituationDefinitionProvider`**

Per spec §Component 4 — full implementation with parameterised `ganglion()` helper, 3 registrations using `ChainMode.Count`, `TriggerAction.NotifyOnly`/`CreateCase`, `TriggerMode.Repeating`/`FireOnce`.

- [ ] **Step 3: Run test — verify pass**

- [ ] **Step 4: Write test — verify situation IDs match constants**

```java
@Test
void registrationSituationIdsMatchConstants() {
    var provider = new FsiTradingSituationDefinitionProvider();
    var ids = provider.registrations().stream()
        .map(r -> r.definition().situationId())
        .toList();
    assertThat(ids).containsExactlyInAnyOrder(
        FsiTradingSituationDefinitionProvider.VOLATILITY_SPIKE,
        FsiTradingSituationDefinitionProvider.MARKET_ANOMALY,
        FsiTradingSituationDefinitionProvider.ACTIVE_BREACH);
}
```

- [ ] **Step 5: Write test — verify event type filters are correct**

```java
@Test
void registrationEventTypesAreCorrect() {
    var provider = new FsiTradingSituationDefinitionProvider();
    var regs = provider.registrations();

    // volatility and anomaly use CONDITION events
    assertThat(regs.get(0).definition().eventTypes())
        .containsExactly(FsiTradingEventTypes.CONDITION);
    assertThat(regs.get(1).definition().eventTypes())
        .containsExactly(FsiTradingEventTypes.CONDITION);
    // breach uses SECURITY events
    assertThat(regs.get(2).definition().eventTypes())
        .containsExactly(FsiTradingEventTypes.SECURITY);
}
```

- [ ] **Step 6: Run all provider tests**

Run: `mvn --batch-mode -o test -pl fsitrading/app -Dtest=FsiTradingSituationDefinitionProviderTest`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/193/fsitrading add .
git -C /Users/mdproctor/claude/casehub/slots/193/fsitrading commit -m "feat(#25): add FsiTradingSituationDefinitionProvider

4 ganglia: volatile-detected, anomalous-detected, stable-detected,
breach-signal. 3 situations: volatility-spike (Count+NotifyOnly+Repeating),
market-anomaly (Count+NotifyOnly+Repeating), active-breach
(Count+CreateCase+FireOnce).

Refs casehubio/casehub-ops#25

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 4: fsitrading — Deployment YAML + Bootstrap + Integration

The deployment YAML that declares fsitrading's agent topology and the bootstrap bean that wires everything together. After this batch, fsitrading has a complete adaptive deployment topology.

### Task 8: Deployment YAML + Bootstrap Bean

**Files:**
- Create: `fsitrading/app/src/main/resources/casehub-deployment.yaml`
- Create: `fsitrading/app/src/main/java/io/casehub/fsitrading/app/situation/FsiTradingDeploymentBootstrap.java`
- Test: `fsitrading/app/src/test/java/io/casehub/fsitrading/app/situation/FsiTradingDeploymentBootstrapTest.java`

**Interfaces:**
- Consumes: `DeploymentGoalLoader` (ops), `DeploymentAdaptiveSituationRecompiler.register()`, `LifecycleManager.start()`, `FsiTradingSituationDefinitionProvider.registrations()` (for clearance windows)
- Produces: Bootstrapped deployment topology at startup

- [ ] **Step 1: Create `casehub-deployment.yaml`**

Per spec §Component 3 — full YAML with 4 agents (strategy-agent, strategy-agent-2, risk-agent, audit-agent), 3 channels, 2 case types, 2 trust policies, 3 adaptation rules.

- [ ] **Step 2: Write test — YAML parses into DeploymentGoals**

```java
@Test
void deploymentYamlParsesIntoDeploymentGoals() throws Exception {
    var loader = new DeploymentGoalLoader(mapper);
    var goals = loader.load(getClass().getResourceAsStream(
        "/casehub-deployment.yaml"));

    assertThat(goals.agents()).hasSize(4);
    assertThat(goals.channels()).hasSize(3);
    assertThat(goals.caseTypes()).hasSize(2);
    assertThat(goals.trust()).hasSize(2);
    assertThat(goals.adaptations()).hasSize(3);
}
```

- [ ] **Step 3: Run test — verify pass**

- [ ] **Step 4: Write test — adaptation rules parse correctly**

```java
@Test
void adaptationRulesHaveCorrectTargets() throws Exception {
    var loader = new DeploymentGoalLoader(mapper);
    var goals = loader.load(getClass().getResourceAsStream(
        "/casehub-deployment.yaml"));

    assertThat(goals.adaptations().get(0).name())
        .isEqualTo("scale-risk-on-volatility");
    assertThat(goals.adaptations().get(0).trigger().situation())
        .isEqualTo("fsitrading.volatility-spike");
}
```

- [ ] **Step 5: Implement `FsiTradingDeploymentBootstrap`**

```java
@ApplicationScoped
public class FsiTradingDeploymentBootstrap {

    @Inject DeploymentGoalLoader goalLoader;
    @Inject DeploymentAdaptiveSituationRecompiler recompiler;
    @Inject FsiTradingSituationDefinitionProvider situationProvider;

    void onStart(@Observes StartupEvent ev) {
        var goals = goalLoader.loadFromClasspath("/casehub-deployment.yaml");
        var clearanceWindows = situationProvider.registrations().stream()
            .collect(Collectors.toMap(
                r -> r.definition().situationId(),
                r -> r.definition().correlationWindow()));
        String tenancyId = goals.tenancyId() != null
            ? goals.tenancyId() : TenancyConstants.DEFAULT_TENANT_ID;
        recompiler.register(tenancyId, goals, clearanceWindows);

        var graph = compiler.compile(goals, graphFactory);
        lifecycleManager.start(tenancyId, graph);
    }
}
```

- [ ] **Step 6: Run fsitrading app tests**

Run: `mvn --batch-mode -o test -pl fsitrading/app`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/193/fsitrading add .
git -C /Users/mdproctor/claude/casehub/slots/193/fsitrading commit -m "feat(#25): add fsitrading deployment YAML + bootstrap bean

Declares agent topology (4 agents, 3 channels, 2 case types, 2 trust
policies) with 3 adaptive rules (scale-risk, tighten-trust, add-forensics).
Bootstrap wires recompiler registration at startup.

Refs casehubio/casehub-ops#25

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 9: End-to-end adaptation test

**Files:**
- Create: `deployment/src/test/java/io/casehub/ops/deployment/adaptation/AdaptiveSituationRecompilerIntegrationTest.java`

**Interfaces:**
- Consumes: `DeploymentAdaptiveSituationRecompiler`, `DeploymentGoalCompiler`, `DeploymentGoalLoader`, `ActiveSituation`
- Produces: Verified end-to-end adaptation flow (register → situation push → graph adaptation → situation resolve → graph restoration)

- [ ] **Step 1: Write integration test — full adaptation lifecycle**

```java
@Test
void fullAdaptationLifecycle() {
    // 1. Load goals with adaptations
    var goals = loader.load(testYaml);
    recompiler.register("t1", goals,
        Map.of("volatility-spike", Duration.ofMinutes(30),
               "market-anomaly", Duration.ofMinutes(15),
               "active-breach", Duration.ofHours(2)));

    var base = compiler.compile(goals, graphFactory);

    // 2. Push volatility-spike — scale up
    var volatility = new ActiveSituation("volatility-spike", "k1", "t1",
        0.85, Map.of(), Instant.now(), Instant.now(), 1);
    var result1 = recompiler.recompile("t1", base, emptyActual,
        volatility, graphFactory);
    assertThat(result1).isPresent();
    var scaled = result1.get().single().graph();
    assertThat(scaled.nodes()).containsKey(NodeId.of("risk-agent~2"));
    assertThat(scaled.nodes()).containsKey(NodeId.of("risk-agent~3"));

    // 3. Push market-anomaly — tighten trust (composing with scaling)
    var anomaly = new ActiveSituation("market-anomaly", "k2", "t1",
        0.7, Map.of(), Instant.now(), Instant.now(), 1);
    var result2 = recompiler.recompile("t1", scaled, emptyActual,
        anomaly, graphFactory);
    assertThat(result2).isPresent();
    var adapted = result2.get().single().graph();
    assertThat(adapted.nodes()).containsKey(NodeId.of("risk-agent~2")); // still scaled
    // trust tightened

    // 4. Clear volatility — scale down, trust still tightened
    recompiler.clearSituationForTenant("t1", "volatility-spike");
    var result3 = recompiler.recompile("t1", adapted, emptyActual,
        anomaly, graphFactory);
    assertThat(result3).isPresent();
    var afterClear = result3.get();
    // risk-agent~2/~3 no longer present (scaling removed)
    // trust still tightened (market-anomaly still active)
}
```

- [ ] **Step 2: Run integration test**

Run: `mvn --batch-mode -o test -pl deployment -Dtest=AdaptiveSituationRecompilerIntegrationTest`
Expected: PASS

- [ ] **Step 3: Commit**

```bash
git add deployment/
git commit -m "test(#25): end-to-end adaptive situation recompiler integration test

Full lifecycle: register → volatility-spike (scale up) → market-anomaly
(tighten trust, composing with scaling) → situation clearing.

Refs #25

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

- [ ] **Step 4: Run full build across both repos**

Run: `mvn --batch-mode install` (ops)
Run: `mvn --batch-mode -o test` (fsitrading)
Expected: PASS in both

---

## References

- [2026-09-14-fsitrading-adaptive-ops-design.md] — design spec this plan implements
- [deployment/src/main/java/io/casehub/ops/deployment/adaptation/TenantAdaptationState.java] — existing class to modify
- [deployment/src/main/java/io/casehub/ops/deployment/adaptation/AdaptiveTopologyManager.java] — existing class to delete
- [deployment/src/main/java/io/casehub/ops/deployment/adaptation/AdaptationRule.java] — existing, consumed
- [api/src/main/java/io/casehub/ops/api/deployment/DeploymentGoals.java] — existing, already has adaptations field
- [fsitrading/app/src/main/java/io/casehub/fsitrading/app/pipeline/MarketPulseConfiguration.java] — existing pipeline to bridge
- [app/src/main/java/io/casehub/ops/app/lifecycle/ras/DeploymentTopologySituationDefinitionProvider.java] — #84 Ganglion pattern reference
- [GE-20260706-eb11a1] — embedding desiredstate: @DefaultBean stubs
- [GE-20260806-272a90] — adding deployment node type: 6+4 pattern
- [GE-20260806-55e158] — SituationRegistration null correlationKeyExtractor
- Protocol `deployment-spec-tenant-scoped` — tenancyId required
- Protocol `engine-spi-noops-defaultbean` — @DefaultBean for SPI stubs
- GitHub #25 — focal issue
