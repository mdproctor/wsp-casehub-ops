# Decisions — Issue #25: fsitrading adaptive ops

## D1: Spec update strategy

**Choice:** Update existing spec (2026-06-29-adaptive-ops-design.md)
**Alternatives:**
- Write fresh spec — unnecessary, Components 1, 4-6 unchanged
- Skip spec update, plan directly — too much API divergence to skip documentation
**Rationale:** The core adaptation logic (YAML format, actions, hysteresis, demo scenario) is unchanged. Only the integration point changed: SituationRecompiler replaced SituationSource/AdaptiveTopologyManager.
**Trade-offs:** May miss design issues that a fresh spec would surface
**Sources:** desiredstate-api 0.2-SNAPSHOT bytecode (SituationRecompiler.class), desiredstate#49 (closed)
**Exploration:** quick
**Status:** captured

## D2: Implementation scope

**Choice:** Full stack including fsitrading RAS
**Alternatives:**
- Ops recompiler + YAML only — doesn't demonstrate the end-to-end adaptive pipeline
- Ops + fsitrading YAML + demo app — misses the RAS integration path which is the novel contribution
**Rationale:** The value proposition is the full pipeline: market signal → RAS situation detection → topology adaptation → reconciliation. Stubbing situations weakens the demo significantly.
**Trade-offs:** Significantly larger scope (L/High). Requires Ganglion implementation work in fsitrading repo.
**Sources:** issue #25 body (scope items 1-5), casehub-fsitrading repo state (zero deployment topology, rich domain model)
**Exploration:** quick
**Status:** captured

## D3: Recompilation model — multi-situation composition

**Choice:** Track active situations internally, recompile from base with all active situations
**Alternatives:**
- Incremental per-situation — simpler but ordering-dependent, violates "never patch incrementally" principle
- Single-situation only — doesn't support concurrent situations (volatility-spike + market-anomaly)
**Rationale:** Matches original spec's principle: "Never incrementally patches the adapted graph. Base → adaptations → result." Consistent regardless of situation delivery order.
**Trade-offs:** Stateful recompiler — maintains per-tenant active-situation set and hysteresis state. More complex than stateless approach.
**Sources:** 2026-06-29-adaptive-ops-design.md §Component 3 (re-compiles from base each time), SituationRecompiler API signature
**Exploration:** quick
**Depends on:** D1 (spec update — inherits original spec's design decisions)
**Status:** captured

## D4: Situation detection sophistication

**Choice:** Full Ganglion with summarisation-enhanced RAS detection
**Alternatives:**
- Threshold-based stubs — demonstrates pipeline but not realistic
- In-memory test stubs only — doesn't demonstrate RAS integration at all
**Rationale:** #84 already delivered the summarisation pipeline pattern for deployment monitoring. fsitrading situations follow the same pattern — YAML pipeline + Ganglion detectors + SituationDefinitionProvider. The pattern is proven; applying it to market signals is the natural extension.
**Trade-offs:** Requires YAML summarisation pipeline design for market events, plus Ganglion detection logic. More implementation work than threshold stubs.
**Sources:** #84 commit (012e4fd), DeploymentTopologySituationDefinitionProvider.java, deployment-monitoring.yaml
**Depends on:** D2 (full stack scope includes RAS detectors)
**Exploration:** quick
**Status:** captured
