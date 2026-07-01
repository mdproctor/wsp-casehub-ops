# Design: Reverse Index (#27) + requiresHuman Unification (#28)

## Problem

Two independent issues addressed together — both are small (XS/S) changes in the same project, and separate specs doubles review overhead with no architectural benefit:

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

Override `DesiredNode.requiresHuman()` to compose both sources via OR:

```java
public record DesiredNode(NodeId id, NodeType type, NodeSpec spec, boolean requiresHuman) {
    @Override
    public boolean requiresHuman() {
        return requiresHuman || spec.requiresHuman();
    }
}
```

This makes the record field an author-level override channel and `NodeSpec.requiresHuman()` the type/instance-level baseline. The composition is monotonic — once true from either source, the node requires human intervention. GoalCompilers pass author-level overrides (or `false` when none exist); they don't need to call `spec.requiresHuman()`.

ASSUMPTION: Java records permit overriding auto-generated accessor methods (JLS §8.10.3 — compiler generates the accessor only if not explicitly declared).

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

With OR composition in `DesiredNode`, spec-level `requiresHuman` is automatically included regardless of what the compiler passes. The record field is reserved for author-level overrides (goal-entry `requiresHuman`). No compilers use author-level overrides today:

| Compiler | Before | After |
|---|---|---|
| DeploymentGoalCompiler | hardcoded `false` | `false` (unchanged — spec composes via OR) |
| InfraGoalCompiler | hardcoded `false` | `false` (unchanged — spec composes via OR) |
| ComplianceGoalCompiler | `spec.requiresHumanReview()` | `false` (`ComplianceControlSpec.requiresHuman()` composes via OR) |
| IoTGoalCompiler | conditional `true`/`false` from `goal.physical()` | `false` (`PhysicalDeviceSpec.requiresHuman()` composes via OR) |

Deployment and Infra compilers are unchanged. Compliance and IoT compilers simplify — the domain-specific logic moves from the compiler to the spec where it belongs.

### IoTNodeProvisioner cleanup

**`doProvision`** — remove `PhysicalDeviceSpec` case. Dead code: `SimpleTransitionExecutor` intercepts `requiresHuman=true` nodes before the provisioner is called.

**`doDeprovision`** — change `PhysicalDeviceSpec` case from `Failed("physical devices cannot be auto-deprovisioned")` to `Success()`. Deprovisioning means "stop managing" — no physical action required, no failure. The current `Failed` return creates infinite fault feedback cycles on every reconciliation.

**Interim gap:** `SimpleTransitionExecutor.executeDeprovision()` does not check `requiresHuman` — it proceeds directly to approval check and then to the provisioner. Until desiredstate#54 adds `requiresHuman` gating to deprovision, physical device removal from YAML will deprovision silently (no human notification). This is acceptable: (a) `Success()` is semantically correct at the provisioner level, (b) the `Failed` alternative is worse (infinite retry/fault cycle), and (c) desiredstate#54 is the proper fix at the executor level.

### Reverse index (#27)

**DeploymentProviderConfigStore** — add reverse index:

```java
private final ConcurrentHashMap<String, Set<String>> providerToAgents = new ConcurrentHashMap<>();
```

Inner sets use `ConcurrentHashMap.newKeySet()` for thread-safe concurrent modification from parallel `store()` calls.

Updated in `store()` (add agent to provider sets; remove from stale provider sets on config change) and `remove()` (remove agent from all provider sets). The reverse index provides **weakly consistent** reads — the same consistency model as the current linear scan over `ConcurrentHashMap` keys. A concurrent reader may briefly see an agent in both old and new provider sets (or neither) during a provider change. This is acceptable for the lookup's purpose (provider initialization).

Exposed via:

```java
public Set<String> agentIdsForProvider(String providerName)
```

Returns `Set.copyOf(set)` — an immutable point-in-time snapshot. Callers never observe internal concurrent modification.

**DeploymentProvisionerConfigRegistry.declaredAgentIds** — delegates to `store.agentIdsForProvider(providerName)` instead of streaming/filtering.

## Pattern guidance

| Pattern | When to use | Example |
|---|---|---|
| Override `requiresHuman()` → `return true` | Every instance of this type always needs human | `PhysicalDeviceSpec` |
| Override with instance field | Same type, varies by instance data | `ComplianceControlSpec` |
| Goal-entry `requiresHuman` field, OR'd with spec | Same type+data, varies by author intent | Not needed in ops today |

## Issue #28 requirements mapping

| # | Requirement | Status | Notes |
|---|---|---|---|
| 1 | GoalEntry/ResourceDeclaration: add `requiresHuman` field | Deferred | Not needed — OR composition in `DesiredNode` means the record field is the author-override channel, populated from goal entries when that pattern is needed |
| 2 | GoalCompilers: propagate `requiresHuman` | Superseded | OR composition makes compiler propagation unnecessary — `spec.requiresHuman()` is composed automatically |
| 3 | `HumanNodeHandler` implementation | Already done | `WorkItemHumanNodeHandler` in `casehub-desiredstate/work-adapter` |
| 4 | IoT fix: `PhysicalDeviceSpec` `requiresHuman=true` | Addressed | Via `PhysicalDeviceSpec.requiresHuman()` override + OR composition |

## Deferred (filed as issues)

- desiredstate#54: Add `requiresHuman` check to `executeDeprovision` + `onDeprovision()` to `HumanNodeHandler`
- desiredstate#55: Update dungeon example to use spec-level `requiresHuman()` pattern

## Files changed

### casehub-desiredstate (cross-repo)
- `api/.../NodeSpec.java` — add `default boolean requiresHuman()`
- `api/.../DesiredNode.java` — override `requiresHuman()` with OR composition (`requiresHuman || spec.requiresHuman()`)

### casehub-ops
- `api/.../iot/PhysicalDeviceSpec.java` — override `requiresHuman()` → true
- `api/.../compliance/ComplianceControlSpec.java` — override `requiresHuman()` → `requiresHumanReview()`
- `api/.../infra/InfraDesiredNodeSpec.java` — override `requiresHuman()` → delegate to inner spec
- `compliance/.../ComplianceGoalCompiler.java` — `spec.requiresHumanReview()` → `false` (spec composes via OR)
- `iot/.../IoTGoalCompiler.java` — conditional `true`/`false` → `false` (spec composes via OR)
- `iot/.../IoTNodeProvisioner.java` — remove dead provision branch, fix deprovision to Success
- `deployment/.../DeploymentProviderConfigStore.java` — add reverse index
- `deployment/.../DeploymentProvisionerConfigRegistry.java` — use reverse index
- Tests for all changes
