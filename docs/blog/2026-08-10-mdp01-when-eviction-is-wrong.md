---
layout: post
title: "When Eviction Is the Wrong Answer"
date: 2026-08-10
entry_type: note
subtype: diary
projects: [casehub-ops]
tags: [service-lifecycle, lazy-reconstruction, eviction, memory-management, design-review]
---

I started this session planning to build eviction — the full machinery. Per-dimension TTL timers, a background sweep, lazy checks on access, two-way state transitions between loaded and evicted. The issue asked for it explicitly: partial branch loading, eviction, memory management.

The adversarial review killed it. Not the approach — the premise. A `ServiceCaseContext` is about 1KB with all nine operational dimensions. At 10,000 managed services, that's 10MB. There's no memory pressure to optimise for. The review reframed the entire scope: this isn't an eviction problem. It's a restart reconstruction problem.

The narrowing was useful. Instead of two-way state transitions (loaded ↔ evicted), we built one-way: not-loaded → loaded. Instead of per-dimension TTL timers and sweep scheduling, we built `load()` on first access after restart. The `ServiceCaseRegistry` reconstructs contexts from engine context on demand — a detection event arrives for a service the registry doesn't know about, it queries the engine for metadata and creates an unloaded context. Each dimension loads independently when it's first needed. A health event doesn't reconstruct the decommission dimension.

The real surprise came when I read `DimensionSection` during implementation. I'd been designing around the assumption that section data was cached in memory — that "loading" a dimension meant warming a local cache from the engine's durable store. It isn't. `DimensionSection.get()` delegates directly to a `ContextReader` lambda on every call. There is no cache. The section is a live view over the engine context.

That insight collapsed the remaining complexity. "Loading" a dimension means exactly one thing: populating `activeResponses` from a single structured key in the engine context. The condition status, the coordination data, the detection history — all of it reads through the live section on demand. The only state that needs explicit reconstruction is the list of active child cases, because `DimensionStatusService.recompute()` needs it to distinguish "broken and nobody knows" from "broken and being fixed."

We also filled three persistence gaps from the original implementation that reconstruction depends on. `register()` now writes service metadata to engine context at registration time — serviceId, category, deployedAt. `DimensionStatusService.recompute()` now persists the composite status after every computation. And `ServiceDetectionBridge` writes the active response list as a single structured key on every add/remove. These aren't new abstractions — they're the persistence that the design described but the implementation hadn't wired up.

Thread safety came along for the ride. `OperationalDimension` gained a per-dimension `ReadWriteLock` and `CopyOnWriteArrayList` for `activeResponses` — fixing an existing thread-safety gap where the original `ArrayList` wasn't safe for concurrent access. The locking model is clean: status reads are lock-free (volatile), section reads are lock-free (live view), only `activeResponses` access coordinates with reconstruction via the read/write lock.

The design also dropped `ServiceCaseHandle` — a separate lightweight type proposed during brainstorming for "always in memory" status summaries. With eviction deferred, the handle has no reason to exist. `ServiceCaseContext` with unloaded dimensions IS the lightweight state. If eviction is ever needed, the handle can be introduced then. The reconstruction infrastructure we built is the same infrastructure eviction would use — load and evict are symmetric.
