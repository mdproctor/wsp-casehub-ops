---
layout: post
title: "Pools That Heal Themselves"
date: 2026-10-24
entry_type: note
subtype: diary
projects: [casehubio/casehub-ops]
tags: [desired-state, topology, pool, reconciliation, claudony]
---

# Pools That Heal Themselves

The topology test ladder has been climbing from "deploy a single LLM" through fleet scaling, lifecycle, config drift, topology mutations, observability, and chat UI. Level 10 is the first level where the desired-state runtime does something that would be genuinely tedious to hand-code: managing a pool of interchangeable instances that converges to a target size and self-repairs when one disappears.

The topology is five nodes. Three infrastructure services — manifest, eidos, engine — as `claudony_app` nodes, one parent application (`support-app`), and one `claudony_pool` node that attaches a managed pool to that parent. The pool spec carries `minActive: 3` and `maxActive: 8`. The runtime handles convergence: provision the pool on the parent app, detect when the actual state drifts from the desired state, and re-provision until they match.

The interesting part is what happens when you change your mind. The scale-up test creates a new `ClaudonyPoolNodeSpec` with different bounds, swaps it into the desired-state graph, and asks the reconciliation loop to converge. The actual state adapter detects that the pool's `minActive`/`maxActive` don't match what's desired, marks the node as `DRIFTED`, and the provisioner updates the parent app. Scale down works identically — the reconciliation loop doesn't know or care whether the bounds went up or down. It sees drift, it fixes it.

The unhealthy-replacement test is more visceral. After the pool converges, the test reaches into the fleet client and strips the pool entry from the parent app — simulating an instance that vanished. The actual state adapter reads the parent app, finds no pool with a matching `agentId`, and reports `ABSENT`. The reconciliation loop provisions it back. No special recovery logic, no health-check polling loop, no retry backoff. The same converge-to-desired mechanism that handles initial deployment handles failure recovery. That's the entire point of desired-state management — the recovery path is the deployment path.

Claude caught a real bug during review. The integration tests used the spec's `nodeId()` method — which returns `claudony_pool:support-app:support-agent` — to look up nodes in the compiled graph. But the YAML compiler assigns graph node IDs from the YAML key, which is `support-pool`. Every call to `findNodeContaining("claudony_pool:...")` would have thrown at runtime. The same problem infected `replacePoolSpec`, which was creating replacement nodes with the wrong ID. The compilation tests happened to pass because they filtered by spec type rather than by node ID. A clean test suite hiding a broken test.

The second issue was a quick fix that had been waiting since the 0.3 release. `DetectionNodeSpec` has a `correlationWindow` field of type `java.time.Duration`. The `jsonRoundTrip()` test serialised it with a plain `ObjectMapper` and called `findAndRegisterModules()` — which auto-discovers the Java Time module. Except `jackson-datatype-jsr310` wasn't on the api module's classpath. The module existed in other modules, the test compiled fine because `ObjectMapper.findAndRegisterModules()` is a runtime discovery call, and everything silently produced a `@Disabled` annotation. Adding the dependency as test-scope and registering `JavaTimeModule` explicitly restored the test. All 248 api tests pass now, including the previously disabled one.

Level 14 — fault recovery — is the natural next step. It builds directly on the pool infrastructure, exercising `FaultPolicy` SPI and automatic health-triggered replacement. The pool deployment tests already use `FailableClaudonyProvisioner` and `ClaudonyFaultPolicy`, so the machinery is warmed up.
