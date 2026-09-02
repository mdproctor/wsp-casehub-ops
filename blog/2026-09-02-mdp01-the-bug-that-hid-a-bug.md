---
layout: post
title: "The Bug That Hid a Bug"
date: 2026-09-02
entry_type: note
subtype: diary
projects: [casehubio/casehub-ops]
tags: [jackson, yaml, deserialization, detection, deployment]
---

# The Bug That Hid a Bug

Issue #71 looked like a one-liner. `DeploymentGoalLoader` creates an `ObjectMapper` with `YAMLFactory` but never registers `JavaTimeModule`. Any `DetectionNodeSpec` with `Duration` fields — `correlationWindow`, `eventBufferDelay` — fails to deserialize from YAML. The fix is literally adding `.registerModule(new JavaTimeModule())` to the field initializer.

The interesting part came when the test went green but a second test didn't.

I wrote a test that loaded a detection YAML through `load()` — the single-file path. It passed. Then I wrote a test that loaded the same YAML through `loadDirectory()` — the directory path that merges multiple fragments. That one returned an empty detections list. No error, no warning. Just silence.

The `merge()` method had been written when `DeploymentGoals` had six fields. When `detections` was added as a seventh, `merge()` wasn't updated — it hardcoded `List.of()` for detections and passed the other six through. Every detection entry loaded via directory mode was silently dropped.

```java
// Before: detections silently dropped
return new DeploymentGoals(agents, channels, caseTypes,
    trust, endpoints, List.of(), adaptations);

// After: detections preserved
return new DeploymentGoals(agents, channels, caseTypes,
    trust, endpoints, detections, adaptations);
```

This is the pattern that sealed classes and exhaustive switches are designed to prevent. A record gains a field; every constructor call site should fail to compile. But `DeploymentGoals` uses a canonical constructor with seven positional parameters, and `List.of()` is a valid `List<GoalEntry<DetectionNodeSpec>>`. The type system can't help you here — the arity matched, the types matched, and the result was wrong.

If I'd only tested through `load()`, the merge bug would have stayed hidden until someone tried loading detection topologies from a directory of YAML fragments — which is exactly how the canonical deployment topology work (#74) will structure its exemplars.

The session also reshaped what comes next. The workspace `.plan` had drifted — state said "drained" with 11 unchecked issues. Epic #74 had been prematurely closed while all eight child issues (#75–#82) remained open, and its scope checklist had items ticked that hadn't been delivered. I reopened the epic, unchecked the scope, added #72 (TransitionPlanner orphan deprovisioning) and #73 (reflective `parseSpec()` consolidation) to the scope, and built a fresh dependency-ordered queue of ten issues.

The critical path starts with #75 — `NodeSpecFactory` SPI — which must land in casehub-desiredstate before any of the ops-side topology work can begin. That's the gate. Everything else chains from it.
