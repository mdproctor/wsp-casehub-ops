# fsitrading Adaptive Ops — Full-Stack Desired-State Topology Adaptation

**Date:** 2026-09-14
**Issue:** casehubio/casehub-ops#25
**Supersedes:** 2026-06-29-adaptive-ops-design.md (Components 2-3 redesigned for actual SituationRecompiler API)
**Status:** Design

## Problem

casehub-ops-deployment manages agent topology through desired-state reconciliation, but the desired state is static — compiled once from YAML. Real applications need topology that adapts to runtime conditions:

- **Self-healing:** agent dies at 3 AM → re-provision automatically
- **Auto-scaling:** market volatility spikes → scale up risk agents
- **Adaptive security posture:** breach detected → tighten trust policies, add forensics agents

casehub-fsitrading is the first real consumer — multi-agent trading automation with overnight incident response. It has a rich domain model (190+ source files, case definitions for overnight-incident, oversight, strategy-evaluation) but zero deployment topology infrastructure.

## Research Backing

Deep research (26 sources, 109 agents, adversarial verification — 10 confirmed, 15 killed):

- Financial trading multi-agent systems lack desired-state reconciliation, self-healing, and auto-scaling (AgenticTrading, NeurIPS 2025 Workshop — 6-0 verified)
- 47.6% of self-adaptive research focuses on infrastructure, 0% on application level (SEAMS 2021 — 5-1 verified)
- "Agent-as-configuration" pattern validated by Snapchat's Auton framework but with instantiation-time reconciliation only (6-0 verified)

## Architecture Decision

**GoalCompiler + SituationRecompiler + FaultPolicy.** Three mechanisms for three genuinely different concerns:

| Concern | Owner | Mechanism |
|---|---|---|
| Static topology (base desired state) | `DeploymentGoalCompiler` | Compiles YAML into `DesiredStateGraph` |
| Planned adaptation (conditions change) | `DeploymentAdaptiveSituationRecompiler` | Implements `SituationRecompiler` SPI — receives active situations, recompiles graph |
| Unplanned response (node failure) | ReconciliationLoop + `FaultPolicy` | Re-provisions ABSENT nodes automatically; `FaultPolicy` escalates after repeated failures (tier-3 review node) |

**Layer separation:**

- **`SituationRecompiler` SPI** in `casehub-desiredstate-api` — domain-agnostic contract. The runtime iterates registered recompilers by ascending priority and returns the first non-empty result (chain-of-responsibility, first-match-wins). Already exists (desiredstate#49, closed).
- **`DeploymentAdaptiveSituationRecompiler`** in `casehub-ops-deployment` — deployment-domain-specific implementation that applies YAML-defined adaptation rules.
- **Situation detection** in `casehub-fsitrading` — Ganglion detectors + summarisation pipeline for market conditions. Adapted from the #84 pattern; the `ganglion()` helper adds a parameterised `eventType` parameter (fsitrading uses both `CONDITION` and `SIGNAL` event types, unlike deployment-monitoring which hardcodes `PHASE`).
- **`ReconciliationLoop` unchanged** — `start()`, `updateDesired()`, `requestReconciliation()` all exist and work as designed.

---

## Component 1: Adaptive YAML Format

*Unchanged from original spec.* The deployment YAML gains an `adaptations:` section. Note: situation IDs in this example are generic for illustration — actual deployments use domain-qualified IDs (e.g., `fsitrading.volatility-spike`) as shown in Component 3.

```yaml
agents:
  - spec:
      agentId: triage-agent
      name: Alert Triage
      slot: worker
  - spec:
      agentId: risk-agent
      name: Risk Monitor
      slot: worker

channels:
  - spec:
      name: trading/work
      semantic: APPEND

trust:
  - spec:
      capability: review
      threshold: 0.7
      minimumObservations: 5

adaptations:
  - name: scale-risk-on-volatility
    trigger:
      situation: volatility-spike
      minConfidence: 0.7
      deactivateBelow: 0.5
      cooldown: PT5M
    actions:
      - type: scale
        target: risk-agent
        min: 1
        max: 5

  - name: add-forensics-on-breach
    trigger:
      situation: active-breach
      minConfidence: 0.8
    actions:
      - type: add
        nodes:
          agents:
            - spec:
                agentId: forensics-agent
                name: Forensics Specialist
                slot: worker
          channels:
            - spec:
                name: soc/forensics
                semantic: APPEND

  - name: tighten-trust-on-anomaly
    trigger:
      situation: market-anomaly
      minConfidence: 0.6
    actions:
      - type: update
        target: review
        nodeType: trust
        fields:
          threshold: 0.9
          bootstrapEscalationRequired: true
```

### Action Types

| Action | What it does | Graph operation |
|---|---|---|
| `scale` | Adjust instance count of a base node | Add/remove `DesiredNode` instances with derived IDs (`risk-agent~2`, `risk-agent~3`, ...) |
| `add` | Insert new nodes (full spec inline) | `withNode()` for each declared node |
| `update` | Modify fields of an existing node's spec | Jackson tree-merge → `withMutation(UpdateNode(...))` |

### Target Resolution

For `update` and `scale` actions, `target` matches against the `nodeId()` value of node specs in the base topology:

| Node type | `nodeId()` returns | Example target |
|---|---|---|
| Agent | `agentId` | `risk-agent` |
| Trust policy | `capability` | `review` |
| Channel | `name` | `trading/work` |
| Case type | `namespace:name:version` | `trading:market-order:1` |
| Endpoint | `path` | `/api/trading/submit` |

When `nodeType` is specified, matching is restricted to nodes of that type.

### Field Merge for Update Actions

The `update` action specifies partial field overrides in the YAML `fields:` map. The graph API's `UpdateNode(NodeId, NodeSpec)` replaces the entire `NodeSpec`, so partial updates require constructing a complete new spec:

```java
ObjectNode base = mapper.valueToTree(existingSpec);
ObjectNode overrides = mapper.valueToTree(fields);
base.setAll(overrides);
NodeSpec merged = mapper.treeToValue(base, existingSpec.getClass());
```

**Parse-time field validation:** At parse time, every key in the `fields` map is validated against the target `NodeSpec` type's Jackson-visible fields. Unknown field names (e.g., `treshold` instead of `threshold`) fail with a descriptive error at startup, not silently at runtime. The `AdaptationRule.fromSpecs()` factory performs this validation using the `ObjectMapper`'s `getSerializationConfig()` to introspect the target class's field names. The target class is determined by `nodeType` (or inferred from the matched base topology node).

### Scale Instance Count

For a scale action with range [min, max], active situation confidence C, and trigger minConfidence T:

```
effective = clamp((C - T) / (1.0 - T), 0.0, 1.0)
instanceCount = clamp(min + (int)((max - min) * effective), min, max)
```

Parse-time validation: `minConfidence` must be strictly less than `1.0`.

The base node (e.g., `risk-agent`) is always present. Derived instances use the `~` separator: `risk-agent~2` through `risk-agent~N`. Derived instances are constructed via `AgentNodeSpec.withAgentId(derivedId)`.

### Scale-Down Ordering

Instances removed highest-numbered first (LIFO). Graceful shutdown is the `TransitionExecutor`'s responsibility.

### Deactivation Semantics

All adaptation actions are ephemeral — they exist only while their triggering situation is active. The recompiler recompiles from the base graph on every invocation, applying only rules whose situations are currently active. When a situation clears:

- **`scale`** — derived instances (e.g., `risk-agent~2`, `risk-agent~3`) are no longer added to the recompiled graph. The reconciliation loop detects them as extraneous and deprovisions them (LIFO order).
- **`add`** — added nodes (e.g., `forensics-agent`, `fsitrading/forensics` channel) are not present in the recompiled graph. Reconciliation deprovisions them. Graceful shutdown is the `TransitionExecutor`'s responsibility.
- **`update`** — the base spec is restored. The next recompilation produces the original field values.

No `remove` action type exists — this is deliberate. The recompile-from-base pattern means node absence is expressed by not adding nodes, not by explicitly removing them. A `remove` action would modify the base topology definition, which is the `GoalCompiler`'s concern, not the `SituationRecompiler`'s.

### NodeId Collision Validation

At parse time:

1. **Scale actions:** `target` node ID must not contain `~`. No base topology node may match `{target}~{n}` for any n in [2, max].
2. **Add actions:** all node IDs declared in `add` actions must be distinct from base topology node IDs. `ImmutableDesiredStateGraph.withNode()` silently overwrites existing nodes with the same `NodeId` — parse-time validation catches this before runtime. Additionally, node IDs across all `add` actions (across all adaptation rules) must be mutually distinct to prevent cross-rule overwrites.
3. **Cross-action:** no `add` action node ID may match any `scale` action's derived ID pattern (`{target}~{n}`).

### Hysteresis and Cooldown

- **`deactivateBelow`** — deactivates when confidence drops below this. Defaults to `minConfidence`.
- **`cooldown`** — ISO 8601 duration. Minimum time between state changes. Defaults to `PT0S`.

### Conflict Detection

Multiple simultaneously active adaptations modifying the same node: YAML declaration order defines precedence — later rules override earlier ones.

Conflicts are observable via:
- **Log warning** — includes rule name and target node ID
- **Metric** — `desiredstate.adaptation.conflict.total` counter (tags: `tenancy_id`, `rule_name`, `node_id`). Alertable in Grafana for operations teams.

Parse-time validation warns (not errors) when multiple adaptation rules target the same `nodeType`/`target` combination, since conflicts only materialise when the corresponding situations are simultaneously active.

---

## Component 2: DeploymentAdaptiveSituationRecompiler

**Replaces the original spec's Components 2-3 (SituationSource + AdaptiveTopologyManager).**

The deployment module implements the `SituationRecompiler` SPI from `casehub-desiredstate-api`:

```java
public interface SituationRecompiler {
    Optional<CompilationResult> recompile(
        String tenancyId,
        DesiredStateGraph currentGraph,
        ActualState actualState,
        ActiveSituation situation,
        DesiredStateGraphFactory factory
    );
    default int priority() { return 0; }

    /**
     * Called by the dispatch layer when a situation resolves (ChangeType.RESOLVED).
     * Allows recompilers to immediately deactivate adaptations rather than
     * waiting for TTL-based clearance.
     *
     * @return non-empty if the graph changed due to deactivation
     */
    default Optional<CompilationResult> situationResolved(
        String tenancyId,
        String situationId,
        DesiredStateGraph currentGraph,
        ActualState actualState,
        DesiredStateGraphFactory factory) {
        return Optional.empty();
    }
}
```

**Note:** `situationResolved()` is a new default method proposed by this spec for `casehub-desiredstate-api` — see §Cross-Repo Dependencies. The default no-op preserves backward compatibility for existing recompilers.

The runtime's `SituationRecompilerEngine` iterates registered `SituationRecompiler` instances by ascending priority and returns the first non-empty result (chain-of-responsibility, first-match-wins). Priority ordering: domain recompilers at default priority (0), `DeploymentAdaptiveSituationRecompiler` at 100, `CbrSituationRecompiler` at `Integer.MAX_VALUE` (fallback). The deployment recompiler returns `Optional.empty()` for situations it doesn't handle, allowing lower-precedence recompilers to respond. The deployment module provides one implementation:

```java
@ApplicationScoped
public class DeploymentAdaptiveSituationRecompiler implements SituationRecompiler {

    @Inject DeploymentGoalCompiler compiler;

    private final ConcurrentHashMap<String, TenantAdaptationState> tenantStates =
        new ConcurrentHashMap<>();

    @Override
    public int priority() {
        return 100; // After default domain recompilers (0), before CBR fallback (MAX_VALUE)
    }

    @Inject ObjectMapper mapper;
    @Inject DesiredStateGraphFactory factory;

    public void register(String tenancyId, DeploymentGoals goals,
                         Map<String, Duration> situationClearanceWindows) {
        List<AdaptationRule> rules = AdaptationRule.fromSpecs(
            goals.adaptations(), compiler, mapper, factory);
        var state = new TenantAdaptationState(goals, rules, situationClearanceWindows);
        tenantStates.put(tenancyId, state);
    }

    @Override
    public Optional<CompilationResult> recompile(
            String tenancyId,
            DesiredStateGraph currentGraph,
            ActualState actualState,
            ActiveSituation situation,
            DesiredStateGraphFactory factory) {

        TenantAdaptationState state = tenantStates.get(tenancyId);
        if (state == null || state.rules().isEmpty()) {
            return Optional.empty();
        }

        synchronized (state) {
            // Update tracked situation state
            state.updateSituation(situation);

            // Recompile from base with ALL currently active situations
            DesiredStateGraph base = compiler.compile(state.goals(), factory);
            DesiredStateGraph adapted = base;
            Set<NodeId> modifiedNodes = new HashSet<>();

            for (AdaptationRule rule : state.rules()) {
                Optional<ActiveSituation> match = state.activeSituationFor(rule);
                if (match.isPresent() && state.shouldActivate(rule, match.get())) {
                    Set<NodeId> targets = rule.targetNodeIds(base);
                    for (NodeId t : targets) {
                        if (modifiedNodes.contains(t)) {
                            LOG.warning("Conflict: rule '%s' modifies '%s' "
                                + "already modified by earlier rule"
                                .formatted(rule.name(), t));
                        }
                    }
                    adapted = rule.apply(adapted, match.get());
                    modifiedNodes.addAll(targets);
                }
            }

            state.clearAbsentSituations();

            if (graphsEqual(adapted, base)) {
                return Optional.empty();
            }
            return Optional.of(CompilationResult.single(adapted));
        }
    }

    @Override
    public Optional<CompilationResult> situationResolved(
            String tenancyId,
            String situationId,
            DesiredStateGraph currentGraph,
            ActualState actualState,
            DesiredStateGraphFactory factory) {

        TenantAdaptationState state = tenantStates.get(tenancyId);
        if (state == null) {
            return Optional.empty();
        }

        synchronized (state) {
            boolean hadSituation = state.clearSituation(situationId);
            if (!hadSituation) {
                return Optional.empty();
            }

            DesiredStateGraph base = compiler.compile(state.goals(), factory);
            DesiredStateGraph adapted = base;
            Set<NodeId> modifiedNodes = new HashSet<>();

            for (AdaptationRule rule : state.rules()) {
                Optional<ActiveSituation> match = state.activeSituationFor(rule);
                if (match.isPresent() && state.shouldActivate(rule, match.get())) {
                    Set<NodeId> targets = rule.targetNodeIds(base);
                    adapted = rule.apply(adapted, match.get());
                    modifiedNodes.addAll(targets);
                }
            }

            if (graphsEqual(adapted, currentGraph)) {
                return Optional.empty();
            }
            return Optional.of(CompilationResult.single(adapted));
        }
    }

    private static boolean graphsEqual(DesiredStateGraph a, DesiredStateGraph b) {
        if (a == b) return true;
        if (a == null || b == null) return false;
        return a.nodes().equals(b.nodes()) && a.dependencies().equals(b.dependencies());
    }
}
```

`graphsEqual()` performs structural comparison — node maps and dependency sets — rather than identity comparison. `ImmutableDesiredStateGraph` has no `equals()` override (it inherits `Object.equals()`, which is identity-based). Structural comparison prevents unnecessary graph updates when rules fire but produce a topology identical to the current desired state.

### Per-Tenant State

`TenantAdaptationState` already exists in `io.casehub.ops.deployment.adaptation` — this spec **modifies** it. Existing fields (`goals`, `rules`, `activePerRule`, `lastChangePerRule`) are retained. Modifications:

| Change | Detail |
|---|---|
| **Add field** | `Map<String, ActiveSituation> trackedSituations` — keyed by `situationId` |
| **Add field** | `Map<String, Duration> situationClearanceWindows` — per-situation TTL for absence detection |
| **Add method** | `updateSituation(ActiveSituation)` — upserts into `trackedSituations` |
| **Add method** | `activeSituationFor(AdaptationRule)` — looks up tracked situation matching the rule's trigger `situationId` |
| **Add method** | `clearSituation(String situationId)` — removes a specific situation from `trackedSituations` and resets the corresponding rule's activation state. Returns `true` if the situation was tracked. Called by `situationResolved()` for immediate deactivation on RESOLVED events. |
| **Replace method** | `clearAbsentSituations(Set<String>)` → `clearAbsentSituations()` — the old signature takes externally-polled IDs from `SituationSource`; the new version uses `lastSignal` timestamps and `situationClearanceWindows` to determine absence internally |
| **Update Javadoc** | References to `AdaptiveTopologyManager` → `DeploymentAdaptiveSituationRecompiler` |

After modification, `TenantAdaptationState` holds:
- The tenant's `DeploymentGoals` (base topology) — *existing*
- Parsed `List<AdaptationRule>` (from `goals.adaptations()`) — *existing*
- Tracked active situations: `Map<String, ActiveSituation>` keyed by `situationId` — *new*
- Situation clearance windows: `Map<String, Duration>` per-situation TTLs — *new*
- Hysteresis state: `activePerRule` and `lastChangePerRule`, keyed by rule name — *existing*

### Situation Tracking

The `SituationRecompiler` is called per-situation. To apply all active situations during recompilation:

1. `updateSituation(situation)` — upserts the situation into the tracked set (keyed by `situationId`)
2. `activeSituationFor(rule)` — looks up the tracked situation matching `rule.trigger().situation()`
3. `clearAbsentSituations()` — after processing, removes tracked situations where `now - lastSignal > situationClearanceWindows.get(situationId)`. The clearance windows are provided by the bootstrap bean during `register()`, extracted from each `SituationDefinition.correlationWindow()`. This avoids a runtime dependency on `SituationDefinitionRegistry` — the TTLs are static per deployment. Absent situations with cooldown are retained until cooldown expires.

This ensures the recompiler has a complete view of all active situations, not just the one being pushed in this call. The per-situation TTL threshold (rather than a fixed 5-minute window) prevents premature clearing of long-lived situations — e.g., a `volatility-spike` with 30-minute `correlationWindow` is not cleared after 5 minutes of no new pushes.

### Deactivation Path

Two-tier deactivation ensures both promptness and reliability:

1. **Primary: `situationResolved()`** — the dispatch layer observes `SituationChangeEvent` with `ChangeType.RESOLVED` and calls `situationResolved()` on the engine. `DeploymentAdaptiveSituationRecompiler.situationResolved()` calls `state.clearSituation(situationId)` and recompiles from base with the remaining active situations. Deactivation is immediate.

2. **Fallback: `clearAbsentSituations()`** — runs on every `recompile()` invocation. If the RESOLVED event was lost (CDI async events can be dropped under load), the TTL-based clearing eventually removes stale situations. This is a safety net, not the primary deactivation path.

For `active-breach` with a 2-hour `correlationWindow`, the primary path deactivates forensics agents and tightened trust policies immediately on breach resolution. The fallback path would take up to 2 hours — acceptable only as a lost-event safety net, not as designed behavior.

### Hysteresis and Cooldown

Same logic as the original spec — `shouldActivate()` handles hysteresis band and cooldown checks. Lives inside `TenantAdaptationState` instead of `AdaptiveTopologyManager`.

### Registration

When the deployment app bootstraps, it calls `register(tenancyId, goals, situationClearanceWindows)` to provide the base topology, adaptation rules, and per-situation clearance windows. The bootstrap bean:

1. Constructs `situationClearanceWindows` from the `SituationDefinitionProvider`'s registrations — mapping each `situationId` to its `SituationDefinition.correlationWindow()`
2. Calls `register(tenancyId, goals, situationClearanceWindows)`
3. Calls `LifecycleManager.start()` with the base (un-adapted) topology

**Cold-start recovery** is the dispatch layer's responsibility (see §Component 6). After the reconciliation loop starts, the dispatch layer queries `SituationSource.activeSituations(tenancyId)` for already-active situations and feeds them through `SituationRecompilerEngine.recompile()`. This produces adapted topology within the first reconciliation cycle — if the market is volatile at restart time, risk agents scale before the next debounce window closes (default 1 second).

### Key Differences from Original Spec

| Original (AdaptiveTopologyManager) | Updated (SituationRecompiler) |
|---|---|
| CDI observer of `SituationChangeEvent` | SPI implementation — runtime pushes situations |
| Pulls situations from `SituationSource` | Receives `ActiveSituation` directly |
| Calls `updateDesired()` and `requestReconciliation()` | Returns `CompilationResult` — runtime handles graph update |
| Periodic re-poll safety net | TTL-based `clearAbsentSituations()` as safety net (fallback only) |
| Pulls `SituationSource` for active set | `SituationSource` retained in `casehub-ras-api` — used by dispatch layer for cold start, not by recompiler |
| `SituationChangeEvent` CDI event observed directly | Dispatch layer in `casehub-desiredstate` bridges events to engine |
| No resolution signal path (polls active set) | `situationResolved()` for immediate deactivation on RESOLVED events |

### Migration: AdaptiveTopologyManager Removal

`DeploymentAdaptiveSituationRecompiler` fully replaces `AdaptiveTopologyManager`. The following must be deleted:

| File | Reason |
|---|---|
| `AdaptiveTopologyManager.java` | Replaced by `DeploymentAdaptiveSituationRecompiler` |
| `StubSituationSource.java` (`ops/app/spi/`) | No-op SPI stub — `SituationSource` no longer needed |

**`SituationSource` interface retained:** The `SituationSource` interface lives in `casehub-ras-api` (package `io.casehub.ras.api`). The `DeploymentAdaptiveSituationRecompiler` does not inject or use `SituationSource` — it receives situations from the dispatch layer via the `SituationRecompiler` SPI. However, `SituationSource` is retained because the dispatch layer in `casehub-desiredstate` uses it for cold-start recovery (querying active situations on startup). `StubSituationSource` in `ops/app/spi/` is deleted — the no-op stub is no longer needed since the recompiler does not inject `SituationSource`. (Note: `StubSituationSource` is `@ApplicationScoped`, not `@DefaultBean` — it was the only implementation in ops.)

**`ReconciliationTarget` inner interface:** defined inside `AdaptiveTopologyManager` — deleted with the class. No external references (the reconciliation loop uses `ReconciliationLoop.start()` directly).

**Documentation references:** `AdaptiveTopologyManager` has 25+ references across docs, specs, guides, and plans. These should be updated to reference `DeploymentAdaptiveSituationRecompiler` as part of this issue's documentation pass. The original spec (`2026-06-29-adaptive-ops-design.md`) is already superseded by this spec.

### Thread Safety

- **Cross-tenant:** `ConcurrentHashMap<String, TenantAdaptationState>` — no cross-tenant interference.
- **Per-tenant:** `synchronized(state)` in `recompile()` — serializes compilation and state mutation.

---

## Component 3: fsitrading Deployment YAML

The fsitrading agent topology declared as desired state. This YAML lives in `casehub-fsitrading/app/src/main/resources/`:

```yaml
# casehub-deployment.yaml — fsitrading agent topology
tenancyId: ${TENANT_ID:default}

agents:
  - spec:
      agentId: strategy-agent
      name: Strategy Evaluator
      slot: worker
  - spec:
      agentId: strategy-agent-2
      name: Strategy Evaluator Backup
      slot: worker
  - spec:
      agentId: risk-agent
      name: Risk Monitor
      slot: worker
  - spec:
      agentId: audit-agent
      name: Trade Auditor
      slot: worker

channels:
  - spec:
      name: fsitrading/work
      semantic: APPEND
  - spec:
      name: fsitrading/alerts
      semantic: APPEND
  - spec:
      name: fsitrading/audit
      semantic: APPEND

caseTypes:
  - spec:
      namespace: fsitrading
      name: overnight-incident
      version: "1.0"
  - spec:
      namespace: fsitrading
      name: strategy-evaluation
      version: "1.0"

trust:
  - spec:
      capability: trade-execution
      threshold: 0.7
      minimumObservations: 5
  - spec:
      capability: risk-assessment
      threshold: 0.8
      minimumObservations: 3

adaptations:
  - name: scale-risk-on-volatility
    trigger:
      situation: fsitrading.volatility-spike
      minConfidence: 0.7
      deactivateBelow: 0.5
      cooldown: PT5M
    actions:
      - type: scale
        target: risk-agent
        min: 1
        max: 5

  - name: tighten-trust-on-anomaly
    trigger:
      situation: fsitrading.market-anomaly
      minConfidence: 0.6
    actions:
      - type: update
        target: trade-execution
        nodeType: trust
        fields:
          threshold: 0.9
          bootstrapEscalationRequired: true

  - name: add-forensics-on-breach
    trigger:
      situation: fsitrading.active-breach
      minConfidence: 0.8
    actions:
      - type: add
        nodes:
          agents:
            - spec:
                agentId: forensics-agent
                name: Forensics Specialist
                slot: worker
          channels:
            - spec:
                name: fsitrading/forensics
                semantic: APPEND
      - type: update
        target: risk-assessment
        nodeType: trust
        fields:
          threshold: 0.95
```

### Agent Topology Rationale

| Agent | Purpose | Count |
|---|---|---|
| `strategy-agent` / `strategy-agent-2` | Evaluate trading strategies, overnight coverage | 2 (base, static — redundancy pair, not scaled adaptively) |
| `risk-agent` | Monitor risk exposure, position limits | 1 (scales to 5 on volatility) |
| `audit-agent` | Trade audit trail, regulatory compliance | 1 (always on) |
| `forensics-agent` | Breach investigation | 0 (added on active-breach) |

---

## Component 4: fsitrading RAS — Summarisation Pipeline + Situation Detectors

Market condition signals flow through a RAS summarisation pipeline following the #84 pattern (deployment-monitoring.yaml). This lives in `casehub-fsitrading`.

### Summarisation Pipeline

```yaml
# META-INF/summarisation/fsitrading-market-monitoring.yaml
pipeline:
  name: fsitrading-market-monitoring
  source:
    type-prefix: io.casehub.fsitrading.market.

  levels:
    - name: signals
      grouping:
        type: windowed
        count: 20
        age: 30000   # 30-second windows
      summariser:
        type: threshold-classify
        rules:
          - name: price-spike
            when: 'abs(priceChange) > volatilityThreshold'
            category: PRICE_SPIKE
            severity: HIGH
          - name: volume-surge
            when: 'volumeRatio > 3.0'
            category: VOLUME_SURGE
            severity: HIGH
          - name: spread-widening
            when: 'spreadBps > normalSpreadBps * 2'
            category: SPREAD_WIDENING
            severity: MEDIUM
          - name: order-imbalance
            when: 'bidAskImbalance > 0.7'
            category: ORDER_IMBALANCE
            severity: MEDIUM
          - name: normal-tick
            when: 'true'
            category: NORMAL
            severity: LOW
      emit:
        cloud-event-type: io.casehub.fsitrading.situation.signal

    - name: conditions
      grouping:
        type: windowed
        age: 60000   # 1-minute windows
        count: 30
      aggregate-fields:
        - severity
        - category
      summariser:
        type: phase-detect
        initial: STABLE
        states:
          - STABLE
          - VOLATILE
          - ANOMALOUS
        transitions:
          - from: STABLE
            to: VOLATILE
            count-field: severity
            count-value: HIGH
            op: ">="
            threshold: 5
          - from: STABLE
            to: ANOMALOUS
            count-field: category
            count-value: SPREAD_WIDENING
            op: ">="
            threshold: 5
          - from: VOLATILE
            to: ANOMALOUS
            count-field: category
            count-value: SPREAD_WIDENING
            op: ">="
            threshold: 3
          - from: ANOMALOUS
            to: VOLATILE
            count-field: category
            count-value: NORMAL
            op: ">="
            threshold: 10
          - from: VOLATILE
            to: STABLE
            count-field: severity
            count-value: HIGH
            op: "=="
            threshold: 0
      emit:
        cloud-event-type: io.casehub.fsitrading.situation.condition
```

### Market Event Source

The summarisation pipeline consumes CloudEvents matching `io.casehub.fsitrading.market.*`. The fsitrading project has an existing code-based pipeline (`MarketPulseConfiguration`) that processes market data through 5 internal levels (tick → OHLCV bars → trends → regime → narrative) via `EventStreamBus`. This internal pipeline does not emit CloudEvents.

A new `MarketConditionCloudEventPublisher` bridges the internal pipeline to RAS:

```java
@ApplicationScoped
public class MarketConditionCloudEventPublisher {

    @Inject CloudEventEmitter emitter;

    public void onRegimeAssessment(RegimeAssessment assessment) {
        emitter.emit(CloudEventBuilder.v1()
            .withType("io.casehub.fsitrading.market.assessment")
            .withSource(URI.create("/fsitrading/market-pulse"))
            .withData(assessment.toCloudEventData())
            .build());
    }
}
```

This publisher subscribes to the `MarketPulseConfiguration` L3 (RegimeAssessment) output and emits CloudEvents that the RAS summarisation pipeline consumes. The summarisation pipeline then applies its own windowed aggregation and phase detection on top of these events — a second stage of summarisation that produces the ganglion-consumable signals.

**Namespace separation:** following the deployment-monitoring pattern (`io.casehub.desiredstate.node.*` → `io.casehub.ops.deployment.anomaly`/`.phase`), the input and output namespaces are disjoint. The publisher emits under `io.casehub.fsitrading.market.*` (pipeline input), while the pipeline emits under `io.casehub.fsitrading.situation.*` (pipeline output consumed by ganglia). This prevents re-ingestion — `CloudEventIngestionAdapter` filters by `startsWith(typePrefix)` with no self-emission guard.

### Event Types

```java
public final class FsiTradingEventTypes {
    public static final String SIGNAL = "io.casehub.fsitrading.situation.signal";
    public static final String CONDITION = "io.casehub.fsitrading.situation.condition";
    public static final String SECURITY = "io.casehub.fsitrading.security.alert";
}
```

**Namespace design:** Market events and security events are separate concerns with separate event types:

| Event type | Source | Consumed by |
|---|---|---|
| `io.casehub.fsitrading.market.*` | `MarketConditionCloudEventPublisher` | Summarisation pipeline (input) |
| `io.casehub.fsitrading.situation.signal` | Summarisation L1 (threshold-classify) | `volatile-detected`, `anomalous-detected`, `stable-detected` ganglia |
| `io.casehub.fsitrading.situation.condition` | Summarisation L2 (phase-detect) | `volatile-detected`, `anomalous-detected`, `stable-detected` ganglia |
| `io.casehub.fsitrading.security.alert` | `SecurityAlertBridge` | `breach-signal` ganglion |

Security alerts bypass the summarisation pipeline entirely — they go directly to the `breach-signal` ganglion. A breach is a security event, not a market condition.

### Security Event Contract

The fsitrading domain defines a security event record for internal signalling:

```java
public record FsiTradingSecurityEvent(
    Severity severity,
    String source,
    String detail,
    Instant timestamp
) {
    public enum Severity { INFORMATIONAL, WARNING, CRITICAL }
}
```

Sources that fire `FsiTradingSecurityEvent` as a CDI event:

- **Foundation module deregistration** — an agent is forcibly removed outside the reconciliation loop
- **Trust policy violation** — an agent attempts an action beyond its trust threshold
- **External SIEM integration** — security monitoring detects anomalous access patterns

This is distinct from Quarkus's `io.quarkus.security.spi.runtime.SecurityEvent` (HTTP security events). The fsitrading domain's security events are domain-specific signals about trading infrastructure integrity.

### Security Alert Bridge

```java
@ApplicationScoped
public class SecurityAlertBridge {

    @Inject CloudEventEmitter emitter;

    public void onSecurityEvent(@ObservesAsync FsiTradingSecurityEvent event) {
        if (event.severity() == FsiTradingSecurityEvent.Severity.CRITICAL) {
            emitter.emit(CloudEventBuilder.v1()
                .withType(FsiTradingEventTypes.SECURITY)
                .withSource(URI.create("/fsitrading/security"))
                .withData(Map.of(
                    "category", "BREACH",
                    "source", event.source(),
                    "detail", event.detail(),
                    "timestamp", event.timestamp().toString()))
                .build());
        }
    }
}
```

The `SecurityAlertBridge` observes `FsiTradingSecurityEvent` via CDI and emits `io.casehub.fsitrading.security.alert` CloudEvents when a CRITICAL severity event occurs. The bridge converts domain security signals into CloudEvents consumable by the `breach-signal` ganglion. The ganglion requires `category: "BREACH"` — the bridge only emits for CRITICAL severity events, filtering out warnings and informational security events.

### Situation Definition Provider

```java
@ApplicationScoped
public class FsiTradingSituationDefinitionProvider implements SituationDefinitionProvider {

    public static final String VOLATILITY_SPIKE = "fsitrading.volatility-spike";
    public static final String MARKET_ANOMALY = "fsitrading.market-anomaly";
    public static final String ACTIVE_BREACH = "fsitrading.active-breach";

    @Override
    public List<GanglionDescriptor> ganglionDescriptors() {
        return List.of(
            ganglion("volatile-detected", FsiTradingEventTypes.CONDITION, ctx -> {
                var d = data(ctx);
                return d != null && "VOLATILE".equals(d.get("to"));
            }, 0.8),
            ganglion("anomalous-detected", FsiTradingEventTypes.CONDITION, ctx -> {
                var d = data(ctx);
                return d != null && "ANOMALOUS".equals(d.get("to"));
            }, 0.95),
            ganglion("stable-detected", FsiTradingEventTypes.CONDITION, ctx -> {
                var d = data(ctx);
                return d != null && "STABLE".equals(d.get("to"));
            }, 0.9),
            ganglion("breach-signal", FsiTradingEventTypes.SECURITY, ctx -> {
                var d = data(ctx);
                return d != null && "BREACH".equals(d.get("category"));
            }, 0.99)
        );
    }

    @Override
    public List<SituationRegistration> registrations() {
        return List.of(
            // volatility-spike: volatile phase detected
            new SituationRegistration(
                new SituationDefinition(
                    VOLATILITY_SPIKE,
                    Set.of(FsiTradingEventTypes.CONDITION),
                    Duration.ofMinutes(30),
                    null,
                    new ChainMode.Count("volatile-detected", 1),
                    new TriggerAction.NotifyOnly(),
                    new TriggerMode.Repeating(Duration.ofMinutes(5))),
                null),

            // market-anomaly: anomalous state detected
            new SituationRegistration(
                new SituationDefinition(
                    MARKET_ANOMALY,
                    Set.of(FsiTradingEventTypes.CONDITION),
                    Duration.ofMinutes(15),
                    null,
                    new ChainMode.Count("anomalous-detected", 1),
                    new TriggerAction.NotifyOnly(),
                    new TriggerMode.Repeating(Duration.ofMinutes(2))),
                null),

            // active-breach: immediate, high-confidence
            new SituationRegistration(
                new SituationDefinition(
                    ACTIVE_BREACH,
                    Set.of(FsiTradingEventTypes.SECURITY),
                    Duration.ofHours(2),
                    null,
                    new ChainMode.Count("breach-signal", 1),
                    new TriggerAction.CreateCase(
                        new CaseTriggerConfig("fsitrading", "overnight-incident",
                            "1.0", Map.of())),
                    new TriggerMode.FireOnce()),
                null)
        );
    }

    @SuppressWarnings("unchecked")
    private static Map<String, Object> data(Map ctx) {
        Object d = ctx.get("data");
        return d instanceof Map ? (Map<String, Object>) d : null;
    }

    @SuppressWarnings("unchecked")
    private static GanglionDescriptor ganglion(String id, String eventType,
                                                java.util.function.Function<Map, Boolean> condition,
                                                double confidence) {
        return new GanglionDescriptor.ExpressionRules(
                id,
                Set.of(eventType),
                List.of(new GanglionDescriptor.ExpressionRules.Rule(
                        new LambdaExpression<>(condition),
                        DetectionSignal.DETECTED,
                        confidence,
                        null,
                        Map.of())),
                Map.of());
    }
}
```

The `ganglion()` helper follows the same pattern as `DeploymentTopologySituationDefinitionProvider.ganglion()` in ops #84, parameterised with `eventType` since fsitrading uses `CONDITION`, `SIGNAL`, and `SECURITY` event types (unlike deployment-monitoring which hardcodes `PHASE`).

### Situation Semantics

| Situation | Chain | TTL | Action | Mode | Adaptation |
|---|---|---|---|---|---|
| `fsitrading.volatility-spike` | `Count("volatile-detected", 1)` | 30 min | `NotifyOnly` | `Repeating(5m)` | Scale risk agents 1→5 |
| `fsitrading.market-anomaly` | `Count("anomalous-detected", 1)` | 15 min | `NotifyOnly` | `Repeating(2m)` | Tighten trade-execution trust to 0.9 |
| `fsitrading.active-breach` | `Count("breach-signal", 1)` | 2 hours | `CreateCase` | `FireOnce` | Add forensics agent + channel, tighten risk-assessment trust to 0.95 |

`TriggerAction.NotifyOnly()` for volatility-spike and market-anomaly — these situations exist purely to drive topology adaptation via `SituationRecompiler`. No case is created. `TriggerMode.Repeating` keeps the situation's TTL refreshed while the underlying condition persists. `active-breach` creates an overnight-incident case AND drives adaptation, firing once per chain activation.

---

## Component 5: FaultPolicy — Unchanged

The existing `DeploymentFaultPolicy` is not modified by this spec. It delegates to `ThresholdFaultPolicy` with tier-3 escalation: after 3 consecutive `PROVISION_FAILED` faults for a node, it adds a `deployment-review` node to the graph for human review. This handles repeated provision failures — a concern orthogonal to topology adaptation.

Self-healing for transient failures is built into the reconciliation cycle:

1. `ActualStateAdapter.readActual()` reports `NodeStatus.ABSENT`
2. `TransitionPlanner.plan()` generates a PROVISION step
3. The node is re-provisioned automatically

The fault policy handles the case where re-provisioning repeatedly fails — escalation, not adaptation.

---

## Component 6: Reconciliation Triggering

The `AdaptiveTopologyManager` observed `SituationChangeEvent` via CDI and called both `reconciliationTarget.updateDesired()` AND `reconciliationTarget.requestReconciliation()` — immediate reconciliation on every situation change.

The `SituationRecompiler` SPI replaces this with a return-value contract: the recompiler returns `CompilationResult`, and the dispatch layer handles graph update and reconciliation triggering. The dispatch flow:

1. RAS detects a situation (e.g., `fsitrading.volatility-spike`) and fires `SituationChangeEvent`
2. Dispatch layer observes the event, calls `SituationRecompilerEngine.recompile()`
3. Engine iterates recompilers by priority; `DeploymentAdaptiveSituationRecompiler` returns adapted graph via `CompilationResult.single()`
4. Dispatch layer calls `LifecycleManager.updateDesired(tenancyId, result)` — swaps graph atomically
5. Dispatch layer calls `ReconciliationLoop.requestReconciliation(tenancyId)` — schedules immediate debounced reconciliation

**Step 5 is critical.** `LifecycleManager.updateDesired()` only swaps the graph reference — it does NOT trigger reconciliation. Without an explicit `requestReconciliation()` call, the new graph takes effect only at the next periodic resync (up to 5 minutes away). For breach response, this latency is unacceptable.

**Dispatch layer ownership:** The dispatch layer that bridges `SituationChangeEvent` to `SituationRecompilerEngine` lives in `casehub-desiredstate` runtime. **This wiring does not yet exist** — desiredstate#49 delivered the SPI (`SituationRecompiler` interface) and engine (`SituationRecompilerEngine` class) but not the dispatch wiring. A new issue on `casehub-desiredstate` is required (see §Cross-Repo Dependencies). The dispatch layer must:

1. Observe `SituationChangeEvent` via CDI (`@ObservesAsync`)
2. On `ChangeType.TRIGGERED`: extract/construct `ActiveSituation`, call `SituationRecompilerEngine.recompile()`
3. On `ChangeType.RESOLVED`: call `SituationRecompilerEngine.situationResolved()` — immediate deactivation (see §Deactivation Path)
4. On non-empty result: call `LifecycleManager.updateDesired(tenancyId, result)`, then `ReconciliationLoop.requestReconciliation(tenancyId)`
5. On startup: query `SituationSource.activeSituations(tenancyId)` for cold-start recovery and feed existing situations through the engine

The deployment module's `DeploymentAdaptiveSituationRecompiler` is a pure SPI implementation — it does not observe CDI events or call reconciliation methods directly.

### External Health Checks

Applications can still emit health events directly for faster self-healing:

- Heartbeat timeout → `emit StateEvent(nodeId, DRIFTED, reason)`
- Foundation module deregistration → `emit StateEvent(nodeId, ABSENT, reason)`

These use real node IDs with real statuses — triggering immediate reconciliation rather than waiting for the periodic `ActualStateAdapter` check.

---

## Component 7: End-to-End Demo Scenario

### fsitrading Overnight Scenario

| Time | Event | System Response | Observable |
|---|---|---|---|
| T0 | App starts | `register(tenancyId, goals, clearanceWindows)` → base topology: 2 strategy, 1 risk, 1 audit agent, 3 channels, 2 trust policies. All provisioned. Dispatch layer queries `SituationSource` for cold-start — if situations already active, immediate recompilation adapts topology within the first reconciliation cycle. | 4 agents in DB |
| T1 | strategy-agent dies | `ActualStateAdapter`: ABSENT. Planner: PROVISION step. Re-provisioned. | Agent reappears. Ledger records fault + recovery. |
| T2 | Market volatility (price spikes) | Summarisation pipeline: STABLE → VOLATILE. Ganglion: `volatile-detected` → RAS situation `fsitrading.volatility-spike` (0.85). Recompiler: scale risk-agent to 3. | 3 risk agents. |
| T3 | Spread widening (anomaly) | Pipeline: VOLATILE → ANOMALOUS. Ganglion → `fsitrading.market-anomaly` (0.7). Recompiler: tighten trust 0.7 → 0.9. | Trust updated. Agents below 0.9 require human oversight. |
| T4 | Volatility resolves | Pipeline: VOLATILE → STABLE. Ganglion chain no longer sustained — no new `volatile-detected` signals. After TTL expires (30 min), RAS resolves `volatility-spike`. Recompiler clears tracked situation, recompiles from base — no scaling adaptation. risk-agent~3, risk-agent~2 deprovisioned (LIFO). | Back to 1 risk agent. |
| T5 | Breach detected | Ganglion → `fsitrading.active-breach` (0.99). Case created + recompiler: add forensics-agent + channel, tighten risk-assessment trust to 0.95. | 5 agents, new channel. Overnight-incident case active. |

### What Makes This Compelling

- Full pipeline: market tick → summarisation → situation detection → topology adaptation → reconciliation → provisioned agents
- `psql` into the database at any point — topology is real, durable, queryable
- Ledger has full audit trail
- Kill an agent row → watch it come back (self-healing, no FaultPolicy needed)
- No code change between fsitrading and SOC — different YAML, same runtime

---

## Runtime Changes Required

### casehub-ops (this repo)

**Already exist (this spec depends on them):**

1. **`AdaptationRuleSpec`** — in `io.casehub.ops.api.deployment`. Sealed interface with `ScaleActionSpec`, `AddActionSpec`, `UpdateActionSpec`. Includes parse-time validation.
2. **`AdaptationRule`** — in `io.casehub.ops.deployment.adaptation`. Runtime wrapper with `fromSpecs(specs, compiler, mapper, factory)` factory and `apply()` method.
3. **`DeploymentGoals.adaptations`** — `List<AdaptationRuleSpec>` field on the existing record. Jackson deserializes directly.
4. **`AgentNodeSpec.withAgentId(String)`** — copy method for scale-derived instances.

**New (introduced by this spec):**

5. **`DeploymentAdaptiveSituationRecompiler`** — implements `SituationRecompiler`, lives in `casehub-ops-deployment`.

**Modified (by this spec):**

6. **`TenantAdaptationState`** — exists in `io.casehub.ops.deployment.adaptation`. Adds situation tracking fields and methods; replaces `clearAbsentSituations(Set<String>)` with timestamp-based clearing. See §Per-Tenant State for the full modification list.

**Deleted (by this spec):**

7. **`AdaptiveTopologyManager`** — replaced by `DeploymentAdaptiveSituationRecompiler`. See §Migration.
8. **`StubSituationSource`** — `SituationSource` no longer needed. See §Migration.

### casehub-fsitrading

9. **`casehub-deployment.yaml`** — agent topology declaration.
10. **`fsitrading-market-monitoring.yaml`** — summarisation pipeline (L1 threshold-classify, L2 phase-detect).
11. **`FsiTradingEventTypes`** — CloudEvent type constants (`SIGNAL`, `CONDITION`, `SECURITY`).
12. **`FsiTradingSituationDefinitionProvider`** — 4 ganglia + 3 situation definitions.
13. **`MarketConditionCloudEventPublisher`** — bridges MarketPulse L3 output to RAS CloudEvents (`io.casehub.fsitrading.market.*`).
14. **`SecurityAlertBridge`** — emits `io.casehub.fsitrading.security.alert` CloudEvents when CRITICAL-severity security events occur. Consumed by the `breach-signal` ganglion.
15. **`FsiTradingSecurityEvent`** — domain-specific security event record (`Severity`, `source`, `detail`, `timestamp`). Fired as a CDI event by trust violation handlers, foundation module deregistration, and external SIEM integration.
16. **Bootstrap bean** — loads YAML, constructs `situationClearanceWindows` from `FsiTradingSituationDefinitionProvider.registrations()`, calls `recompiler.register(tenancyId, goals, situationClearanceWindows)`, then calls `LifecycleManager.start()` with the base topology. Cold-start recovery is handled by the dispatch layer in `casehub-desiredstate`.

---

## Cross-Repo Dependencies

| Repo | What's needed | Status |
|---|---|---|
| casehub-desiredstate-api | `SituationRecompiler` SPI, `CompilationResult` | **Exists** (desiredstate#49, closed) |
| casehub-desiredstate-api | `SituationRecompiler.situationResolved()` default method | **New** — cross-repo change required |
| casehub-desiredstate | `SituationRecompilerEngine`, `ReconciliationLoop.requestReconciliation()`, `LifecycleManager` | **Exists** (desiredstate#49, closed) |
| casehub-desiredstate | Dispatch layer: `SituationChangeEvent` → `SituationRecompilerEngine` wiring + cold-start recovery | **New** — cross-repo change required |
| casehub-ras | `ActiveSituation` record, `SituationDefinitionProvider`, `GanglionDescriptor`, `SituationRegistration`, `SituationSource`, `SituationChangeEvent` | **Exists** (ras-api 0.2-SNAPSHOT) |
| casehub-ops | `AdaptationRule` types, `DeploymentAdaptiveSituationRecompiler`, `DeploymentGoals.adaptations` field | **This issue** |
| casehub-fsitrading | Deployment YAML, summarisation pipeline, situation detectors, `FsiTradingSecurityEvent` | **This issue** |

**Cross-repo changes required:**

1. **casehub-desiredstate-api:** Add `situationResolved()` default method to `SituationRecompiler` interface. The default returns `Optional.empty()` — no existing recompiler is affected.

2. **casehub-desiredstate runtime:** Build the dispatch layer that bridges `SituationChangeEvent` CDI events to `SituationRecompilerEngine`. Responsibilities: TRIGGERED → `recompile()`, RESOLVED → `situationResolved()`, cold-start via `SituationSource`, result → `LifecycleManager.updateDesired()` + `ReconciliationLoop.requestReconciliation()`. See §Component 6 for the full specification.

## Non-Goals

- Custom health-check framework — ActualStateAdapter + periodic reconciliation handles drift
- Dynamic GoalCompiler replacing static YAML — base topology is always YAML-declared
- Cross-domain dependency graphs — each domain adapts independently (#23 tracks this). Note: `DeploymentAdaptiveSituationRecompiler` uses CDI-discovered bean registration. When `CrossDomainCompositionEngine` is active, `DesiredStateReplanDispatch.handleReplan()` uses `DomainRegistration.situationRecompilers()` instead. This is acceptable — cross-domain composition is explicitly out of scope.
- FaultPolicy-based self-healing — reconciliation loop handles re-provisioning
- SOC topology (#26) — separate issue, same patterns

## References

- `2026-06-29-adaptive-ops-design.md` — original spec (Components 1, 4-6 carried forward)
- `SituationRecompiler.class` in desiredstate-api 0.2-SNAPSHOT — actual SPI signature
- `CbrSituationRecompiler.class` in desiredstate runtime — reference implementation
- `ActiveSituation.class` in ras-api 0.2-SNAPSHOT — 8-field record
- `DeploymentTopologySituationDefinitionProvider.java` (ops #84) — Ganglion pattern reference
- `deployment-monitoring.yaml` (ops #84) — summarisation pipeline pattern reference
- `GE-20260706-eb11a1` — embedding desiredstate: @DefaultBean stubs
- `GE-20260806-272a90` — adding deployment node type: 6+4 pattern
- `GE-20260616-3d2605` — ReconciliationLoop CAS race with fault mutations
- `GE-20260806-55e158` — SituationRegistration null correlationKeyExtractor
- Protocol `deployment-spec-tenant-scoped` — tenancyId is required first-class field
- Protocol `engine-spi-noops-defaultbean` — @DefaultBean for SPI defaults
- Protocol `consumer-spi-placement` — SPI interfaces in api/spi/
