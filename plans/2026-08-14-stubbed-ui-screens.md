# Stubbed UI Screens — Full Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #38 — ops-console: stubbed UI screens — full implementation
**Issue group:** #38 (batch 4 of epic #29)

**Goal:** Replace all stubbed REST endpoints with real service wiring, build
five new blocks-ui components, and replace the two remaining stub case
descriptors with full implementations.

**Architecture:** Three workstreams: (A) persistence + lifecycle service
foundation, (B) case descriptors + registrar, (C) SSE infrastructure +
REST wiring + blocks-ui components. Each workstream produces independently
testable deliverables.

**Tech Stack:** Java 21, Quarkus, Panache ORM, JAX-RS SSE, Lit 3.x,
vitest, React Flow (via graph-renderer)

## Global Constraints

- IntelliJ MCP mandatory for all Java source file operations
- Pre-release platform — breaking changes cost nothing
- TDD: write failing test → verify fail → implement → verify pass → commit
- Each commit references issue: `Refs casehubio/casehub-ops#38`
- Package root: `io.casehub.ops.app`
- Source root: `app/src/main/java/io/casehub/ops/app/`
- Test root: `app/src/test/java/io/casehub/ops/app/`
- blocks-ui root: `/Users/mdproctor/claude/casehub/blocks-ui/`
- Named datasource: `quarkus.datasource.app.*` for all JPA entities
- Flyway path: `app/src/main/resources/db/app/migration/`
- Run tests: `mvn --batch-mode -o test -pl app`

---

### Task 1: CVE Persistence — CveStore + JpaCveStore + CveEntity + Flyway

**Files:**
- Create: `app/src/main/java/io/casehub/ops/app/model/CveRecord.java`
- Create: `app/src/main/java/io/casehub/ops/app/model/CveStatus.java`
- Create: `app/src/main/java/io/casehub/ops/app/persistence/CveStore.java`
- Create: `app/src/main/java/io/casehub/ops/app/persistence/JpaCveStore.java`
- Create: `app/src/main/java/io/casehub/ops/app/entity/CveEntity.java`
- Create: `app/src/main/resources/db/app/migration/V6__cve_record.sql`
- Test: `app/src/test/java/io/casehub/ops/app/persistence/JpaCveStoreTest.java`

**Interfaces:**
- Consumes: `CveSeverity` (existing in `app/src/main/java/io/casehub/ops/app/model/CveEvent.java`)
- Produces: `CveStore` interface — consumed by SecurityResource (Task 8), CveStatusObserver (Task 5)

- [ ] **Step 1: Create CveStatus enum and CveRecord**

`CveStatus`: `DETECTED`, `REMEDIATING`, `RESOLVED`, `ESCALATED`

`CveRecord`: record with fields `cveId`, `severity` (CveSeverity), `affectedImage`,
`affectedServices` (List<String>), `fixedInTag`, `status` (CveStatus), `applicationId`
(UUID), `tenancyId`, `detectedAt` (Instant).

- [ ] **Step 2: Create CveStore interface**

```java
public interface CveStore {
    void store(CveRecord record);
    List<CveRecord> findByApplicationId(UUID applicationId);
    List<CveRecord> findByServiceId(UUID applicationId, String serviceId);
    Optional<CveRecord> findByCveId(UUID applicationId, String cveId);
    void updateStatus(UUID applicationId, String cveId, CveStatus newStatus);
}
```

- [ ] **Step 3: Create Flyway migration V6__cve_record.sql**

```sql
CREATE TABLE cve_record (
    cve_id          VARCHAR(64)  NOT NULL,
    severity        VARCHAR(16)  NOT NULL,
    affected_image  VARCHAR(512) NOT NULL,
    affected_services TEXT,
    fixed_in_tag    VARCHAR(256),
    status          VARCHAR(16)  NOT NULL DEFAULT 'DETECTED',
    application_id  UUID         NOT NULL,
    tenancy_id      VARCHAR(128) NOT NULL,
    detected_at     TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (application_id, cve_id)
);
CREATE INDEX idx_cve_record_app ON cve_record(application_id);
CREATE INDEX idx_cve_record_tenancy ON cve_record(tenancy_id);
```

- [ ] **Step 4: Create CveEntity JPA entity**

Panache entity mapping to `cve_record` table. Composite key `(applicationId, cveId)`.
Follow `ApprovalPlanEntity` pattern for the named datasource. Include
`findByApplicationId`, `findByServiceId`, `findByCveId` static query methods.

- [ ] **Step 5: Write failing tests for JpaCveStore**

Test class: `JpaCveStoreTest.java`. Unit tests with mocked entity manager or
Panache mock. Test: store + findByApplicationId, findByServiceId,
findByCveId, updateStatus, store duplicate cveId (upsert or error).

- [ ] **Step 6: Implement JpaCveStore**

`@ApplicationScoped` CDI bean implementing `CveStore`. Maps between `CveRecord`
and `CveEntity`. Uses Panache queries.

- [ ] **Step 7: Run tests, verify pass**

Run: `mvn --batch-mode -o test -pl app -Dtest=JpaCveStoreTest`

- [ ] **Step 8: Commit**

```
feat(#38): add CveStore persistence — interface, JPA impl, Flyway V6
```

---

### Task 2: ApplicationLifecycleService — updateServiceImage + rollbackToDeployment

**Files:**
- Modify: `app/src/main/java/io/casehub/ops/app/service/ApplicationLifecycleService.java`
- Test: `app/src/test/java/io/casehub/ops/app/service/ApplicationLifecycleServiceTest.java`

**Interfaces:**
- Consumes: `ApplicationEntity`, `ServiceDefinition`, `ClusterService`, `ReconciliationLoop`, `ApplicationGoalCompiler`
- Produces: `updateServiceImage(UUID, String, String, String) → Set<String>` — consumed by CveResponseCaseDescriptor (Task 3), ServiceUpgradeCaseDescriptor (Task 4); `rollbackToDeployment(UUID, UUID, String)` — consumed by DeploymentResource (Task 8)

- [ ] **Step 1: Write failing tests for updateServiceImage**

In `ApplicationLifecycleServiceTest.java`, add tests following the existing
`updateServiceConfig` test pattern:
- Happy path: image updated, graph recompiled, loop updated, returns node IDs
- Unknown service: throws `IllegalArgumentException`
- Unknown application: throws `IllegalArgumentException`
- DRAFT application (no engineCaseId): updates entity only, no reconciliation

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn --batch-mode -o test -pl app -Dtest=ApplicationLifecycleServiceTest#*updateServiceImage*`

- [ ] **Step 3: Implement updateServiceImage**

Follow the exact pattern of `updateServiceConfig` (lines 324-370):
1. Load `ApplicationEntity` by ID
2. Parse services from JSON
3. Find matching service by serviceId
4. Create new `ServiceDefinition` with updated `image` field (keep all other fields)
5. Serialize back to JSON, persist
6. Recompile per cluster, update reconciliation loops
7. Return affected node IDs

The key difference from `updateServiceConfig`: instead of merging env map,
replace the `image` field.

- [ ] **Step 4: Write failing tests for rollbackToDeployment**

- Happy path: loads DeploymentRecordEntity, restores topologyJson to app, recompiles
- Unknown deployment: throws
- Unknown application: throws

- [ ] **Step 5: Implement rollbackToDeployment**

1. Load `ApplicationEntity` and `DeploymentRecordEntity` by IDs
2. Replace `app.servicesJson` with `deployment.topologyJson`
3. Persist, recompile per cluster, update reconciliation loops
4. Record new `DeploymentRecordEntity` with trigger=ROLLBACK

- [ ] **Step 6: Run all tests, verify pass**

Run: `mvn --batch-mode -o test -pl app -Dtest=ApplicationLifecycleServiceTest`

- [ ] **Step 7: Commit**

```
feat(#38): add updateServiceImage and rollbackToDeployment to lifecycle service
```

---

### Task 3: CveResponseCaseDescriptor

**Files:**
- Create: `app/src/main/java/io/casehub/ops/app/case_/CveResponseCaseDescriptor.java`
- Test: `app/src/test/java/io/casehub/ops/app/case_/CveResponseCaseDescriptorTest.java`

**Interfaces:**
- Consumes: `ApplicationLifecycleService.updateServiceImage()` (Task 2), `NodeConvergenceTracker.register()`
- Produces: `CaseDefinition` via `build(ApplicationLifecycleService, NodeConvergenceTracker)` — consumed by CaseDefinitionRegistrar (Task 6)

- [ ] **Step 1: Write failing tests**

Follow `ComplianceRemediationCaseDescriptorTest` pattern exactly. Test each worker:
- `assessCve` — fixedInTag present + services found → `.cveAssessment` with action=`update-image`
- `assessCve` — fixedInTag null → `.cveEscalationRequired`
- `assessCve` — no affected services (image not matching) → `.cveEscalationRequired`
- `assessCve` — HIGH severity → `.cveApprovalRequired = true`
- `assessCve` — missing required fields (null cveId, severity, affectedImage) → `WorkerResult.failed()`
- `remediateCve` — happy path → calls `updateServiceImage`, returns `.cveRemediationExecuted`
- `remediateCve` — lifecycle service throws → `.cveEscalationRequired`
- `verifyCve` — registers with convergenceTracker
- `escalateCve` — writes `.cveStatus = "escalated"` and summary
- Case definition identity: namespace=ops, name=cve-response, version=1.0

- [ ] **Step 2: Run tests to verify they fail**

- [ ] **Step 3: Implement CveResponseCaseDescriptor**

Follow `ComplianceRemediationCaseDescriptor` structure exactly:
- `build()` static method, private constructor
- Four capabilities: `assess-cve`, `remediate-cve`, `verify-cve`, `escalate-cve`
- Four workers with `WorkerFunction.Sync<Map, Map>`
- Three bindings: `.cveAssessment` → remediate, `.cveRemediationExecuted` → verify, `.cveEscalationRequired` → escalate
- Completion: `.cveStatus == "resolved" || .cveStatus == "escalated"`

Assess worker logic:
- Validate required fields (cveId, severity, affectedImage)
- Extract `affectedServices` from input (list of service IDs whose image matches `affectedImage`) — the REST endpoint resolves this from ApplicationEntity before signalling
- If `fixedInTag` present + `affectedServices` non-empty → `update-image`
- If severity HIGH/CRITICAL → set `cveApprovalRequired = true` in output
- Otherwise → escalate

Remediate worker:
- For each affected service: `lifecycleService.updateServiceImage(appId, serviceId, fixedInTag, tenancyId)`
- Collect all affected node IDs
- On exception → write `.cveEscalationRequired`

- [ ] **Step 4: Run tests, verify pass**

- [ ] **Step 5: Commit**

```
feat(#38): add CveResponseCaseDescriptor — four-phase CVE remediation
```

---

### Task 4: ServiceUpgradeCaseDescriptor

**Files:**
- Create: `app/src/main/java/io/casehub/ops/app/case_/ServiceUpgradeCaseDescriptor.java`
- Test: `app/src/test/java/io/casehub/ops/app/case_/ServiceUpgradeCaseDescriptorTest.java`

**Interfaces:**
- Consumes: `ApplicationLifecycleService.updateServiceImage()` (Task 2), `NodeConvergenceTracker.register()`
- Produces: `CaseDefinition` via `build(ApplicationLifecycleService, NodeConvergenceTracker)` — consumed by CaseDefinitionRegistrar (Task 6)

- [ ] **Step 1: Write failing tests**

Same pattern as Task 3:
- `assessUpgrade` — valid input (serviceId, newImage, applicationId, tenancyId) → `.upgradeAssessment`
- `assessUpgrade` — missing serviceId → `WorkerResult.failed()`
- `assessUpgrade` — missing newImage → `WorkerResult.failed()`
- `executeUpgrade` — happy path → calls `updateServiceImage`, returns `.upgradeExecuted`
- `executeUpgrade` — lifecycle service throws → `.upgradeEscalationRequired`
- `verifyUpgrade` — registers with convergenceTracker
- `escalateUpgrade` — writes `.upgradeStatus = "escalated"`
- Case definition identity: namespace=ops, name=service-upgrade, version=1.0

- [ ] **Step 2: Run tests to verify they fail**

- [ ] **Step 3: Implement ServiceUpgradeCaseDescriptor**

Same structure as CveResponseCaseDescriptor:
- Four capabilities: `assess-upgrade`, `execute-upgrade`, `verify-upgrade`, `escalate-upgrade`
- Four workers
- Three bindings: `.upgradeAssessment` → execute, `.upgradeExecuted` → verify, `.upgradeEscalationRequired` → escalate
- Completion: `.upgradeStatus == "completed" || .upgradeStatus == "escalated"`

- [ ] **Step 4: Run tests, verify pass**

- [ ] **Step 5: Commit**

```
feat(#38): add ServiceUpgradeCaseDescriptor — four-phase service upgrade
```

---

### Task 5: CveStatusObserver + CaseDefinitionRegistrar Update

**Files:**
- Create: `app/src/main/java/io/casehub/ops/app/service/CveStatusObserver.java`
- Modify: `app/src/main/java/io/casehub/ops/app/case_/CaseDefinitionRegistrar.java`
- Test: `app/src/test/java/io/casehub/ops/app/service/CveStatusObserverTest.java`
- Modify: `app/src/test/java/io/casehub/ops/app/case_/CaseDefinitionRegistrarTest.java`

**Interfaces:**
- Consumes: `CveStore.updateStatus()` (Task 1), `CveResponseCaseDescriptor.build()` (Task 3), `ServiceUpgradeCaseDescriptor.build()` (Task 4)
- Produces: CDI observer that updates CVE status on case completion; registrar wires real case descriptors

- [ ] **Step 1: Write failing tests for CveStatusObserver**

Test that when a `CaseStateChangedEvent` with case type `ops:cve-response` arrives
at terminal state (COMPLETED or FAULTED), the observer:
- Extracts cveId from case context
- Calls `CveStore.updateStatus(appId, cveId, RESOLVED)` for COMPLETED
- Calls `CveStore.updateStatus(appId, cveId, ESCALATED)` for FAULTED
- Ignores events for other case types
- Ignores non-terminal states

- [ ] **Step 2: Implement CveStatusObserver**

`@ApplicationScoped` CDI bean with `@ObservesAsync CaseStateChangedEvent`.
Check case definition type, extract context data, call CveStore.

- [ ] **Step 3: Update CaseDefinitionRegistrar**

Replace in `onStartup`:
```java
// Before:
StubChildCaseDescriptor.build("ops", "cve-response", "1.0"),
StubChildCaseDescriptor.build("ops", "service-upgrade", "1.0"),

// After:
CveResponseCaseDescriptor.build(applicationLifecycleService, convergenceTracker),
ServiceUpgradeCaseDescriptor.build(applicationLifecycleService, convergenceTracker),
```

- [ ] **Step 4: Update CaseDefinitionRegistrarTest**

Verify cve-response and service-upgrade have real capabilities (not `*-stub`).

- [ ] **Step 5: Run all tests, verify pass**

Run: `mvn --batch-mode -o test -pl app`

- [ ] **Step 6: Commit**

```
feat(#38): wire real case descriptors, add CveStatusObserver
```

---

### Task 6: ApplicationEventBroadcaster — SSE Infrastructure

**Files:**
- Create: `app/src/main/java/io/casehub/ops/app/service/ApplicationEventBroadcaster.java`
- Test: `app/src/test/java/io/casehub/ops/app/service/ApplicationEventBroadcasterTest.java`

**Interfaces:**
- Consumes: CDI events (`ReconciliationCompletedEvent`, `CaseStateChangedEvent`, `ApplicationStatusChangedEvent`)
- Produces: `subscribe(UUID appId, EventFilter filter, SseEventSink sink, Sse sse)` and `unsubscribe(SseEventSink sink)` — consumed by CaseResource and ReconciliationResource (Task 8)

- [ ] **Step 1: Write failing tests**

Test the `ApplicationEventBroadcaster`:
- `publish(appId, cloudEvent)` stores in ring buffer
- `subscribe(appId, ALL, sink)` receives all events
- `subscribe(appId, CASE, sink)` receives only case events
- `subscribe(appId, RECONCILIATION, sink)` receives only reconciliation events
- Ring buffer eviction at capacity (1000 default)
- `replayFrom(lastEventId)` replays from buffer position
- Gap event sent when `lastEventId` not found in buffer
- `unsubscribe` removes the sink

- [ ] **Step 2: Implement ApplicationEventBroadcaster**

`@ApplicationScoped` CDI bean.

**EventFilter enum:** `ALL`, `CASE`, `RECONCILIATION`

**Ring buffer:** `ConcurrentHashMap<UUID, RingBuffer>` — one per application.
`RingBuffer` is a simple circular array of CloudEvent objects with an ID index
for `Last-Event-ID` lookup.

**CDI observers:** Three `@ObservesAsync` methods that call `publish()`:
- Reconciliation events → CloudEvent type `io.casehub.desiredstate.reconciliation.*`
- Case events → CloudEvent type `io.casehub.ops.app.case.*`
- Application events → CloudEvent type `io.casehub.ops.app.status.*`

**subscribe():** Creates an SSE sink subscription with the filter. On each
published event matching the filter, sends an SSE event with CloudEvent JSON
body, `event:` = CloudEvent type, `id:` = CloudEvent id.

**Gap detection:** When `Last-Event-ID` is provided but not in the ring buffer,
send a synthetic event with `event: io.casehub.ops.app.stream.gap` before
replaying from the oldest buffered event.

- [ ] **Step 3: Run tests, verify pass**

- [ ] **Step 4: Commit**

```
feat(#38): add ApplicationEventBroadcaster — SSE with ring buffer and gap detection
```

---

### Task 7: ScalingService Extraction

**Files:**
- Create: `app/src/main/java/io/casehub/ops/app/service/ScalingService.java`
- Modify: `app/src/main/java/io/casehub/ops/app/rest/ScalingResource.java`
- Test: `app/src/test/java/io/casehub/ops/app/service/ScalingServiceTest.java`
- Modify: `app/src/test/java/io/casehub/ops/app/rest/ScalingResourceTest.java`

**Interfaces:**
- Consumes: `SituationScalingEvaluator`, `Event<ScalingRequestedEvent>`, `ObjectMapper`
- Produces: `ScalingService.scale(String applicationId, UUID engineCaseId, ApplicationStatus status, String servicesJson, String serviceId, ScaleServiceRequest request) → Response` — consumed by ScalingResource and ServiceOperationResource (Task 8)

- [ ] **Step 1: Write failing tests for ScalingService**

Extract the scaling logic tests from `ScalingResourceTest`. The service-layer
tests verify: cooldown checking, policy evaluation, CDI event emission,
validation (unknown service, missing fields).

- [ ] **Step 2: Extract ScalingService from ScalingResource**

Move the `doScale()` method and its helper methods (`maxCooldownForService`,
`mergedPolicy`) from `ScalingResource` into a new `ScalingService` bean.
The `ScalingResource` becomes a thin REST wrapper delegating to `ScalingService`.

Use `ide_insert_member` for the new class, then modify `ScalingResource`
to inject and delegate to `ScalingService`.

- [ ] **Step 3: Update ScalingResourceTest**

Verify `ScalingResource` still works through the service delegation.

- [ ] **Step 4: Run all tests, verify pass**

Run: `mvn --batch-mode -o test -pl app -Dtest=ScalingServiceTest,ScalingResourceTest`

- [ ] **Step 5: Commit**

```
refactor(#38): extract ScalingService from ScalingResource
```

---

### Task 8: REST Endpoint Wiring

**Files:**
- Modify: `app/src/main/java/io/casehub/ops/app/rest/ApplicationResource.java`
- Modify: `app/src/main/java/io/casehub/ops/app/rest/CaseResource.java`
- Modify: `app/src/main/java/io/casehub/ops/app/rest/DeploymentResource.java`
- Modify: `app/src/main/java/io/casehub/ops/app/rest/ReconciliationResource.java`
- Modify: `app/src/main/java/io/casehub/ops/app/rest/SecurityResource.java`
- Modify: `app/src/main/java/io/casehub/ops/app/rest/ServiceOperationResource.java`
- Create: `app/src/main/java/io/casehub/ops/app/rest/dto/UpgradeServiceRequest.java`
- Create: `app/src/main/java/io/casehub/ops/app/rest/dto/RollbackRequest.java`
- Test: `app/src/test/java/io/casehub/ops/app/rest/CaseResourceTest.java`
- Test: `app/src/test/java/io/casehub/ops/app/rest/ReconciliationResourceTest.java`
- Test: `app/src/test/java/io/casehub/ops/app/rest/SecurityResourceTest.java`
- Test: `app/src/test/java/io/casehub/ops/app/rest/ServiceOperationResourceTest.java`
- Modify: `app/src/test/java/io/casehub/ops/app/rest/DeploymentResourceTest.java`

**Interfaces:**
- Consumes: `ApplicationLifecycleService` (Task 2), `CveStore` (Task 1), `ApplicationEventBroadcaster` (Task 6), `ScalingService` (Task 7), `ServiceCaseRegistry`, `CaseHubRuntime`, `ReconciliationLoop`, `KubernetesEventSource`

This task wires all remaining stubbed endpoints. Work through each resource
sequentially: write tests, then wire the implementation.

- [ ] **Step 1: Wire ApplicationResource PUT**

Test: PUT with valid request updates the application and returns 200.
Implementation: Load entity, update name/description/servicesJson, if not DRAFT
recompile desired state. Follow existing `create` pattern.

- [ ] **Step 2: Wire DeploymentResource list/current/rollback**

Tests: GET list returns DeploymentRecordEntity list, GET current returns latest
deployment + reconciliation state, POST rollback delegates to
`lifecycleService.rollbackToDeployment()`.

- [ ] **Step 3: Wire CaseResource list/get/events**

Inject `CaseHubRuntime` and `ApplicationEventBroadcaster`.
- `listCases`: query by `ApplicationEntity.engineCaseId` → return case + child case summaries
- `getCase`: query by caseId → return detail view
- `streamEvents`: call `broadcaster.subscribe(appId, CASE, sinkContext)`

Tests: list returns cases, get returns case detail, events endpoint returns SSE content type.

- [ ] **Step 4: Wire ReconciliationResource status/trigger/events**

Inject `ReconciliationLoop`, `KubernetesEventSource`, `ApplicationEventBroadcaster`.
- `getStatus`: query ReconciliationLoop for per-cluster node counts
- `trigger`: call `KubernetesEventSource.emitDrift()` for all active loop keys
- `streamEvents`: call `broadcaster.subscribe(appId, RECONCILIATION, sinkContext)`

Tests: status returns per-cluster data, trigger returns 202, events returns SSE.

- [ ] **Step 5: Wire SecurityResource cves/posture**

Inject `CveStore`, `CaseHubRuntime`, `ServiceCaseRegistry`.
- `getCves`: `CveStore.findByApplicationId(id)`
- `scanCves`: store in CveStore + signal case via `CaseHubRuntime.signal()`
- `getPosture`: query `ServiceCaseRegistry` for dimension statuses on COMPLIANCE/SECURITY

Tests: getCves returns list, scanCves stores + signals, getPosture aggregates.

- [ ] **Step 6: Wire ServiceOperationResource status/scale/upgrade**

Inject `ServiceCaseRegistry`, `ScalingService`, `CaseHubRuntime`.
- `getStatus`: query `ServiceCaseRegistry.getByServiceId()` → dimension view
- `scale`: delegate to `ScalingService`
- `upgrade`: signal case with `upgradeRequested` context

Tests: status returns dimensions, scale delegates correctly, upgrade signals case.

- [ ] **Step 7: Run all app tests**

Run: `mvn --batch-mode -o test -pl app`

- [ ] **Step 8: Commit**

```
feat(#38): wire all stubbed REST endpoints to real services
```

---

### Task 9: blocks-ui Components

**Pre-requisite:** Add blocks-ui to slot 86:
```
/work-slot add-repo blocks-ui
```

This task creates five new Lit Web Components in the blocks-ui repo. Each
component follows the established convention: Lit 3.x, Shadow DOM,
`DataSourceMixin` or SSE for data, `emitPagesEvent` for events, vitest
for tests, `data` prop for mocks.

**Files (per component):**
- `components/<name>/package.json`
- `components/<name>/tsconfig.json`
- `components/<name>/tsconfig.build.json`
- `components/<name>/vitest.config.ts`
- `components/<name>/src/<name>.ts`
- `components/<name>/src/<name>.test.ts`
- `components/<name>/src/index.ts`
- `components/<name>/src/types.ts` (if needed)

**Each component follows the same pattern:**

1. Create directory and package.json (copy from `compliance-summary` as template,
   update name/description/dependencies)
2. Create tsconfig files (copy from existing component)
3. Define types in `types.ts`
4. Write failing test — component renders with data prop, emits events
5. Implement component class extending `DataSourceMixin(LitElement)` or `LitElement`
6. Run tests: `cd components/<name> && npx vitest run`
7. Commit

#### 9a: blocks-service-card

Renders a service status card with health badge, replicas, image, per-cluster status.
Props: `data?: ServiceCardData`, `endpoint?: string`.
Event: `service-card.selected`.

#### 9b: blocks-cluster-panel

Renders a cluster list with registration form and connectivity test.
Props: `data?: ClusterInfo[]`, `endpoint?: string`, `readonly?: boolean`.
Events: `cluster.registered`, `cluster.deleted`, `cluster.tested`.

#### 9c: blocks-reconciliation-status

Renders desired vs actual diff with per-node status indicators. Supports SSE live updates.
Props: `data?: ReconciliationSnapshot`, `endpoint?: string`, `sseEndpoint?: string`.
Events: `reconciliation.node-selected`, `reconciliation.trigger-requested`.

#### 9d: blocks-dimension-dashboard

Renders N dimensions with status indicators and severity badges. Compact layout option.
Props: `data?: DimensionDashboardData`, `endpoint?: string`, `compact?: boolean`.
Event: `dimension.selected`.

#### 9e: blocks-topology-viewer

Renders service dependency DAG using `graph-renderer` + `computeElkLayout`. Extends
`blocks-dag-viewer` pattern with custom node types for service status badges.
Props: `data?: TopologySnapshot`, `endpoint?: string`, `sseEndpoint?: string`.
Event: `topology.node-selected`.
Dependencies: `@casehubio/graph-core`, `@casehubio/graph-renderer`.

- [ ] **Step 1: Create blocks-service-card (9a)**
- [ ] **Step 2: Create blocks-cluster-panel (9b)**
- [ ] **Step 3: Create blocks-reconciliation-status (9c)**
- [ ] **Step 4: Create blocks-dimension-dashboard (9d)**
- [ ] **Step 5: Create blocks-topology-viewer (9e)**
- [ ] **Step 6: Run all component tests**

```bash
cd /Users/mdproctor/claude/casehub/blocks-ui
npx vitest run --project service-card --project cluster-panel --project reconciliation-status --project dimension-dashboard --project topology-viewer
```

- [ ] **Step 7: Commit**

```
feat(#38): add five ops building blocks — service-card, cluster-panel, reconciliation-status, dimension-dashboard, topology-viewer
```

---

## Dependency Graph

```
Task 1 (CveStore) ──────────────────┐
Task 2 (updateServiceImage) ────────┤
                                    ├─→ Task 3 (CveResponseDescriptor)
                                    ├─→ Task 4 (ServiceUpgradeDescriptor)
                                    │          │
                                    │          ▼
                                    ├─→ Task 5 (Observer + Registrar) ─→ Task 8 (REST wiring)
                                    │
Task 6 (EventBroadcaster) ─────────┤
Task 7 (ScalingService) ───────────┘

Task 9 (blocks-ui) — independent, can run any time after slot add-repo
```

Tasks 1, 2, 6, 7, 9 have no dependencies on each other.
Tasks 3, 4 depend on Task 2.
Task 5 depends on Tasks 1, 3, 4.
Task 8 depends on Tasks 1, 2, 5, 6, 7.
