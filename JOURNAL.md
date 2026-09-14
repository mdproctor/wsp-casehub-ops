# Design Journal — issue-25-fsitrading-adaptive-ops

## 2026-09-14 — Session 1: Design + Batch 1 implementation

### Key discovery: SituationRecompiler API diverged from spec

The original spec (2026-06-29) proposed `SituationSource` (pull-based) + `AdaptiveTopologyManager` (CDI observer). The actual API delivered by desiredstate#49 is `SituationRecompiler` (push-based) — the runtime pushes each `ActiveSituation` to domain recompilers. Cleaner design: the domain module is a pure SPI implementation, no CDI event wiring needed.

### Design decisions

- **D1:** Update existing spec (not fresh) — Components 1, 4-6 unchanged
- **D2:** Full stack scope — ops recompiler + fsitrading RAS with summarisation-enhanced Ganglion detectors
- **D3:** Track active situations internally, recompile from base every time (matches original "never patch incrementally" principle)
- **D4:** Full Ganglion with summarisation — follows #84 pattern
- **Late addition:** YAML-based situation definitions — `YamlSituationDefinitionProvider` generic base class instead of hand-coded Java

### Design review

Standard review (4 dimensions, 3 rounds each): 69 issues found, 0 unresolved, $79. Key findings:
- RAS sealed variant names (`Count` not `Single`, `NotifyOnly` not `Noop`, `Repeating` not `Continuous`)
- `FaultPolicy.onFault()` has 4 params not 2 in current API
- `ImmutableDesiredStateGraph` has no `equals()` override — structural comparison needed
- STABLE→ANOMALOUS direct transition missing in original pipeline design

### Implementation — Batch 1 complete

1. `TenantAdaptationState` extended with situation tracking (7 tests)
2. `DeploymentAdaptiveSituationRecompiler` implementing `SituationRecompiler` at priority 100 (6 tests)
3. `AdaptiveTopologyManager` + `StubSituationSource` removed (1111 lines deleted, 203 tests pass)

Also fixed pre-existing `AgentCapability` API mismatch across 6 test files.

### What's next — Batches 2-4

All remaining work is in fsitrading repo:
- Batch 2: Event types, security bridge, CloudEvent publisher
- Batch 3: Summarisation pipeline YAML, `YamlSituationDefinitionProvider` (ops), situation definitions YAML (fsitrading)
- Batch 4: Deployment YAML, bootstrap bean, end-to-end integration test
