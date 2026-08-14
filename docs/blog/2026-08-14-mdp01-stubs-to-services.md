---
layout: post
title: "When Stubs Become the Architecture"
date: 2026-08-14
entry_type: note
subtype: diary
projects: [casehub-ops]
tags: [case-descriptors, sse, rest, desiredstate, architecture]
series: issue-38-stubbed-ui-screens
---

*Continues from [2026-08-10: When Eviction is Wrong](2026-08-10-mdp01-when-eviction-is-wrong.md).*

The ops console had six REST resources returning empty lists and `Response.ok().build()`. Twenty-odd endpoints that did nothing. I'd stubbed them during the initial topology work because the services they depended on didn't exist yet. Today those services exist, and the stubs turned into something worth writing about.

## The case descriptor pattern held

CveResponseCaseDescriptor was the third four-phase case descriptor I'd built (after IncidentResponse and ComplianceRemediation). ServiceUpgradeCaseDescriptor was the fourth. By this point the pattern — assess, act, verify, escalate — isn't just repetition. It's revealing what the abstraction actually is.

Every operational concern in the ops console follows the same shape: something happens, the system evaluates whether it can respond automatically, it acts or escalates to a human, then it verifies the result propagated through the desiredstate reconciliation loop. The four-phase pattern isn't a coding convention. It's the operational lifecycle itself, expressed as a case definition.

The interesting part is the verify phase. CVE remediation calls `updateServiceImage` on ApplicationLifecycleService, which updates the entity, recompiles the desired-state graph, and feeds it to the ReconciliationLoop. The verify worker doesn't poll for completion — it registers with NodeConvergenceTracker, which observes reconciliation CloudEvents. When the affected nodes converge, the tracker signals the case context, and the completion expression resolves. The case descriptor never needs to know how Kubernetes deployments work. It speaks in node IDs and convergence.

## CveStatusObserver and the lifecycle event seam

The CveResponseCaseDescriptor handles CVE remediation through the case engine. But CveStore — the query-side persistence — needs to know when a CVE case completes so it can update the record's status from DETECTED to RESOLVED.

The seam is `CaseLifecycleEvent`, a CDI event the engine fires on every auditable case transition. A `@ObservesAsync` observer filters by namespace and case definition name, checks for terminal states, extracts the CVE ID from the context snapshot, and updates CveStore. The observer is six lines of logic wrapped in standard CDI boilerplate.

What makes this worth noting: the observer sees the case context as a JsonNode snapshot frozen at transition time. It's read-only. If you try to mutate it, nothing happens — the engine has already moved on. I initially reached for the working context before realising the snapshot is the correct tool. You want the state at the moment of completion, not whatever the context looks like by the time the observer runs on its async thread.

## The SSE broadcaster's ring buffer

ApplicationEventBroadcaster aggregates three event sources — reconciliation, case lifecycle, and application status — into a single per-application ring buffer. SSE clients subscribe with a filter (ALL, CASE, or RECONCILIATION) and receive only matching events.

The design decision that matters is gap detection. When a client reconnects with a `Last-Event-ID` that's been evicted from the ring buffer, the broadcaster sends a synthetic gap event before replaying from the oldest buffered entry. The client uses this to trigger a full state refresh. For a monitoring dashboard, losing a few events is fine as long as the client *knows* it lost them. Silent data loss is the failure mode; acknowledged gaps with a recovery path are operational.

The ring buffer itself is a synchronized circular array. Nothing clever — `add()`, `snapshot()`, `size()`. I briefly considered a lock-free ring buffer and decided against it. The buffer is per-application, the write path is CDI observers on the event bus, and contention is low. Correctness over cleverness.

## Extracting ScalingService

ScalingResource had accumulated real domain logic — cooldown checking, policy merging, event emission — behind a REST facade. ServiceOperationResource needed the same scaling logic for its `/scale` endpoint. Rather than duplicate or inject the resource directly, we extracted ScalingService with the same functional-interface constructor pattern: `Consumer<ScalingRequestedEvent>`, `BiPredicate<String, String>` for cooldown, `BiConsumer<String, String>` for timestamp recording.

The existing tests migrated cleanly because they were already testing through the functional interface, not through HTTP. ScalingResource became twelve lines — load entity, delegate to service.

## Six resources, one pattern

Wiring the REST endpoints revealed a consistency I hadn't planned for. Every resource follows the same pattern: inject a domain service, load the ApplicationEntity by ID, delegate. The resources that need cross-cutting infrastructure (SSE, reconciliation status) inject ApplicationEventBroadcaster or ReconciliationLoop. None of them contain domain logic.

SecurityResource is the most complete example. `getCves` delegates to CveStore. `scanCves` stores the CVE record *and* signals the application case via `CaseHubRuntime.signal()` — the signal triggers the cve-response child case through the engine's binding mechanism. `getPosture` aggregates CVE status counts. Three endpoints, each a single responsibility, each testable in isolation from JAX-RS.

The one surprise: ReconciliationLoop has no `status()` method. To check whether a loop is active for a given cluster, you call `getDesired(compositeKey)` — null means no loop, non-null gives you the current graph and its node count. Fifteen public methods on the class and none of them are "is this thing running."

## What this opens up

The ops console now has real data flowing through every endpoint. The backend is no longer pretending — CVE scans produce case workflows, deployments are queryable, reconciliation state is observable. The blocks-ui repo is now in the slot, ready for the five Lit components that will consume these APIs. That's Task 9 — the last piece before this epic batch closes.

