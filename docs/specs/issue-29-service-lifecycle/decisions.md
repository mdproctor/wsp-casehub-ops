# Decisions — #31 Operational Efficiency

## D1: Partial loading architecture

**Choice:** Lazy reconstruction wrapper — lightweight wrapper around ServiceCaseContext that defers dimension reconstruction until first access after restart
**Alternatives:**
- Registry-level cache — per-dimension cache entries, breaks ServiceCaseContext coherence
- Actor per dimension — clean concurrency but heavyweight for 9 dimensions per service
- Full eviction proxy — two-state proxy with proactive eviction (deferred — no evidence of memory pressure)
**Rationale:** After restart, ServiceCaseContext objects need reconstruction from engine context. A lazy wrapper defers this cost until each service is first accessed (event, dashboard query), avoiding eager reconstruction of all services on startup. The wrapper is simpler than a full eviction proxy — it has one state transition (not-loaded → loaded) rather than two-way (loaded ↔ evicted).
**Trade-offs:** First access after restart incurs reconstruction latency. Dashboard "list all services" still needs discovery (see D11).
**Exploration:** quick
**Status:** revised (R1-02: narrowed from eviction proxy to lazy reconstruction; proactive eviction deferred until evidence of memory pressure)

## D2: Reconstruction granularity

**Choice:** Hybrid — case-level reconstruction unit, dimension-level lazy loading
**Alternatives:**
- Dimension-level — each of 9 OperationalDimensions evicts/loads independently (deferred — over-engineers for current needs)
- Case-level only — entire ServiceCaseContext reconstructed on any access (simpler but loads unused dimensions)
**Rationale:** A health event shouldn't trigger reconstruction of the decommission dimension. Hybrid gives dimension-level lazy loading on reconstruction without the complexity of per-dimension eviction (9 timers per service, per-dimension eviction state). Case-level is the reconstruction unit (one wrapper per service); dimension-level is the loading granularity (each dimension loads on first access within the wrapper).
**Trade-offs:** Slightly more complex than case-level-only (per-dimension loaded flag). But avoids the 9× timer/state overhead of full dimension-level eviction.
**Exploration:** quick
**Status:** revised (R1-05: accepted hybrid alternative over pure dimension-level; R1-02: reframed from eviction to reconstruction)

## D3: Eviction trigger

**Choice:** Deferred — no proactive eviction until evidence of memory pressure
**Alternatives:**
- Time-based TTL with per-dimension hints (original choice — deferred)
- Status-driven — dimensions at severity > OK stay resident (preferred future approach if eviction is needed)
- Purely time-based — single configurable TTL for all dimensions
**Rationale:** ServiceCaseContext objects are ~3KB each; at 10,000 services that's 30MB. No evidence justifies proactive eviction at this scale. When/if eviction is needed, status-driven approach (base TTL + hold-open for any dimension at severity > OK) is preferred over static per-dimension hints — runtime status IS the correct eviction signal.
**Trade-offs:** If memory pressure materializes at extreme scale, eviction will need to be designed then. But the cost of designing it later is low because the lazy reconstruction infrastructure (D1, D2) provides the foundation.
**Exploration:** quick
**Status:** revised (R1-02, R1-06: deferred; noted status-driven as preferred future approach)

## D4: Minimal sentinel (handle)

**Choice:** Lightweight ServiceCaseHandle — caseId, serviceId, category, per-dimension status summary (DimensionType → DimensionStatus), lastActivity timestamp (~200 bytes per service)
**Alternatives:**
- Full eviction — even the handle evicts, case only in engine persistent context
- Status + resident health dimension — handle plus HEALTH_MONITORING always loaded
**Rationale:** The handle provides enough data for event routing (which dimension to target) and dashboard rendering (status summary) without reconstructing any dimension. ~200 bytes per service scales to thousands.
**Synchronization:** Handle status updates on every `DimensionStatusService.recompute()` call. `DimensionStatusService` is the single owner — it writes dimension status AND updates the handle summary in the same operation. Handle status map uses ConcurrentHashMap for thread-safe reads during concurrent dashboard access. Between a status change and the next recompute, the handle may show a briefly stale value — acceptable because recompute is triggered by every detection and child case lifecycle event.
**Trade-offs:** Handle status is a point-in-time snapshot — may lag behind the actual dimension status by at most the time between recompute calls.
**Exploration:** quick
**Status:** revised (R1-07: added synchronization mechanism — DimensionStatusService as single owner, ConcurrentHashMap for thread-safe reads)

## D5: Event routing for non-loaded dimensions

**Choice:** Load-on-demand — reconstruct dimension from engine context, process event, dimension stays loaded until next restart
**Alternatives:**
- Queue-then-batch — buffer events, periodic sweep loads and drains (adds latency)
- Passthrough to engine — skip ops-layer dimension, write directly to CaseHubRuntime.signal() (loses DimensionStatusService recomputation)
**Rationale:** Consistent with restart reconstruction path. The dimension loads, processes the event with full DimensionStatusService recomputation, and stays loaded. No re-eviction — with proactive eviction deferred (D3), dimensions stay loaded once accessed.
**Trade-offs:** Bursty events on multiple dimensions of the same service trigger multiple sequential reconstructions. Bounded: at most 9 reconstructions per service.
**Exploration:** quick
**Status:** revised (R1-02: reframed from "evicted dimensions" to "non-loaded dimensions after restart"; removed TTL re-eviction concern)

## D6: Engine scope

**Choice:** Ops-only — all reconstruction logic in ops-api and ops-app. No cross-repo engine changes.
**Alternatives:**
- Engine-assisted — add queryPrefix(caseId, prefix) to engine for efficient dimension section loading
- Engine-level passivation — generic sub-case partition passivation in the engine (rejected — engine CaseContext is flat, context prefixes are ops-layer convention)
**Rationale:** The engine's CaseContext is a flat key-value store with no partition concept. Context prefixes (health., security.) are an ops-layer naming convention, not an engine concept. The engine's existing signal()/query() API is sufficient for per-key reconstruction. Dimension sections store bounded coordination data (~7-8 keys per dimension). activeResponses stored as structured data (list of {caseId, bindingName, createdAt} objects in a single context key) avoids N+1 queries — single query returns the full list.
**Trade-offs:** Reconstruction requires ~65 individual query() calls per service (7-8 keys × 9 dimensions). Bounded and acceptable for a restart-only cost.
**Depends on:** D2 (hybrid means each lazy load reconstructs one dimension's keys, not all 9×N)
**Exploration:** quick
**Status:** revised (R1-08: clarified activeResponses storage as structured data to avoid N+1; R1-03: engine-level passivation rejected — engine CaseContext is flat, no partition concept)

## D7: Eviction scheduling mechanism

**Choice:** Deferred — no proactive eviction scheduling until evidence of memory pressure
**Alternatives:**
- Both lazy check + periodic sweep (original choice — deferred with eviction)
- Scheduled sweep only
- Lazy on access only
**Rationale:** With proactive eviction deferred (D3), eviction scheduling is moot. Lazy reconstruction on first access (D1, D2) handles the restart scenario without any background scheduling.
**Exploration:** quick
**Status:** revised (R1-02: deferred with proactive eviction)

## D8: Proxy read behavior when non-loaded

**Choice:** Explicit loading — status reads use the handle (fast, no I/O). Section and activeResponses access require explicit dimension.load() call that makes the I/O cost visible.
**Alternatives:**
- Transparent auto-load on section read (original choice — rejected: transparent proxies that hide I/O behind property access are a known anti-pattern)
- Return empty, caller decides — section reads on non-loaded dimensions return null (too extreme — forces eviction awareness into every caller)
**Rationale:** A caller writing dimension.section().get("condition", String.class) expects nanosecond in-memory access. If the dimension is not loaded after restart, they'd get millisecond reconstruction from engine context — a 10,000× latency difference hidden behind an identical API. Explicit load() makes the I/O cost visible. Status reads from the handle (D4) remain fast for the common dashboard/routing path. This avoids the Hibernate LazyInitializationException class of problems.
**Trade-offs:** Every call site that needs section data must call load() first. This is the point — it forces callers to be explicit about reconstruction cost. Event routing (D5) calls load() as part of the load-on-demand path.
**Depends on:** D1 (lazy reconstruction wrapper), D4 (handle for status reads), D5 (load-on-demand)
**Exploration:** quick
**Status:** revised (R1-09: changed from transparent auto-load to explicit loading — JPA lazy loading anti-pattern argument accepted)

## D9: TTL configuration

**Choice:** Deferred — no TTL configuration until proactive eviction is needed
**Alternatives:**
- DimensionType enum field (original choice — rejected: violates API-implementation separation, no other platform enum carries deployment tuning)
- DimensionEvictionConfig in ops-app with application.properties overrides (preferred future approach)
- EvictionPolicy SPI — pluggable per-dimension policy (over-engineers a simple TTL)
**Rationale:** With proactive eviction deferred (D3), TTL configuration is moot. If needed in the future, the platform convention is application.properties for deployment-specific tuning (@ConfigProperty — already used in ops-app for approval roles). No other platform enum carries deployment tuning. DimensionEvictionConfig in ops-app with defaults in code and overrides via application.properties follows the established pattern.
**Exploration:** quick
**Status:** revised (R1-02: deferred; R1-10: DimensionType enum field rejected, DimensionEvictionConfig noted as future approach)

## D10: Thread safety model

**Choice:** Per-dimension ReadWriteLock with CopyOnWriteArrayList for activeResponses
**Alternatives:**
- Per-service lock — coarser granularity, simpler but blocks unrelated dimension access during reconstruction
- Lock-free CAS — complex, no practical benefit for the bounded contention expected
- Synchronized blocks — adequate but ReadWriteLock allows concurrent reads
**Rationale:** Dimensions are independently modifiable — two threads accessing different dimensions should not contend. Within a dimension: reads (dashboard, status checks) are concurrent; writes (reconstruction, status update, response add/remove) are exclusive. activeResponses uses CopyOnWriteArrayList — low write frequency (child case creation/completion events), frequent reads (status recomputation iterates the list). volatile DimensionStatus is sufficient for status visibility (already present in OperationalDimension). Concurrent load-on-demand requests for the same dimension coalesce via the write lock — first acquirer reconstructs, second acquirer finds it loaded when it gets the lock.
**Trade-offs:** Per-dimension lock adds 9 lock objects per service. At 1,000 services, 9,000 locks — negligible memory, clean concurrency.
**Exploration:** new (surfaced by R1-12)
**Status:** captured

## D11: Service discovery after restart

**Choice:** Lazy discovery with engine case scan fallback
**Alternatives:**
- Eager startup scan — query engine for all service lifecycle cases on startup (blocks startup, loads all services immediately)
- Persistent ID mapping — external service-ID-to-case-ID table (additional persistence infrastructure)
**Rationale:** Most entry points carry the case ID or service ID (events carry correlation data, dashboard queries carry service ID). The registry discovers services lazily on first access: event arrives → resolve case ID from correlation data → getOrReconstruct(). For "list all services" queries (dashboard), a one-time scan populates the registry's discovery index — deferred until first list query, not on startup. The engine runtime tracks active cases internally (required for signal routing), but does not expose a public cross-case listing API. Discovery requires either: (a) querying the persistence layer directly (JPA repository for hibernate-backed deployments), or (b) adding a `listCasesByDefinition()` method to `CaseHubRuntime`. Option (b) is cleaner — it's a read-only query that belongs in the runtime API.
**Trade-offs:** First "list all services" query after restart triggers the engine scan (bounded by total service case count). Subsequent list queries are served from the populated index. Event-driven discovery handles the common case without any scan.
**Exploration:** new (surfaced by R1-13)
**Status:** captured
