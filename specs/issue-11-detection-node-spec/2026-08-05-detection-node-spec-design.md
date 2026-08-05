# DetectionNodeSpec — Design Spec

**Issue:** casehubio/casehub-ops#11
**Date:** 2026-08-05
**Status:** Approved
**Review:** Light post-spec (coherence + structure + robustness + cross-cutting) — 0 dimension issues, 5 cross-cutting verifications resolved

---

## 1. Problem Statement

RAS situation definitions are hardcoded in `DesiredStateSituationDefinitionProvider`. Operators cannot declaratively add detection rules alongside their topology (agents, channels, endpoints). There is no way to express "this deployment should monitor for X" in a YAML goal file.

## 2. Scope

**In scope:**
- `DetectionNodeSpec` record — 6th deployment node type in `ops-api`
- `DeploymentGoals.detections` field
- `DeploymentGoalCompiler` wiring
- `DetectionProvisionHandler` — provision/deprovision via `SituationRegistrar`
- `DetectionDriftChecker` — existence check for `DeploymentActualStateAdapter`
- `DeploymentNodeProvisioner` — `"detection"` node type added
- Cross-repo: extract `SituationRegistrar` interface to `casehub-ras-api`

**Out of scope:**
- `GanglionNodeSpec` (future — separate node type for user-declared ganglia)
- Expression fields (correlationKeyExpression, eventFilter, dynamicCaseData) — addable as nullable fields later
- Changes to the system-layer `DesiredStateSituationDefinitionProvider`

## 3. Design Decisions

| # | Decision | Choice | Rationale |
|---|---|---|---|
| D1 | Layering model | System + user-declared | Hardcoded situations remain as platform defaults; DetectionNodeSpec adds domain-specific situations. Both feed `SituationDefinitionRegistry`. |
| D2 | Field surface | Mirror serializable SituationDefinition surface | Follows the established NodeSpec pattern (AgentNodeSpec mirrors AgentDescriptor). The serialization boundary naturally curates the subset — ExpressionEvaluator fields are runtime objects, not YAML-serializable. |
| D3 | Provisioner interaction | Imperative register/deregister | `SituationDefinitionRegistry` already has `register()` and `deregister()` methods (lines 186–212). Matches the existing provisioner pattern (AgentRegistry.register). |
| D4 | Dependency path | Extract SituationRegistrar to casehub-ras-api | Follows the pattern — ops modules depend on foundation APIs (eidos-api, platform-api), not runtimes. |
| D5 | Ganglion references | Reference only, no inline descriptors | Each DesiredNode is one thing with one lifecycle. Inlining ganglia would break shared-ganglion semantics (duplicate IDs) and require reference counting on deprovision. Future `GanglionNodeSpec` is the clean path. |

## 4. DetectionNodeSpec Record

```java
package io.casehub.ops.api.deployment;

@JsonIgnoreProperties(ignoreUnknown = true)
public record DetectionNodeSpec(
    String situationId,
    Set<String> eventTypes,
    Duration correlationWindow,
    Duration eventBufferDelay,
    ChainMode chainMode,
    TriggerAction triggerAction,
    TriggerMode triggerMode
) implements DeploymentNodeSpec {

    @Override public String nodeId() { return situationId; }
    @Override public String nodeType() { return "detection"; }

    public SituationRegistration toRegistration() {
        return new SituationRegistration(
            new SituationDefinition(situationId, eventTypes,
                correlationWindow, eventBufferDelay,
                chainMode, triggerAction, triggerMode),
            null);
    }
}
```

Validation: `situationId` required and non-blank, `eventTypes` non-empty, `chainMode` and `triggerAction` required. `eventBufferDelay` and `triggerMode` nullable (triggerMode defaults to FireOnce in SituationDefinition constructor).

## 5. SituationRegistrar Interface (casehub-ras-api)

```java
package io.casehub.ras.api;

public interface SituationRegistrar {
    void register(SituationRegistration registration);
    void deregister(String situationId);
    boolean exists(String situationId);
}
```

`SituationDefinitionRegistry` implements this — `register()` and `deregister()` already exist; `exists()` delegates to `findBySituationId(id).isPresent()`.

## 6. Goal Compiler Wiring

`DeploymentGoals` gains `List<GoalEntry<DetectionNodeSpec>> detections`. The compact constructor defaults it to `List.of()` (matching existing fields) for backward compatibility with YAML goals that don't include detections.

`DeploymentGoalCompiler.compile()` gains one line:
```java
compileEntries(goals.detections(), nodes, dependencies);
```

The existing generic `compileEntries()` handles it — no new compiler code beyond the call.

## 7. Provisioner

`DetectionProvisionHandler` — provision calls `registrar.register(spec.toRegistration())`, deprovision calls `registrar.deregister(spec.situationId())`.

`DeploymentNodeProvisioner`:
- Constructor gains `SituationRegistrar` injection
- `handledTypes()` adds `NodeType.of("detection")`
- `doProvision()`/`doDeprovision()` switch gains `case DetectionNodeSpec` arms

## 8. Drift Checker

`DetectionDriftChecker implements NodeDriftChecker`:
- `nodeType()` returns `"detection"`
- `check()` returns PRESENT if `registrar.exists(situationId)`, ABSENT otherwise
- Layer 2 spec-hash drift handled by existing `SpecHashStore` in `DeploymentActualStateAdapter`

`DeploymentActualStateAdapter.handledTypes()` adds `NodeType.of("detection")`.

## 9. YAML Goal Example

```yaml
detections:
  - spec:
      situationId: "app.repeated-agent-failure"
      eventTypes: ["desiredstate.node.faulted", "desiredstate.node.recovered"]
      correlationWindow: "PT10M"
      chainMode:
        type: "streak"
        ganglionId: "node-fault"
        threshold: 5
      triggerAction:
        type: "createCase"
        config:
          domain: "ops"
          caseType: "incident-response"
          version: "1.0"
      triggerMode:
        type: "fireOnce"
    dependsOn: []
```

## 10. Testing

| Test | What it verifies |
|---|---|
| DetectionNodeSpecTest | Value semantics, validation, toRegistration(), JSON round-trip |
| DeploymentGoalCompilerTest | Detection entries compile to DesiredNodes with correct NodeType |
| DetectionProvisionHandlerTest | Provision calls register(), deprovision calls deregister() |
| DetectionDriftCheckerTest | PRESENT/ABSENT/UNKNOWN for exists() results |
| DeploymentNodeProvisionerTest | Detection case in provision/deprovision switch |
| DeploymentActualStateAdapterTest | Detection nodes with stub drift checker |
| SituationRegistrarTest (cross-repo) | Registry implements interface, exercises register/deregister/exists |

## 11. Verified Integration Points

Cross-cutting review flagged 5 areas for code-level verification. All resolved:

| Concern | Verification | Status |
|---|---|---|
| `toRegistration()` passes `null` extractor to `SituationRegistration` | Existing `DesiredStateSituationDefinitionProvider` passes `null` on 2 of 3 registrations — valid and expected for default correlation key | Verified |
| `"detection"` nodeType collision | No existing DeploymentNodeSpec returns `"detection"` — checked all 5 specs | Verified |
| `SpecHashStore` coverage for DetectionNodeSpec | `SpecHashStore` is generic on `NodeId` + `NodeSpec` — works for any spec type automatically | Verified |
| Cross-repo sequencing — API/runtime match | `SituationRegistrar` extracts methods already on `SituationDefinitionRegistry` — additive, non-breaking | Verified |
| `DeploymentGoals` backward compat with missing `detections` | `@JsonIgnoreProperties(ignoreUnknown = true)` + compact constructor defaulting to `List.of()` — Jackson handles missing fields | Verified |

## 12. Cross-Repo Sequencing

1. `casehub-ras-api` — add `SituationRegistrar` interface
2. `casehub-ras` — `SituationDefinitionRegistry implements SituationRegistrar`, add `exists()`
3. `casehub-ops/api` — `DetectionNodeSpec`, update `DeploymentNodeSpec` permits, update `DeploymentGoals`
4. `casehub-ops/deployment` — handler, drift checker, provisioner/adapter/compiler updates

## 12. Future Extensions

- **GanglionNodeSpec** — 7th deployment node type for user-declared ganglia, connected to DetectionNodeSpec via `dependsOn` edges
- **Expression fields** — `correlationKeyExpression`, `eventFilter`, `dynamicCaseData` as nullable String fields compiled to ExpressionEvaluator at provision time
- **Ganglion descriptors on DetectionNodeSpec** — once GanglionNodeSpec exists, evaluate whether inline descriptors add value as syntactic sugar
