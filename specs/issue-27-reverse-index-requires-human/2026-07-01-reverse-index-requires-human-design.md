# Design: Reverse Index (#27) + requiresHuman Unification (#28)

## Problem

Two independent issues addressed together:

**#27 — Reverse index for declaredAgentIds.** `DeploymentProvisionerConfigRegistry.declaredAgentIds(providerName)` iterates all agents via `store.agentIds()` and filters by providerName — O(n). Add a reverse index in the store.

**#28 — requiresHuman + HumanNodeHandler.** The platform has four organic approaches to `requiresHuman` with no unified contract. GoalCompilers hardcode `false` or use domain-specific logic. IoTNodeProvisioner has a dead provision branch and a live deprovision bug.

## Analysis: requiresHuman from First Principles

The desired-state runtime has two orthogonal approval mechanisms:

| Mechanism | Question | Timing | Decision point |
|---|---|---|---|
| `requiresHuman` | Can this be automated? | Compile-time | Structural — inherent to the node |
| `PendingApproval` | Should this be automated now? | Runtime | Dynamic — risk/policy per cycle |

Four organic approaches evolved independently:

| Approach | Source of truth | When valid |
|---|---|---|
| Goal-entry field (dungeon) | YAML author declares it | Same spec type can go either way by author choice |
| Spec-instance field (compliance) | Spec data carries it | Same type, varies by instance properties |
| Compiler logic (IoT) | Compiler domain rules | Approach 4 in disguise — IoT already splits spec types |
| Separate spec type (pipeline) | Type system | Human requirement correlates with type |

All three valid approaches (2, 3, 4 — approach 3 being a variant of 4) answer the same question at different levels. They compose via OR:

```
requiresHuman = spec.requiresHuman()   // type-level OR instance-level
             || entry.requiresHuman()  // author opt-in (if present, future)
```

## Design: Unified `NodeSpec.requiresHuman()`

### Cross-repo: casehub-desiredstate-api

Add a default method to the marker interface:

```java
public interface NodeSpec {
    default boolean requiresHuman() { return false; }
}
```

All existing implementations inherit `false`. Specs that are structurally non-automatable override.

### casehub-ops spec overrides

**PhysicalDeviceSpec** — type-level, always true:
```java
@Override public boolean requiresHuman() { return true; }
```

**ComplianceControlSpec** — instance-level, bridges domain field to generic contract:
```java
@Override public boolean requiresHuman() { return requiresHumanReview(); }
```

`requiresHumanReview` stays as the record field (domain-specific name). The override satisfies the generic contract.

**InfraDesiredNodeSpec** — wrapper delegation:
```java
@Override public boolean requiresHuman() { return resourceSpec.requiresHuman(); }
```

Without this, the wrapper swallows the inner spec's answer.

### GoalCompiler unification

All four compilers use the same pattern:

```java
nodes.add(new DesiredNode(id, type, spec, spec.requiresHuman()));
```

| Compiler | Before | After |
|---|---|---|
| DeploymentGoalCompiler | hardcoded `false` | `spec.requiresHuman()` |
| InfraGoalCompiler | hardcoded `false` | `wrapper.requiresHuman()` (delegates to inner) |
| ComplianceGoalCompiler | `spec.requiresHumanReview()` | `spec.requiresHuman()` (equivalent via override) |
| IoTGoalCompiler | conditional `true`/`false` from `goal.physical()` | `spec.requiresHuman()` (PhysicalDeviceSpec intrinsically true) |

### IoTNodeProvisioner cleanup

**`doProvision`** — remove `PhysicalDeviceSpec` case. Dead code: `SimpleTransitionExecutor` intercepts `requiresHuman=true` nodes before the provisioner is called.

**`doDeprovision`** — change `PhysicalDeviceSpec` case from `Failed("physical devices cannot be auto-deprovisioned")` to `Success()`. Deprovisioning means "stop managing" — no physical action required, no failure.

### Reverse index (#27)

**DeploymentProviderConfigStore** — add reverse index:

```java
private final ConcurrentHashMap<String, Set<String>> providerToAgents = new ConcurrentHashMap<>();
```

Maintained atomically in `store()` (add agent to provider sets; remove from stale provider sets on config change) and `remove()` (remove agent from all provider sets). Exposed via:

```java
public Set<String> agentIdsForProvider(String providerName)
```

**DeploymentProvisionerConfigRegistry.declaredAgentIds** — delegates to `store.agentIdsForProvider(providerName)` instead of streaming/filtering.

## Pattern guidance

| Pattern | When to use | Example |
|---|---|---|
| Override `requiresHuman()` → `return true` | Every instance of this type always needs human | `PhysicalDeviceSpec` |
| Override with instance field | Same type, varies by instance data | `ComplianceControlSpec` |
| Goal-entry `requiresHuman` field, OR'd with spec | Same type+data, varies by author intent | Not needed in ops today |

## Deferred (filed as issues)

- desiredstate#54: Add `requiresHuman` check to `executeDeprovision` + `onDeprovision()` to `HumanNodeHandler`
- desiredstate#55: Update dungeon example to use spec-level `requiresHuman()` pattern

## Files changed

### casehub-desiredstate (cross-repo)
- `api/.../NodeSpec.java` — add `default boolean requiresHuman()`

### casehub-ops
- `api/.../iot/PhysicalDeviceSpec.java` — override `requiresHuman()` → true
- `api/.../compliance/ComplianceControlSpec.java` — override `requiresHuman()` → `requiresHumanReview()`
- `api/.../infra/InfraDesiredNodeSpec.java` — override `requiresHuman()` → delegate to inner spec
- `deployment/.../DeploymentGoalCompiler.java` — `false` → `spec.requiresHuman()`
- `infra/.../InfraGoalCompiler.java` — `false` → `wrapper.requiresHuman()`
- `compliance/.../ComplianceGoalCompiler.java` — `spec.requiresHumanReview()` → `spec.requiresHuman()`
- `iot/.../IoTGoalCompiler.java` — hardcoded booleans → `spec.requiresHuman()`
- `iot/.../IoTNodeProvisioner.java` — remove dead provision branch, fix deprovision to Success
- `deployment/.../DeploymentProviderConfigStore.java` — add reverse index
- `deployment/.../DeploymentProvisionerConfigRegistry.java` — use reverse index
- Tests for all changes
