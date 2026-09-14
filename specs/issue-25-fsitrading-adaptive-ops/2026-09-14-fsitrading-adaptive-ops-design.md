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
| Unplanned response (node failure) | ReconciliationLoop | Re-provisions ABSENT nodes automatically; `FaultPolicy` returns `List.of()` |

**Layer separation:**

- **`SituationRecompiler` SPI** in `casehub-desiredstate-api` — domain-agnostic contract. The runtime pushes each `ActiveSituation` (from `casehub-ras-api`) to all registered recompilers. Already exists (desiredstate#49, closed).
- **`DeploymentAdaptiveSituationRecompiler`** in `casehub-ops-deployment` — deployment-domain-specific implementation that applies YAML-defined adaptation rules.
- **Situation detection** in `casehub-fsitrading` — Ganglion detectors + summarisation pipeline for market conditions. Follows the #84 pattern.
- **`ReconciliationLoop` unchanged** — `start()`, `updateDesired()`, `requestReconciliation()` all exist and work as designed.

---

## Component 1: Adaptive YAML Format

*Unchanged from original spec.* The deployment YAML gains an `adaptations:` section:

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

### NodeId Collision Validation

At parse time: (1) `target` node ID must not contain `~`, (2) no base topology node may match `{target}~{n}` for any n in [2, max].

### Hysteresis and Cooldown

- **`deactivateBelow`** — deactivates when confidence drops below this. Defaults to `minConfidence`.
- **`cooldown`** — ISO 8601 duration. Minimum time between state changes. Defaults to `PT0S`.

### Conflict Detection

Multiple simultaneously active adaptations modifying the same node: YAML declaration order defines precedence — later rules override earlier ones. Warning logged.

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
}
```

The runtime's `SituationRecompilerEngine` pushes each `ActiveSituation` (from `io.casehub.ras.api`) to all registered `SituationRecompiler` instances by priority. The deployment module provides one implementation:

```java
@ApplicationScoped
public class DeploymentAdaptiveSituationRecompiler implements SituationRecompiler {

    @Inject DeploymentGoalCompiler compiler;
    @Inject DesiredStateGraphFactory graphFactory;

    private final ConcurrentHashMap<String, TenantAdaptationState> tenantStates =
        new ConcurrentHashMap<>();

    @Override
    public int priority() {
        return 100; // Domain-specific, higher than CBR's default
    }

    public void register(String tenancyId, DeploymentGoals goals) {
        List<AdaptationRule> rules = AdaptationRule.fromSpecs(goals.adaptations());
        tenantStates.put(tenancyId, new TenantAdaptationState(goals, rules));
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

            if (adapted.equals(base)) {
                return Optional.empty();
            }
            return Optional.of(CompilationResult.single(adapted));
        }
    }
}
```

### Per-Tenant State

`TenantAdaptationState` holds:
- The tenant's `DeploymentGoals` (base topology)
- Parsed `List<AdaptationRule>` (from `goals.adaptations()`)
- Tracked active situations: `Map<String, ActiveSituation>` keyed by `situationId`
- Hysteresis state: `activePerRule` and `lastChangePerRule`, keyed by rule name

### Situation Tracking

The `SituationRecompiler` is called per-situation. To apply all active situations during recompilation:

1. `updateSituation(situation)` — upserts the situation into the tracked set (keyed by `situationId`)
2. `activeSituationFor(rule)` — looks up the tracked situation matching `rule.trigger().situation()`
3. `clearAbsentSituations()` — after processing, mark situations as absent if their `lastSignal` is older than the resync window (5 minutes). Absent situations with cooldown are retained until cooldown expires.

This ensures the recompiler has a complete view of all active situations, not just the one being pushed in this call.

### Hysteresis and Cooldown

Same logic as the original spec — `shouldActivate()` handles hysteresis band and cooldown checks. Lives inside `TenantAdaptationState` instead of `AdaptiveTopologyManager`.

### Registration

When the deployment app bootstraps, it calls `register(tenancyId, goals)` to provide the base topology and adaptation rules. This happens before `ReconciliationLoop.start()`.

### Key Differences from Original Spec

| Original (AdaptiveTopologyManager) | Updated (SituationRecompiler) |
|---|---|
| CDI observer of `SituationChangeEvent` | SPI implementation — runtime pushes situations |
| Pulls situations from `SituationSource` | Receives `ActiveSituation` directly |
| Calls `updateDesired()` and `requestReconciliation()` | Returns `CompilationResult` — runtime handles graph update |
| Periodic re-poll safety net | Not needed — runtime handles delivery |
| `SituationSource` SPI in desiredstate-api | Not needed — `SituationRecompiler` already exists |
| `SituationChangeEvent` CDI event | Not needed — runtime orchestrates |

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
| `strategy-agent` / `strategy-agent-2` | Evaluate trading strategies, overnight coverage | 2 (base) |
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
        cloud-event-type: io.casehub.fsitrading.market.signal

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
        cloud-event-type: io.casehub.fsitrading.market.condition
```

### Event Types

```java
public final class FsiTradingEventTypes {
    public static final String SIGNAL = "io.casehub.fsitrading.market.signal";
    public static final String CONDITION = "io.casehub.fsitrading.market.condition";
}
```

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
            ganglion("volatile-detected", FsiTradingEventTypes.CONDITION,
                ctx -> "VOLATILE".equals(data(ctx).get("to")), 0.8),
            ganglion("anomalous-detected", FsiTradingEventTypes.CONDITION,
                ctx -> "ANOMALOUS".equals(data(ctx).get("to")), 0.95),
            ganglion("stable-detected", FsiTradingEventTypes.CONDITION,
                ctx -> "STABLE".equals(data(ctx).get("to")), 0.9),
            ganglion("breach-signal", FsiTradingEventTypes.SIGNAL,
                ctx -> "BREACH".equals(data(ctx).get("category")), 0.99)
        );
    }

    @Override
    public List<SituationRegistration> registrations() {
        return List.of(
            // volatility-spike: sustained volatile state (2 consecutive detections)
            new SituationRegistration(
                new SituationDefinition(
                    VOLATILITY_SPIKE,
                    Set.of(FsiTradingEventTypes.CONDITION),
                    Duration.ofMinutes(30),
                    null,
                    new ChainMode.Streak("volatile-detected", 2),
                    new TriggerAction.Noop(),  // situation only — no case creation
                    new TriggerMode.Continuous()),
                null),

            // market-anomaly: anomalous state detected
            new SituationRegistration(
                new SituationDefinition(
                    MARKET_ANOMALY,
                    Set.of(FsiTradingEventTypes.CONDITION),
                    Duration.ofMinutes(15),
                    null,
                    new ChainMode.Single("anomalous-detected"),
                    new TriggerAction.Noop(),
                    new TriggerMode.Continuous()),
                null),

            // active-breach: immediate, high-confidence
            new SituationRegistration(
                new SituationDefinition(
                    ACTIVE_BREACH,
                    Set.of(FsiTradingEventTypes.SIGNAL),
                    Duration.ofHours(2),
                    null,
                    new ChainMode.Single("breach-signal"),
                    new TriggerAction.CreateCase(
                        new CaseTriggerConfig("fsitrading", "overnight-incident",
                            "1.0", Map.of())),
                    new TriggerMode.FireOnce()),
                null)
        );
    }
}
```

### Situation Semantics

| Situation | Trigger | TTL | Action | Adaptation |
|---|---|---|---|---|
| `fsitrading.volatility-spike` | 2 consecutive VOLATILE phase detections | 30 min | None (situation only) | Scale risk agents 1→5 |
| `fsitrading.market-anomaly` | Single ANOMALOUS phase detection | 15 min | None (situation only) | Tighten trade-execution trust to 0.9 |
| `fsitrading.active-breach` | Single breach signal | 2 hours | Create overnight-incident case | Add forensics agent + channel, tighten risk-assessment trust to 0.95 |

`TriggerAction.Noop()` for volatility-spike and market-anomaly — these situations exist purely to drive topology adaptation via `SituationRecompiler`. No case is created. `active-breach` creates an overnight-incident case AND drives adaptation.

---

## Component 5: FaultPolicy — No-Op

*Unchanged from original spec.* Self-healing is built into the reconciliation cycle:

1. `ActualStateAdapter.readActual()` reports `NodeStatus.ABSENT`
2. `TransitionPlanner.plan()` generates a PROVISION step
3. The node is re-provisioned automatically

```java
@ApplicationScoped
public class DeploymentFaultPolicy implements FaultPolicy {
    @Override
    public List<GraphMutation<DesiredNode>> onFault(FaultEvent event,
            DesiredStateGraph current) {
        return List.of();
    }
}
```

---

## Component 6: Reconciliation Triggering

Simpler than the original spec. The runtime handles situation delivery via `SituationRecompilerEngine`:

1. RAS detects a situation (e.g., `fsitrading.volatility-spike`)
2. Runtime pushes `ActiveSituation` to `DeploymentAdaptiveSituationRecompiler.recompile()`
3. Recompiler returns adapted graph via `CompilationResult.single()`
4. Runtime calls `updateDesired()` and triggers reconciliation

**No custom wiring needed.** `requestReconciliation()` exists but is called by the runtime, not the domain module. No `SituationChangeEvent` CDI events. No periodic re-poll — the runtime handles delivery reliability.

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
| T0 | App starts | `register(tenancyId, goals)` → base topology: 2 strategy, 1 risk, 1 audit agent, 3 channels, 2 trust policies. All provisioned. | 4 agents in DB |
| T1 | strategy-agent dies | `ActualStateAdapter`: ABSENT. Planner: PROVISION step. Re-provisioned. | Agent reappears. Ledger records fault + recovery. |
| T2 | Market volatility (price spikes) | Summarisation pipeline: STABLE → VOLATILE. Ganglion: `volatile-detected` × 2 → RAS situation `fsitrading.volatility-spike` (0.85). Recompiler: scale risk-agent to 3. | 3 risk agents. |
| T3 | Spread widening (anomaly) | Pipeline: VOLATILE → ANOMALOUS. Ganglion → `fsitrading.market-anomaly` (0.7). Recompiler: tighten trust 0.7 → 0.9. | Trust updated. Agents below 0.9 require human oversight. |
| T4 | Volatility resolves | Pipeline: VOLATILE → STABLE. Situation TTL expires. Recompiler: no scaling adaptation. Risk-agent~3, risk-agent~2 deprovisioned (LIFO). | Back to 1 risk agent. |
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

1. **`AdaptationRule`** — new types in `io.casehub.ops.api.deployment`: `AdaptationRuleSpec`, `AdaptationRule` with `ScaleAction`, `AddAction`, `UpdateAction`. Includes `fromSpecs()` factory and parse-time validation.
2. **`DeploymentGoals.adaptations`** — new field (`List<AdaptationRuleSpec>`) on the existing record. Jackson deserializes directly. `@JsonIgnoreProperties(ignoreUnknown = true)` ensures backward compat.
3. **`AgentNodeSpec.withAgentId(String)`** — copy method for scale-derived instances.
4. **`DeploymentAdaptiveSituationRecompiler`** — implements `SituationRecompiler`, lives in `casehub-ops-deployment`.
5. **`TenantAdaptationState`** — per-tenant state: goals, rules, tracked situations, hysteresis.

### casehub-fsitrading

6. **`casehub-deployment.yaml`** — agent topology declaration.
7. **`fsitrading-market-monitoring.yaml`** — summarisation pipeline (L1 threshold-classify, L2 phase-detect).
8. **`FsiTradingEventTypes`** — CloudEvent type constants.
9. **`FsiTradingSituationDefinitionProvider`** — 4 ganglia + 3 situation definitions.
10. **Bootstrap bean** — loads YAML, calls `recompiler.register()`, calls `reconciliationLoop.start()`.

---

## Cross-Repo Dependencies

| Repo | What's needed | Status |
|---|---|---|
| casehub-desiredstate | `SituationRecompiler` SPI, `CompilationResult`, `requestReconciliation()` | **Exists** (desiredstate#49, closed) |
| casehub-ras | `ActiveSituation` record, `SituationDefinitionProvider`, `GanglionDescriptor`, `SituationRegistration` | **Exists** (ras-api 0.2-SNAPSHOT) |
| casehub-ops | `AdaptationRule` types, `DeploymentAdaptiveSituationRecompiler`, `DeploymentGoals.adaptations` field | **This issue** |
| casehub-fsitrading | Deployment YAML, summarisation pipeline, situation detectors | **This issue** |

No cross-repo changes required — all dependencies already exist.

## Non-Goals

- Custom health-check framework — ActualStateAdapter + periodic reconciliation handles drift
- Dynamic GoalCompiler replacing static YAML — base topology is always YAML-declared
- Cross-domain dependency graphs — each domain adapts independently (#23 tracks this)
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
