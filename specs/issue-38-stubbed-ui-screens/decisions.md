## D1: Repo and slot strategy for cross-repo work

**Choice:** Add blocks-ui to slot 86 — both repos on the same branch, single session, coherent design
**Alternatives:**
- Separate slot for blocks-ui — decouple component work from backend wiring (adds roundtrips, components designed without backend context)
- Sequential in this slot — ops backend first, then blocks-ui separately (delays end-to-end validation)
**Rationale:** Co-designing the data contracts (component prop interfaces) alongside the backend REST API that produces the data avoids roundtrips. The components themselves are generic (D2) — they don't hardcode REST paths.
**Trade-offs:** Slot 86 grows to two repos, slightly more complex branch management.
**Exploration:** quick
**Status:** revised (R1-09: clarified rationale — co-design of data contracts, not API coupling)

## D2: Component scope — generic building blocks, not ops-branded

**Choice:** Build generic building blocks in blocks-ui. Five new components needed; six existing ones reused via registration/composition.
**Reuse existing:**
- `case-explorer` — register ops case types via its entity SPI
- `approval-gate` + `work-item-inbox` — approval workflow already built
- `blocks-timeline` — pluggable strategy for ops events
- `compliance-summary` — regulation grid already built
- `audit-trail-viewer` — already exists
**Build new (generic names):**
- `blocks-service-card` — per-service health, replicas, image, per-cluster status
- `blocks-cluster-panel` — register/deregister clusters, connectivity status
- `blocks-reconciliation-status` — desired vs actual diff per cluster/node
- `blocks-dimension-dashboard` — multi-dimension status overview with severity aggregation
- `blocks-topology-viewer` — service dependency DAG using existing graph stack (see D3)
**Alternatives:**
- Ops-branded components (`ops-topology-view`, `ops-service-card`) — tightly coupled to ops REST API, not reusable by scaffold or other apps
**Rationale:** blocks-ui philosophy is generic building blocks that work via props/endpoints. Domain coupling belongs in the consuming app, not the component. Components work from mocks or any REST API.
**Trade-offs:** Generic components require more thought on the data contracts — they can't assume ops-specific field names.
**Exploration:** quick
**Status:** captured

## D3: Topology graph rendering approach

**Choice:** `blocks-topology-viewer` component using the existing `blocks-dag-viewer` read-only pattern — data props → `graph-renderer` (React Flow Lit-wrapped) with `computeElkLayout`. Explore extending `blocks-dag-viewer` with pluggable node renderers rather than creating a separate `graph-stencil-topology` package. If `blocks-dag-viewer`'s HTN stencil supports custom node types (as React Flow does), a topology-specific node renderer avoids package proliferation.
**Alternatives:**
- New `graph-stencil-topology` package — follows one-stencil-per-domain pattern but adds a package for what may be a node renderer difference only
- Full `DiagramBaseMixin` — overkill for read-only display
- ECharts graph — wrong tool for structural DAGs
**Rationale:** Read-only service topology (nodes with health status, edges for dependencies) is structurally similar to DAG plan viewing. The rendering infrastructure (React Flow, ELK layout) is shared. If the difference is only node appearance, a new stencil package is unnecessary.
**Depends on:** D2 (component scope)
**Trade-offs:** If topology semantics diverge significantly from DAG plans (e.g., bidirectional edges, grouping by cluster), a separate stencil may be needed. Start with extension, extract if the abstraction leaks.
**Exploration:** quick
**Status:** revised (R1-10: explore extending blocks-dag-viewer before creating new stencil)

## D4: Backend wiring strategy — wire everything

**Choice:** Wire all stubbed REST endpoints to real services, including building a CVE registry for SecurityResource. Also replace the two remaining `StubChildCaseDescriptor` stubs (cve-response, service-upgrade) with real case descriptors.
**Endpoints to wire:**
- Category 1 (data exists): CaseResource list/get, DeploymentResource list/current, SecurityResource posture, ApplicationResource update
- Category 2 (new logic): ReconciliationResource status/trigger/SSE, SecurityResource CVEs (new CVE registry), ServiceOperationResource status/scale/upgrade, DeploymentResource rollback, CaseResource/ReconciliationResource SSE streams
**Case descriptors to replace:**
- `ops:cve-response` — StubChildCaseDescriptor → real CveResponseCaseDescriptor
- `ops:service-upgrade` — StubChildCaseDescriptor → real ServiceUpgradeCaseDescriptor
**New ApplicationLifecycleService method required:**
- `updateServiceImage(UUID applicationId, String serviceId, String newImage, String tenancyId) → Set<String>` — same pattern as `updateServiceConfig`/`updateServiceReplicas`: update entity → recompile desired-state graph → `ReconciliationLoop.updateDesired()` → return affected node IDs. Image update flows through the desiredstate reconciliation loop, not bypassing it.
**Alternatives:**
- Wire Category 1 + most impactful Category 2, defer CVE tracking — smaller scope but leaves SecurityResource hollow
- Category 1 only — minimal, leaves reconciliation and SSE stubbed
**Rationale:** "Full implementation" means full. CVE tracking, service-upgrade, and cve-response case types complete the ops console's operational story.
**Trade-offs:** Larger scope — CVE registry, two new case descriptors, and new lifecycle service method add implementation work.
**Exploration:** quick
**Status:** revised (R1-01: added updateServiceImage method — no existing method handles image updates; R1-07/R1-15: confirmed remediation flows through desiredstate loop via lifecycle service, NodeConvergenceTracker works correctly)

## D5: CVE registry design — app-scoped persistence

**Choice:** `CveStore` interface and `JpaCveStore` implementation both in `app/`. Flyway migration for the CVE table. Queryable by applicationId and serviceId.
**Data flow:**
- REST ingest (`SecurityResource.scanCves`) → writes to `CveStore` AND signals the application case via `CaseHubRuntime.signal()`
- `CveResponseCaseDescriptor` assess worker reads from case context (signal payload), NOT from `CveStore`
- `SecurityResource.getCves` reads from `CveStore` for the list view
- `CveStore` is the query store; case context is the processing store
**Alternatives:**
- SPI in `api/` with implementation in `app/` — leaks app-specific concern into shared API; no domain module consumes CVE data
- In-memory `ConcurrentHashMap` — transient, lost on restart
- Query from case blackboard only — depends on case existing, no independent queryability
- Compliance domain `EvidenceCollector` — CVEs drive operational remediation (update image NOW), not compliance evidence collection. Different concern despite surface similarity.
**Rationale:** CveStore has no cross-domain consumers — only `SecurityResource` and the list view in `app/`. Placing it in `api/` would pollute the shared module. The `PlanStore` precedent doesn't apply because `PlanStore` is consumed by domain module provisioners.
**Depends on:** D4 (CVE registry is part of wiring everything)
**Trade-offs:** JPA entity + Flyway migration. Justified by operational data persistence across restarts.
**Exploration:** quick
**Status:** revised (R1-02: moved from api/ to app/ — no cross-domain consumers; R1-03: explicitly rejected compliance evidence alternative; R1-11: specified CveStore data flow)

## D6: SSE event streaming — single ApplicationEventBroadcaster

**Choice:** JAX-RS `SseBroadcaster` with a single `ApplicationEventBroadcaster` service aggregating all event sources. In-memory ring buffer (configurable, default 1000) per application for `Last-Event-ID` reconnection. CloudEvents JSON format. Two SSE endpoints are filtered views of the same stream.
**Event sources aggregated:**
- Reconciliation events — emitted by `ReconciliationLoop` callbacks (new wiring needed)
- Case lifecycle — child case spawned/completed, status changes (CDI `@ObservesAsync`)
- Application lifecycle — deployment started, status changed (CDI `@ObservesAsync`)
**Filtered views:**
- `/api/applications/{id}/cases/events` — all events (reconciliation + case + application)
- `/api/applications/{id}/reconciliation/events` — reconciliation events only
**Gap detection:** When a client reconnects with `Last-Event-ID` and the ID has been evicted from the ring buffer, the broadcaster sends a synthetic `gap` event before streaming from the oldest buffered event. The client uses this to trigger a full state refresh.
**Alternatives:**
- Reactive `Multi<ServerSentEvent>` via Mutiny — adds reactive complexity to blocking resources
- Separate broadcaster per event type — per-concern ring buffers prevent cross-domain eviction, but 3× broadcaster infrastructure
**Rationale:** Single broadcaster simplifies wiring. The shared ring buffer concern (R1-04) is mitigated by the gap event — clients know when they've missed events and can refresh. For a monitoring dashboard, a full state refresh after reconnection is acceptable.
**Trade-offs:** Shared ring buffer means a reconciliation event burst can evict case events. Gap event + client refresh is the recovery path.
**Exploration:** quick
**Status:** revised (R1-04: acknowledged shared buffer concern, mitigated by gap event; R1-05: added gap event specification)

## D7: CveResponseCaseDescriptor and ServiceUpgradeCaseDescriptor

**Choice:** Four-phase worker chains following the proven pattern. CVE → affected service resolution happens in the cve-response assess worker, not the REST ingest endpoint. Both descriptors inject `ApplicationLifecycleService` + `NodeConvergenceTracker`.
**CveResponseCaseDescriptor (`ops:cve-response`):**
- assess-cve: validate input, query ApplicationLifecycleService for services using affected image, determine remediation (update image if fixedInTag present, else escalate). Case-level approval gate via engine's `ActionRiskClassifier` → `HumanTaskScheduleHandler` → WorkItem when severity HIGH/CRITICAL — same mechanism as described in ARC42STORIES §8 for case-level approval (distinct from provisioner-level `ApprovalEvaluator`).
- remediate-cve: call `ApplicationLifecycleService.updateServiceImage()` — flows through desiredstate reconciliation loop (update entity → recompile → reconcile), returns affected node IDs
- verify-cve: register with NodeConvergenceTracker for affected node IDs — works because remediation flows through reconciliation, which emits convergence CloudEvents
- escalate-cve: produce summary for human review when no fix available or remediation fails
**ServiceUpgradeCaseDescriptor (`ops:service-upgrade`):**
- assess-upgrade: validate input (serviceId, newImage, applicationId), confirm service exists
- execute-upgrade: call `ApplicationLifecycleService.updateServiceImage()`
- verify-upgrade: register with NodeConvergenceTracker
- escalate-upgrade: produce summary if lifecycle service throws
**Alternatives:**
- CVE resolution in REST endpoint — couples REST to resolution logic, breaks the pattern where cases own their assessment
**Rationale:** Every other case type follows assess → act → verify → escalate with the case owning assessment logic. Consistency and testability. Remediation flows through the desiredstate reconciliation loop (not bypassing it), so NodeConvergenceTracker receives convergence events correctly.
**Depends on:** D4 (wiring everything includes replacing stub case types and adding `updateServiceImage`), D5 (CveStore for REST query views — case workers use signal payload, not CveStore)
**Trade-offs:** Two new case descriptors with four workers each is substantial code. But the pattern is proven and each worker is independently testable.
**Exploration:** quick
**Status:** revised (R1-01: explicitly references new `updateServiceImage` method from D4; R1-06: clarified approval mechanism — engine-level ActionRiskClassifier, not ApprovalEvaluator; R1-07: confirmed remediation flows through desiredstate loop, convergence tracking works; R1-11: clarified CveStore vs case context data flow)
