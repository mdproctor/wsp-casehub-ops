---
layout: post
title: "Four Paths to the Same Boolean"
date: 2026-07-01
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ops]
tags: [desiredstate, requiresHuman, unification, architecture]
series: issue-27-reverse-index-requires-human
---

The desiredstate runtime has two approval mechanisms: `requiresHuman` (compile-time — the node structurally cannot be automated) and `PendingApproval` (runtime — the provisioner evaluates risk and decides). These are orthogonal by design and need to stay that way.

The problem was that `requiresHuman` had four different implementations, each evolved independently. The dungeon example put it on the goal entry — the YAML author declares it. Compliance put it on the spec instance — `ComplianceControlSpec.requiresHumanReview()`. IoT inferred it from compiler logic — `goal.physical()` triggers `requiresHuman=true`. The pipeline example used a separate spec type — `HumanReviewSpec` IS the signal.

I traced each approach to its root. The key variable is what determines the need: the spec TYPE (all instances always require human), the spec INSTANCE DATA (same type, some do, some don't), or the AUTHOR'S CHOICE (same instance could go either way). Each organic approach was answering the same question at a different level. The compiler-logic approach (IoT) turned out to be the type approach in disguise — IoT already splits into `PhysicalDeviceSpec` and `DeviceConfigSpec`, the compiler just hadn't named what it was doing.

The unified mechanism is a default method on `NodeSpec`:

```java
public interface NodeSpec {
    default boolean requiresHuman() { return false; }
}
```

Domain specs override it: `PhysicalDeviceSpec` returns `true` unconditionally. `ComplianceControlSpec` delegates to its existing `requiresHumanReview()` field. The composition happens in `DesiredNode` itself — the record accessor is overridden to OR the field with the spec method:

```java
public record DesiredNode(... boolean requiresHuman) {
    @Override
    public boolean requiresHuman() {
        return requiresHuman || spec.requiresHuman();
    }
}
```

This makes the record field an author-level override channel and the spec method the type/instance baseline. Monotonic — once true from either source, it stays true. GoalCompilers just pass `false` unless there's an explicit author override. All four compilers now use the same one-liner instead of their domain-specific logic.

The IoT provisioner had two bugs hiding behind the original design. The provision path had a dead branch — `Failed("physical devices cannot be auto-provisioned")` — that could never execute because the executor intercepts `requiresHuman` nodes. The deprovision path had a live bug: `Failed("physical devices cannot be auto-deprovisioned")` triggers fault feedback on every reconciliation cycle, creating an infinite loop. The executor only gates provision on `requiresHuman`, not deprovision — `HumanNodeHandler` has no `onDeprovision()` method. The fix was to return `Success()` for physical device deprovisioning. Deprovisioning means "stop managing," not "physically remove."

The asymmetry between provision and deprovision gating is a runtime design gap. I filed desiredstate#54 for the proper fix — adding `requiresHuman` to `executeDeprovision` and `onDeprovision()` to `HumanNodeHandler`. The provisioner-level fix is correct regardless.

Also added a reverse index to `DeploymentProviderConfigStore` — a second `ConcurrentHashMap` mapping `providerName → Set<agentId>`, maintained alongside the primary map. The `declaredAgentIds` SPI method goes from O(n) to O(1). The method has zero callers today, but the data structure should honestly represent both lookup directions.
