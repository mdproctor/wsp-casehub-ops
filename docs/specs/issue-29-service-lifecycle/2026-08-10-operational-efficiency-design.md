# Operational Efficiency — Lazy Reconstruction Design Spec

**Issue:** casehubio/casehub-ops#31 (parent: #29)
**Date:** 2026-08-10
**Status:** Draft
**Depends on:** #30 (service lifecycle domain model)

---

## 1. Problem Statement

Service cases sit in WAITING for months. After a JVM restart, all in-memory state in `ServiceCaseRegistry` is lost. Detection events for managed services arrive at `ServiceDetectionBridge`, which calls `registry.get(caseId)`, gets `null`, and silently drops them. The service's operational dimensions show no status. The dashboard is blank.

The #30 spec designed `ServiceCaseContext` and `OperationalDimension` as runtime projections of the engine's durable context, and spec'd a `getOrReconstruct()` method on the registry — but the implementation stores everything in memory with no persistence foundation and no reconstruction path.

## 2. Core Concept

**Lazy reconstruction, not proactive eviction.** After restart, service cases reconstruct from the engine's durable context on first access. Each dimension loads independently — a health event loads only the HEALTH_MONITORING dimension.

**Why not eviction?** `ServiceCaseContext` is ~1KB with unloaded dimensions. At 10,000 managed services: 10MB. No evidence of memory pressure justifies the complexity of two-way state transitions (loaded ↔ evicted), per-dimension TTL timers, and eviction scheduling. If scale demands it later, the lazy reconstruction infrastructure built here is the same infrastructure eviction would use — load and evict are symmetric.

**No ServiceCaseHandle.** The design presentation proposed a separate `ServiceCaseHandle` type as the always-in-memory sentinel. This is unnecessary abstraction — `ServiceCaseContext` with unloaded dimensions already provides identity, per-dimension status (via `OperationalDimension.status()`), and dashboard aggregation (via `toServiceHealth()`). A separate handle adds a second object to synchronize per service with no benefit when eviction is deferred. If eviction is needed later, the handle can be introduced then.

## 3. Design Decisions

| # | Decision | Choice | Key rationale |
|---|----------|--------|---------------|
| D1 | Architecture | Lazy reconstruction — one-way (not-loaded → loaded) | Eviction deferred. Simpler than two-way proxy. |
| D2 | Reconstruction granularity | Hybrid — case-level unit, dimension-level loading | A health event shouldn't load the decommission dimension. |
| D3 | Eviction trigger | Deferred | ~1KB/service × 10K = 10MB. No evidence of pressure. |
| D4 | Lightweight state | ServiceCaseContext with unloaded dimensions | Separate handle is unnecessary abstraction without eviction. |
| D5 | Event routing (non-loaded) | Load-on-demand, stays loaded | Consistent with reconstruction path. |
| D6 | Engine scope | Ops-only | Engine signal()/query() sufficient. No cross-repo changes. |
| D7 | Eviction scheduling | Deferred | No eviction, no scheduling. |
| D8 | Read behavior (not loaded) | status() always available; activeResponses requires load() | DimensionSection is a live view — section reads are always engine queries. Only activeResponses needs explicit loading. |
| D9 | TTL configuration | Deferred | No eviction. |
| D10 | Thread safety | Per-dimension ReadWriteLock + CopyOnWriteArrayList | Independent dimensions, concurrent reads, exclusive reconstruction. |
| D11 | Service discovery | Event-driven; dashboard scan deferred to #38 | Most entry points carry caseId. |

Full decision rationale and alternatives: `decisions.md` (same directory).

## 4. Persistence Foundation

The #30 implementation stores all state in memory. Reconstruction requires durable state in the engine context. Three gaps must be filled — these are not new abstractions, they are the persistence that #30 spec'd (§4.5) but didn't implement.

### 4.1 Service Metadata Persistence

`ServiceCaseRegistry.register()` must write service identity to the engine context at registration time:

| Engine context key | Value | Written by |
|---|---|---|
| `service.serviceId` | String | register() |
| `service.serviceName` | String | register() |
| `service.category` | ManagedServiceCategory name | register() |
| `service.deployedAt` | Instant ISO string | register() |

The `ContextWriter` passed to `register()` already points at the engine context. The register method writes these keys via the writer before creating the `ServiceCaseContext`.

### 4.2 Composite Status Persistence

`DimensionStatusService.recompute()` must write the computed composite status to the engine context after every recomputation:

```
dimension.section().put("status", ((Enum<?>) newStatus).name());
```

This writes to `<prefix>status` (e.g., `health.status`) via the live `DimensionSection`. After restart, an unloaded dimension's status is reconstructed by reading this key — showing the correct last-known composite status, not the default healthy status.

### 4.3 activeResponseIds Persistence

`ServiceDetectionBridge` (or whatever code adds/removes `CaseRef` entries) must write the current active response list to the engine context after every modification:

```
dimension.section().put("activeResponseIds", serializeActiveResponses(dimension.activeResponses()));
```

Stored as a JSON-serializable list of `{caseId, bindingName, createdAt}` objects in a single context key. This avoids N+1 queries during reconstruction — one `section.get("activeResponseIds")` returns the full list.

CopyOnWrite semantics: on every add/remove, the entire list is re-written. The list is bounded by the maximum concurrent child cases per dimension (typically 1-3, never unbounded).

## 5. Type Changes

### 5.1 OperationalDimension (modified, ops-api)

Gains per-dimension loading state and thread safety.

**New fields:**
- `volatile boolean loaded` — `false` after reconstruction, `true` after `load()` or on fresh creation
- `ReentrantReadWriteLock lock` — concurrent reads (status checks, dashboard), exclusive writes (reconstruction, status update, response add/remove)

**Changed fields:**
- `activeResponses`: `ArrayList<CaseRef>` → `CopyOnWriteArrayList<CaseRef>` — fixes existing thread-safety gap. Low write frequency (child case events), frequent reads (recompute iterates the list).

**New methods:**
- `load(DimensionSection.ContextReader reader)` — reconstructs `activeResponses` from engine context. Acquires write lock, double-checks `loaded` flag (coalesces concurrent requests), reads `<prefix>activeResponseIds`, deserializes, populates list, sets `loaded = true`.
- `isLoaded()` — returns `loaded` flag (volatile read, no lock needed).

**Modified methods:**
- `addResponse(CaseRef)` — acquires write lock
- `removeResponse(UUID)` — acquires write lock
- `updateStatus(DimensionStatus)` — acquires write lock
- `activeResponses()` — acquires read lock, returns unmodifiable view. `CopyOnWriteArrayList` makes iteration snapshot-consistent, but the read lock coordinates with reconstruction.
- Constructor for fresh creation sets `loaded = true` (all state is known).

**Locking rules:**

| Operation | Lock | Rationale |
|---|---|---|
| `status()`, `severity()`, `isLoaded()`, `type()`, `subscriptions()` | None | Volatile reads or immutable fields |
| `section()` | None | Live view — always available |
| `activeResponses()` | Read lock | Coordinates with reconstruction |
| `load(reader)` | Write lock | Exclusive reconstruction |
| `updateStatus(status)` | Write lock | Exclusive mutation |
| `addResponse(ref)` | Write lock | Exclusive mutation |
| `removeResponse(caseId)` | Write lock | Exclusive mutation |

### 5.2 ServiceCaseContext (modified, ops-api)

**New factory method:**

```java
public static ServiceCaseContext createForReconstruction(
        String serviceId, String serviceName,
        ManagedServiceCategory category, Instant deployedAt,
        Map<String, Object> metadata,
        DimensionSection.ContextWriter writer,
        DimensionSection.ContextReader reader)
```

Creates a context with 9 `OperationalDimension` objects, each in the **not-loaded** state:
- `status` set from `reader.read("dimensions.<type>.status")` — the persisted composite status. Falls back to `defaultStatus(type)` if not found (first-ever reconstruction before any recompute has persisted status).
- `activeResponses` empty — populated by `load()` on first access
- `section` wired to the provided writer/reader (live view to engine context)
- `subscriptions` empty — GanglionBinding restoration is a #47 concern
- `loaded = false`

The existing `create()` factory is unchanged — used at deploy time, sets `loaded = true` on all dimensions.

**`toServiceHealth()` unchanged** — reads `status()` from each dimension, which is always available (populated eagerly during reconstruction from persisted status).

### 5.3 ServiceCaseRegistry (modified, ops-app)

**New fields:**
- `ConcurrentHashMap<String, UUID> serviceIndex` — serviceId → caseId mapping for service-oriented lookups

**New methods:**

```java
public ServiceCaseContext getOrReconstruct(UUID caseId, CaseHubRuntime runtime)
```

1. Check `contexts.get(caseId)` — if found, return it
2. Query engine context for service metadata:
   - `runtime.query(caseId, "service.serviceId")`
   - `runtime.query(caseId, "service.serviceName")`
   - `runtime.query(caseId, "service.category")`
   - `runtime.query(caseId, "service.deployedAt")`
3. If `serviceId` is null → case doesn't exist or isn't a service lifecycle case → return null
4. Create `ContextWriter`/`ContextReader` lambdas wrapping `runtime.signal(caseId, ...)` / `runtime.query(caseId, ...)`
5. Call `ServiceCaseContext.createForReconstruction(...)` — creates context with persisted statuses, unloaded dimensions
6. `contexts.putIfAbsent(caseId, ctx)` — races between concurrent reconstructions resolved by ConcurrentHashMap atomicity
7. `serviceIndex.put(serviceId, caseId)` — populate the lookup index
8. Return the context

```java
public ServiceCaseContext getByServiceId(String serviceId)
```

Lookup via `serviceIndex` → `contexts`.

**Modified `register()`:**
- Writes service metadata to engine context (§4.1) via the writer before creating the context
- Populates `serviceIndex`
- Sets `loaded = true` on all dimensions (fresh deployment, all state known)

**Modified `deregister()`:**
- Removes from both `contexts` and `serviceIndex`

### 5.4 ServiceDetectionBridge (modified, ops-app)

**Modified `onDetection()`:**

```java
public void onDetection(String situationType, UUID caseId, Map<String, Object> detectionData) {
    var ctx = registry.getOrReconstruct(caseId, runtime);  // was: registry.get(caseId)
    if (ctx == null) return;

    var bindings = bindingsMap.getOrDefault(caseId, List.of());
    for (var binding : bindings) {
        if (binding.situationType().equals(situationType)) {
            var dimension = ctx.dimensions().get(binding.dimension());

            if (!dimension.isLoaded()) {
                dimension.load(/* reader from section */);
            }

            dimension.section().put(binding.contextKey(), detectionData);
            if (binding.conditionStatus() != null) {
                dimension.section().put("condition", ((Enum<?>) binding.conditionStatus()).name());
            }
            statusService.recompute(dimension);
        }
    }
}
```

**New dependency:** `CaseHubRuntime runtime` injected into the bridge (for `getOrReconstruct`).

**GanglionBinding restoration:** After restart, `bindingsMap` is empty. Events for services with no registered bindings are silently ignored (empty loop). GanglionBinding restoration is a #47 concern (RAS integration). For #31, the reconstruction path ensures the `ServiceCaseContext` exists and dimensions can be loaded — binding registration is a separate lifecycle step.

### 5.5 DimensionStatusService (modified, ops-app)

**Modified `recompute()`:**
- After computing the new status, writes it to the engine context: `dimension.section().put("status", ((Enum<?>) newStatus).name())` (§4.2)
- Precondition: dimension must be loaded. If `!dimension.isLoaded()`, the `activeResponses().isEmpty()` check produces incorrect results. The bridge ensures loading before calling recompute.

## 6. Engine Context Data Model

All keys are stored in the engine's `CaseContext` for each service lifecycle case. The `DimensionSection` auto-prepends the dimension's `contextPrefix()` (e.g., `health.`, `security.`).

### 6.1 Service-Level Keys (written at deploy time)

| Key | Type | Example value |
|---|---|---|
| `service.serviceId` | String | `"order-api"` |
| `service.serviceName` | String | `"Order API"` |
| `service.category` | String (enum name) | `"APPLICATION"` |
| `service.deployedAt` | String (ISO instant) | `"2026-08-01T10:00:00Z"` |

### 6.2 Per-Dimension Keys (written by bridge and status service)

Using HEALTH_MONITORING as example (prefix `health.`):

| Key | Type | Written by | Purpose |
|---|---|---|---|
| `health.condition` | String (enum name) | ServiceDetectionBridge | Raw condition from detection |
| `health.status` | String (enum name) | DimensionStatusService | Computed composite status |
| `health.activeResponseIds` | Serialized list | ServiceDetectionBridge | Active child case references |
| `health.lastCheckTime` | String (ISO instant) | Detection handler | Coordination data |
| `health.consecutiveFailures` | Integer | Detection handler | Coordination data |
| `health.lastIncidentId` | String (UUID) | Detection handler | Coordination data |

**Total keys per service:** ~6 service-level + ~7 per dimension × 9 dimensions ≈ 69 keys. All bounded.

### 6.3 Reconstruction Key Usage

| Reconstruction step | Keys read | Count |
|---|---|---|
| Service metadata | `service.*` | 4 |
| Per-dimension status (eager) | `<prefix>status` × 9 | 9 |
| Per-dimension load (lazy) | `<prefix>activeResponseIds` | 1 per dimension |
| **Total for unloaded context** | | **13 queries** |
| **Total for single dimension load** | | **1 additional query** |

## 7. Reconstruction Flow

### 7.1 Event-Driven Discovery (primary path)

```
Detection event arrives (CloudEvent / CDI async)
  → ServiceDetectionBridge.onDetection(situationType, caseId, data)
    → registry.getOrReconstruct(caseId, runtime)
      → contexts.get(caseId) == null (first access after restart)
      → query engine: service.serviceId, serviceName, category, deployedAt
      → create ContextWriter/ContextReader wrapping runtime.signal()/query()
      → ServiceCaseContext.createForReconstruction(...)
        → 9 dimensions created, each reads <prefix>status for initial status
        → all dimensions loaded=false
      → contexts.putIfAbsent(caseId, ctx)
      → serviceIndex.put(serviceId, caseId)
      → return ctx
    → dimension = ctx.dimensions().get(binding.dimension())
    → dimension.isLoaded() == false
    → dimension.load(reader)
      → write lock acquired
      → read <prefix>activeResponseIds → deserialize → populate activeResponses
      → loaded = true
      → write lock released
    → section.put(contextKey, detectionData)  [live write to engine context]
    → section.put("condition", conditionStatus)  [live write]
    → statusService.recompute(dimension)
      → reads condition from section [live read]
      → checks activeResponses [now populated]
      → computes composite status
      → dimension.updateStatus(newStatus) [under write lock]
      → section.put("status", statusName) [persists for future reconstruction]
```

### 7.2 Dashboard Query (deferred to #38)

First "list all services" query after restart requires discovering all service lifecycle cases in the engine. The engine's `CaseHubRuntime` does not expose a `listCasesByDefinition()` API. Options:

1. Add `listCasesByDefinition(namespace, name, version)` to `CaseHubRuntime` — clean but requires engine change (contradicts D6)
2. Query persistence layer directly — couples to persistence implementation

For #31, dashboard discovery is deferred. Services appear in the registry as events arrive. A full discovery mechanism is a #38 concern.

### 7.3 Concurrent Load Coalescing

Two threads receiving detection events for the same dimension simultaneously:

```
Thread A: acquires write lock → loaded==false → reconstructs → loaded=true → releases
Thread B: acquires write lock → loaded==true → returns immediately → releases → proceeds
```

The write lock double-check on `loaded` prevents duplicate reconstruction. No wasted work.

## 8. Thread Safety Model

### 8.1 Per-Dimension ReadWriteLock

Each `OperationalDimension` has its own `ReentrantReadWriteLock`. Dimensions on the same service don't contend — a health detection and a security detection process concurrently without blocking each other.

Within a dimension: reads (dashboard status checks, active response iteration during recompute) are concurrent. Writes (reconstruction, status update, response add/remove) are exclusive.

### 8.2 activeResponses

`CopyOnWriteArrayList<CaseRef>` replaces `ArrayList<CaseRef>`. Iteration during `recompute()` is snapshot-consistent — no `ConcurrentModificationException` even without the read lock. The write lock coordinates add/remove with reconstruction (preventing add during load from losing the added entry).

### 8.3 Status Visibility

`volatile DimensionStatus status` — unchanged from #30. Cross-thread visibility guaranteed by the volatile write in `updateStatus()`. Dashboard reads see the latest status without acquiring any lock.

### 8.4 Registry Concurrency

`ConcurrentHashMap<UUID, ServiceCaseContext>` — unchanged from #30. `putIfAbsent` in `getOrReconstruct()` resolves races between concurrent reconstruction attempts for the same case. The loser's context object is discarded (no side effects — construction is pure).

### 8.5 Lock Ordering

No deadlock risk: each dimension has one lock, and no code path acquires locks on two different dimensions simultaneously. The ConcurrentHashMap operations in the registry are lock-free.

## 9. Module Placement

| Change | Module | Package |
|---|---|---|
| OperationalDimension (loading state, thread safety) | ops-api | `io.casehub.ops.api.lifecycle` |
| ServiceCaseContext (reconstruction factory) | ops-api | `io.casehub.ops.api.lifecycle` |
| ServiceCaseRegistry (getOrReconstruct, index) | app | `io.casehub.ops.app.lifecycle` |
| DimensionStatusService (status persistence) | app | `io.casehub.ops.app.lifecycle` |
| ServiceDetectionBridge (load-on-demand, runtime injection) | app | `io.casehub.ops.app.lifecycle` |

No new types. No new modules. No new packages.

## 10. Scope Boundaries

**In scope for #31:**
- Persistence foundation (§4) — service metadata, composite status, activeResponseIds written to engine context
- OperationalDimension loading state and thread safety (§5.1)
- ServiceCaseContext reconstruction factory (§5.2)
- ServiceCaseRegistry.getOrReconstruct() with serviceId index (§5.3)
- ServiceDetectionBridge load-on-demand integration (§5.4)
- DimensionStatusService status persistence (§5.5)

**Deferred:**
- Proactive eviction — no evidence of memory pressure (D3). If needed: status-driven approach (hold dimensions at severity > OK, evict the rest). The loading infrastructure built here is the eviction infrastructure.
- TTL configuration — moot without eviction (D9). When needed: `DimensionEvictionConfig` in ops-app with `@ConfigProperty` overrides.
- Dashboard discovery — `listCasesByDefinition()` engine API or persistence query (D11, #38)
- GanglionBinding restoration after restart — #47 (RAS integration)
- DimensionSection partial hydration — #31 doesn't change DimensionSection. It remains a live view. If section reads become a performance concern (database-backed persistence), section-level caching is a future optimization.

## 11. Open Questions

1. **activeResponseIds serialization format:** JSON list of `{caseId, bindingName, createdAt}` objects? Or a simpler format (comma-separated UUIDs with metadata in separate keys)? Single-key structured data is preferred (D6 revision) to avoid N+1 queries. Implementation will choose the format — the spec constrains only that it's a single key containing the full list.

2. **DimensionStatusService.resolveStatus() for reconstruction:** The existing `resolveStatus(type, name)` method (private in `DimensionStatusService`) is needed by `ServiceCaseContext.createForReconstruction()` to parse persisted status strings. Options: make it static/public on `DimensionStatusService`, or duplicate the switch in `ServiceCaseContext`. The former is preferred — it's the canonical status resolution and shouldn't be duplicated.
