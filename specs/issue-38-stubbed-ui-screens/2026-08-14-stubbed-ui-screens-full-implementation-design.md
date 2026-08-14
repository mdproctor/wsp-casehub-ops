# Stubbed UI Screens — Full Implementation Design

Issue: #38
Epic: #29 (Service lifecycle management)
Branch: `issue-38-stubbed-ui-screens`

## Summary

Replace all stubbed REST endpoints in the ops console `app/` module with real service
wiring, build five new blocks-ui components for ops-domain building blocks, and replace
the two remaining `StubChildCaseDescriptor` stubs (cve-response, service-upgrade) with
full case descriptors.

Three workstreams:
1. **Backend wiring** — wire stubbed REST endpoints to real services in casehub-ops
2. **Case descriptors** — replace `ops:cve-response` and `ops:service-upgrade` stubs
3. **blocks-ui components** — five new generic building blocks in casehub/blocks-ui

Cross-repo: blocks-ui added to slot 86 alongside ops.

## Backend Wiring

### Endpoints Already Real (no changes needed)

| Resource | Endpoints |
|----------|-----------|
| `ClusterResource` | All (register, list, get, delete, testConnectivity) |
| `ApprovalResource` | All (list, get, approve, reject) — wired to WorkItemService + PlanStore |
| `ScalingResource` | All — wired to SituationScalingEvaluator + CDI events |

### Endpoints to Wire

#### ApplicationResource

| Endpoint | Current | Change |
|----------|---------|--------|
| `PUT /{id}` | Returns empty 200 | Wire to `ApplicationLifecycleService` — update application definition, recompile desired state per cluster if status is not DRAFT |

#### DeploymentResource

| Endpoint | Current | Change |
|----------|---------|--------|
| `GET /` | Returns `List.of()` | Query `DeploymentRecordEntity.findByApplicationId(id)` |
| `GET /current` | Returns empty 200 | Query latest `DeploymentRecordEntity` by applicationId + per-cluster reconciliation state from `ReconciliationLoop` |
| `POST /rollback` | Returns 202 | Accept `RollbackRequest(deploymentId)`, load target `DeploymentRecordEntity`, restore the full topology snapshot (services JSON) from that record onto the `ApplicationEntity`, recompile desired state per cluster, record new `DeploymentRecordEntity` with trigger=ROLLBACK. Uses a new `rollbackToDeployment(UUID appId, UUID deploymentId, String tenancyId)` method on `ApplicationLifecycleService`. **Interaction with per-service mutations:** rollback restores the full snapshot, silently undoing any post-deployment per-service changes (image updates, config changes, replica adjustments). This is intentional — rollback means "return to this known-good state." Active child cases (CVE remediation, upgrade) that are still modifying services will detect the spec change on their next convergence check and either re-apply their mutation or escalate. |

#### CaseResource

| Endpoint | Current | Change |
|----------|---------|--------|
| `GET /` | Returns `List.of()` | Query engine `CaseHubRuntime` for the application case + child cases. Inject `CaseHubRuntime`, query by application case ID (stored on `ApplicationEntity`). Return case summaries (id, type, status, createdAt). |
| `GET /{caseId}` | Returns empty 200 | Query `CaseHubRuntime` for case detail — state, blackboard summary, timeline. Return structured view. |
| `GET /events` (SSE) | Returns empty 200 | Wire to `ApplicationEventBroadcaster` (D6), all events filter |

#### ReconciliationResource

| Endpoint | Current | Change |
|----------|---------|--------|
| `GET /status` | Returns empty 200 | Query `ReconciliationLoop` for per-cluster desired vs actual state. Return `ReconciliationStatusView(clusters: List<ClusterReconciliationStatus>)` with nodeCount, convergedCount, faultedCount, driftedCount per cluster. |
| `POST /trigger` | Returns 202 | Call `KubernetesEventSource.emitDrift()` to trigger immediate reconciliation cycle for all clusters of this application |
| `GET /events` (SSE) | Returns empty 200 | Wire to `ApplicationEventBroadcaster` (D6), reconciliation events filter |

#### SecurityResource

| Endpoint | Current | Change |
|----------|---------|--------|
| `GET /cves` | Returns `List.of()` | Query `CveStore.findByApplicationId(id)`. Return list of `CveRecord` with cveId, severity, affectedImage, affectedServices, fixedInTag, status, createdAt. |
| `POST /cves` | Returns 202 | Accept `CveEvent`, write to `CveStore`, signal application case via `CaseHubRuntime.signal(appCaseId, "cveDetected", cveData)`. Returns 202. |
| `GET /posture` | Returns empty 200 | Query `ServiceCaseRegistry` for the application's service cases, aggregate dimension statuses across COMPLIANCE and SECURITY dimensions. Return per-framework posture summary. |

#### ServiceOperationResource

| Endpoint | Current | Change |
|----------|---------|--------|
| `GET /status` | Returns empty 200 | Query `ServiceCaseRegistry.getByServiceId(serviceId)` for the service's `ServiceCaseContext`. Return per-dimension status, active child cases, service health. |
| `POST /scale` | Returns 202 | Accept `ScaleServiceRequest`, delegate to `ScalingService` (new service bean extracted from `ScalingResource`'s scaling logic — cooldown checks, policy evaluation, CDI event emission). Both `ScalingResource` and `ServiceOperationResource` delegate to the same service bean. |
| `POST /upgrade` | Returns 202 | Accept `UpgradeServiceRequest(newImage)`, signal application case via `CaseHubRuntime.signal(appCaseId, "upgradeRequested", upgradeSpec)`. The binding triggers `ops:service-upgrade` child case. |

### New ApplicationLifecycleService Method

```java
public Set<String> updateServiceImage(UUID applicationId, String serviceId,
                                       String newImage, String tenancyId)
```

1. Load `ApplicationEntity` by ID and tenancyId
2. Find the service in the application's service definitions
3. Update the service's `image` field
4. Persist the updated Application
5. Recompile desired-state graph per cluster via `compileForCluster()`
6. Call `ReconciliationLoop.updateDesired()` per cluster
7. Return the set of affected node IDs

Same persistence and recompilation pattern as `updateServiceConfig()` and
`updateServiceReplicas()`. The image update flows through the desiredstate
reconciliation loop — the runtime detects the spec change (image differs),
plans a re-provision, and the `KubernetesNodeProvisioner` applies a rolling
update via fabric8.

## SSE Event Streaming

### ApplicationEventBroadcaster

New service in `app/src/main/java/io/casehub/ops/app/service/`.

`@ApplicationScoped` CDI bean aggregating three event sources into a single
SSE stream per application:

**Event sources:**
- Reconciliation events — `ReconciliationLoop` completion callbacks (new wiring via
  CDI `@ObservesAsync ReconciliationCompletedEvent`)
- Case lifecycle — `@ObservesAsync CaseStateChangedEvent` from engine
- Application lifecycle — `@ObservesAsync ApplicationStatusChangedEvent` (existing CDI pattern)

**Ring buffer:** `ConcurrentHashMap<UUID, RingBuffer<CloudEvent>>` — one ring buffer per
applicationId. Default capacity 1000, configurable via `@ConfigProperty`.

**Reconnection:** Client sends `Last-Event-ID` header. If the ID is found in the ring
buffer, replay from that point. If evicted, send a synthetic `gap` event
(`type: io.casehub.ops.app.stream.gap`) before streaming from the oldest buffered event.
The client uses the gap event to trigger a full state refresh via the REST API.

**CloudEvents format:** All events use CloudEvents JSON envelope. SSE `event:` field
carries the CloudEvent `type`. SSE `id:` field carries the CloudEvent `id` (UUID).

**Filter enum:** `ALL`, `CASE`, `RECONCILIATION`

**Filtered views:**
- `CaseResource.streamEvents()` → `broadcaster.subscribe(appId, CASE)`
- `ReconciliationResource.streamEvents()` → `broadcaster.subscribe(appId, RECONCILIATION)`

## CVE Registry

### CveStore (app-scoped)

Interface and implementation both in `app/` — no cross-domain consumers.

```java
// app/src/main/java/io/casehub/ops/app/persistence/CveStore.java
public interface CveStore {
    void store(CveRecord record);
    List<CveRecord> findByApplicationId(UUID applicationId);
    List<CveRecord> findByServiceId(UUID applicationId, String serviceId);
    Optional<CveRecord> findByCveId(UUID applicationId, String cveId);
    void updateStatus(UUID applicationId, String cveId, CveStatus newStatus);
}
```

### CveRecord

```java
public record CveRecord(
    String cveId,
    CveSeverity severity,
    String affectedImage,
    List<String> affectedServices,
    String fixedInTag,          // null if no fix known
    CveStatus status,           // DETECTED, REMEDIATING, RESOLVED, ESCALATED
    UUID applicationId,
    String tenancyId,
    Instant detectedAt
) {}
```

### CveStatus enum

`DETECTED` → `REMEDIATING` (case spawned) → `RESOLVED` (convergence verified) or `ESCALATED` (no fix / remediation failed)

### JpaCveStore + CveEntity

JPA entity with Flyway migration. Named datasource (`quarkus.datasource.app.*`) following
existing convention (`ApprovalPlanEntity` uses the same datasource).

### Data flow

1. `SecurityResource.scanCves(CveEvent)` → `CveStore.store(record)` + `CaseHubRuntime.signal(appCaseId, "cveDetected", cveData)`
2. Case assess worker reads from signal payload (case context), NOT from CveStore
3. When case completes → `CveStore.updateStatus(appId, cveId, RESOLVED/ESCALATED)` (via case completion observer)
4. `SecurityResource.getCves()` → `CveStore.findByApplicationId(id)` for the list view
5. `CveStatusObserver` (`@ObservesAsync CaseStateChangedEvent`) → when `ops:cve-response`
   case reaches terminal state, extracts cveId from case context and calls
   `CveStore.updateStatus(appId, cveId, RESOLVED/ESCALATED)`

## Case Descriptors

### CveResponseCaseDescriptor (`ops:cve-response`)

**Case identity:** `ops:cve-response` v1.0

**Entry point (unchanged — binding already exists):**

| Source | Binding | Trigger | Input mapping |
|--------|---------|---------|---------------|
| `ApplicationCaseDescriptor` | `on-cve-detected` | `.cveDetected` | `.cveData` |
| `ServiceCaseDescriptor` | `security:cve-detected` | `.security.cveDetected` | `.security.cveData` |

**Capabilities:**
- `assess-cve` — classify CVE, find affected services, determine remediation
- `remediate-cve` — update service image via ApplicationLifecycleService
- `verify-cve` — track node convergence
- `escalate-cve` — produce human review summary

**Workers:**

| Worker | Capability | Input | Output |
|--------|-----------|-------|--------|
| `cve-assess-worker` | `assess-cve` | `Map` (CVE data) | `.cveAssessment` or `.cveEscalationRequired` |
| `cve-remediate-worker` | `remediate-cve` | `Map` (assessment) | `.cveRemediationExecuted` |
| `cve-verify-worker` | `verify-cve` | `Map` (execution result) | (NodeConvergenceTracker) |
| `cve-escalate-worker` | `escalate-cve` | `Map` (assessment + diagnostics) | `.cveStatus = "escalated"` |

**Bindings (context change chaining):**

| Binding | Trigger | Target |
|---------|---------|--------|
| `on-cve-assessment` | `.cveAssessment` | `remediate-cve` |
| `on-cve-remediation-executed` | `.cveRemediationExecuted` | `verify-cve` |
| `on-cve-escalation-required` | `.cveEscalationRequired` | `escalate-cve` |

**Completion:** `.cveStatus == "resolved" || .cveStatus == "escalated"`

**Assessment logic:**

| Condition | Action |
|-----------|--------|
| `fixedInTag` present + affected services found | `update-image` — write `.cveAssessment` |
| `fixedInTag` null (no fix known) | `escalate` — write `.cveEscalationRequired` |
| No affected services found (image not in any service) | `escalate` — write `.cveEscalationRequired` |
| Missing required fields (cveId, severity, affectedImage) | `WorkerResult.failed()` |

**Approval gate:** When `severity` is HIGH or CRITICAL, the assess worker writes
`.cveApprovalRequired = true` to the case context. The case definition includes an
`ActionRiskClassifier` that maps this flag to a WorkItem via `HumanTaskScheduleHandler`.
The human approves the remediation plan before the remediate-cve worker runs.

**Remediation:** `ApplicationLifecycleService.updateServiceImage(appId, serviceId, fixedInTag, tenancyId)`
for each affected service. Returns affected node IDs. On exception → `.cveEscalationRequired`.

**Verification:** `convergenceTracker.register(caseId, affectedNodeIds, "cveStatus", "resolved")`

### ServiceUpgradeCaseDescriptor (`ops:service-upgrade`)

**Case identity:** `ops:service-upgrade` v1.0

**Entry point (unchanged — binding already exists):**

| Source | Binding | Trigger | Input mapping |
|--------|---------|---------|---------------|
| `ApplicationCaseDescriptor` | `on-upgrade-requested` | `.upgradeRequested` | `.upgradeSpec` |

**Capabilities:**
- `assess-upgrade` — validate input, confirm service exists
- `execute-upgrade` — update service image
- `verify-upgrade` — track node convergence
- `escalate-upgrade` — produce summary on failure

**Workers:**

| Worker | Capability | Input | Output |
|--------|-----------|-------|--------|
| `upgrade-assess-worker` | `assess-upgrade` | `Map` (upgrade spec) | `.upgradeAssessment` |
| `upgrade-execute-worker` | `execute-upgrade` | `Map` (assessment) | `.upgradeExecuted` |
| `upgrade-verify-worker` | `verify-upgrade` | `Map` (execution result) | (NodeConvergenceTracker) |
| `upgrade-escalate-worker` | `escalate-upgrade` | `Map` (assessment + error) | `.upgradeStatus = "escalated"` |

**Bindings:**

| Binding | Trigger | Target |
|---------|---------|--------|
| `on-upgrade-assessment` | `.upgradeAssessment` | `execute-upgrade` |
| `on-upgrade-executed` | `.upgradeExecuted` | `verify-upgrade` |
| `on-upgrade-escalation-required` | `.upgradeEscalationRequired` | `escalate-upgrade` |

**Completion:** `.upgradeStatus == "completed" || .upgradeStatus == "escalated"`

**Input data shape:**

```json
{
  "serviceId": "order-api",
  "newImage": "quay.io/app/orders:2.0.0",
  "applicationId": "uuid",
  "tenancyId": "tenant-1"
}
```

**Upgrade execution:** `ApplicationLifecycleService.updateServiceImage(appId, serviceId, newImage, tenancyId)`

**Verification:** `convergenceTracker.register(caseId, affectedNodeIds, "upgradeStatus", "completed")`

### Dependencies

Both descriptors: `build(ApplicationLifecycleService, NodeConvergenceTracker)` — same
injection pattern as `IncidentResponseCaseDescriptor`.

### CaseDefinitionRegistrar changes

Replace the two stub registrations:

```java
// Before:
StubChildCaseDescriptor.build("ops", "cve-response", "1.0"),
StubChildCaseDescriptor.build("ops", "service-upgrade", "1.0"),

// After:
CveResponseCaseDescriptor.build(applicationLifecycleService, convergenceTracker),
ServiceUpgradeCaseDescriptor.build(applicationLifecycleService, convergenceTracker),
```

## blocks-ui Components

### 1. blocks-service-card

Per-service status card showing health, replicas, image, and per-cluster deployment status.

**Package:** `@casehubio/blocks-ui-service-card`

**Props:**
- `serviceName: string` — display name
- `serviceId: string` — unique identifier
- `image: string` — container image reference
- `replicas: number` — desired replica count
- `status: string` — overall health status (RUNNING, DEGRADED, etc.)
- `clusters: ClusterDeploymentStatus[]` — per-cluster status array
- `endpoint?: string` — REST endpoint for live data (alternative to props)
- `data?: ServiceCardData` — full data object (alternative to endpoint)

**ClusterDeploymentStatus:**
```typescript
interface ClusterDeploymentStatus {
  clusterId: string;
  clusterName: string;
  status: 'converged' | 'provisioning' | 'faulted' | 'unknown';
  readyReplicas: number;
  desiredReplicas: number;
}
```

**Events:** `service-card.selected` (emitted on click)

**Mock support:** `data` prop accepts pre-built data for testing and showcase.

### 2. blocks-cluster-panel

Cluster management panel: list registered clusters, register new clusters,
test connectivity, deregister.

**Package:** `@casehubio/blocks-ui-cluster-panel`

**Props:**
- `endpoint: string` — REST base URL for cluster operations
- `data?: ClusterInfo[]` — static data for mocks
- `readonly: boolean` — hide mutation actions

**ClusterInfo:**
```typescript
interface ClusterInfo {
  id: string;
  name: string;
  apiUrl: string;
  namespace: string;
  type: 'KUBERNETES' | 'OPENSHIFT';
  status: 'CONNECTED' | 'UNREACHABLE' | 'UNKNOWN';
  applicationCount: number;
}
```

**Sub-components:**
- Cluster list with connectivity badges
- Registration form (name, apiUrl, namespace, type, credentials)
- Connectivity test with status indicator

**Events:** `cluster.registered`, `cluster.deleted`, `cluster.tested`

### 3. blocks-reconciliation-status

Desired vs actual state diff per cluster, showing node-level status.

**Package:** `@casehubio/blocks-ui-reconciliation-status`

**Props:**
- `endpoint: string` — REST endpoint for reconciliation status
- `sseEndpoint?: string` — SSE endpoint for live updates
- `data?: ReconciliationSnapshot` — static data for mocks

**ReconciliationSnapshot:**
```typescript
interface ReconciliationSnapshot {
  clusters: ClusterReconciliationStatus[];
  lastReconciled?: string; // ISO timestamp
}

interface ClusterReconciliationStatus {
  clusterId: string;
  clusterName: string;
  nodeCount: number;
  convergedCount: number;
  driftedCount: number;
  faultedCount: number;
  nodes: NodeReconciliationStatus[];
}

interface NodeReconciliationStatus {
  nodeId: string;
  nodeType: string;
  desired: string;  // summary of desired spec
  actual: string;   // summary of actual state
  status: 'CONVERGED' | 'DRIFTED' | 'FAULTED' | 'PROVISIONING' | 'ABSENT';
}
```

**Live updates:** Uses `SSEManager` to subscribe to reconciliation events when
`sseEndpoint` is provided. Updates node statuses in real-time.

**Events:** `reconciliation.node-selected`, `reconciliation.trigger-requested`

### 4. blocks-dimension-dashboard

Multi-dimension status overview with severity aggregation. Displays N
dimensions, each with a status indicator and severity badge.

**Package:** `@casehubio/blocks-ui-dimension-dashboard`

**Props:**
- `endpoint: string` — REST endpoint for dimension data
- `data?: DimensionDashboardData` — static data for mocks
- `compact: boolean` — compact layout for sidebar use

**DimensionDashboardData:**
```typescript
interface DimensionDashboardData {
  serviceId: string;
  serviceName: string;
  overallHealth: string;
  dimensions: DimensionView[];
}

interface DimensionView {
  type: string;        // e.g., "health", "security", "compliance"
  label: string;       // display label
  status: string;      // current status value
  severity: 'OK' | 'LOW' | 'MEDIUM' | 'HIGH' | 'CRITICAL';
  activeResponses: number;  // count of active child cases
  lastUpdated?: string;
}
```

**Events:** `dimension.selected` (emitted on dimension click, carries dimension type)

### 5. blocks-topology-viewer

Service dependency DAG visualisation using the existing graph rendering stack.

**Package:** `@casehubio/blocks-ui-topology-viewer`

**Props:**
- `data?: TopologySnapshot` — static topology data for mocks
- `endpoint?: string` — REST endpoint for topology data
- `sseEndpoint?: string` — SSE endpoint for live status updates
- `selectionTopic?: string` — event topic for node selection

**TopologySnapshot:**
```typescript
interface TopologySnapshot {
  services: TopologyNode[];
  edges: TopologyEdge[];
}

interface TopologyNode {
  id: string;
  name: string;
  status: 'RUNNING' | 'DEGRADED' | 'DEPLOYING' | 'FAULTED' | 'ABSENT';
  replicas?: number;
  image?: string;
  type?: string; // for icon selection
}

interface TopologyEdge {
  source: string;  // service ID
  target: string;  // service ID
  label?: string;
}
```

**Rendering:** Extends `blocks-dag-viewer` with a pluggable node renderer for service
status badges. The DAG viewer already uses `graph-renderer` (React Flow Lit-wrapped) +
`computeElkLayout`. Topology-specific rendering (health colour coding, replica badges)
is provided via custom React Flow node types registered through the existing stencil
pattern — no separate `graph-stencil-topology` package needed.

**Events:** `topology.node-selected` (carries serviceId)

**Live updates:** When `sseEndpoint` is provided, subscribes via `SSEManager` and
updates node status/colours in real-time without re-layout.

## Flyway Migrations

### V6__cve_record.sql

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

## File Changes

### casehub-ops (app/ module)

| File | Change |
|------|--------|
| `app/.../rest/ApplicationResource.java` | Wire PUT to lifecycle service |
| `app/.../rest/CaseResource.java` | Wire to CaseHubRuntime + ApplicationEventBroadcaster |
| `app/.../rest/DeploymentResource.java` | Wire list/current/rollback to entities + lifecycle service |
| `app/.../rest/ReconciliationResource.java` | Wire to ReconciliationLoop + ApplicationEventBroadcaster |
| `app/.../rest/SecurityResource.java` | Wire to CveStore + ServiceCaseRegistry |
| `app/.../rest/ServiceOperationResource.java` | Wire to ServiceCaseRegistry + lifecycle service |
| `app/.../service/ApplicationLifecycleService.java` | Add `updateServiceImage()` |
| `app/.../service/ApplicationEventBroadcaster.java` | **New** — SSE event aggregation |
| `app/.../persistence/CveStore.java` | **New** — CVE store interface |
| `app/.../persistence/JpaCveStore.java` | **New** — JPA implementation |
| `app/.../entity/CveEntity.java` | **New** — JPA entity |
| `app/.../case_/CveResponseCaseDescriptor.java` | **New** — full case descriptor |
| `app/.../case_/ServiceUpgradeCaseDescriptor.java` | **New** — full case descriptor |
| `app/.../case_/CaseDefinitionRegistrar.java` | Replace stubs with real descriptors |
| `app/.../service/CveStatusObserver.java` | **New** — CDI observer updating CveStore on case completion |
| `app/.../service/ScalingService.java` | **New** — extracted from ScalingResource, shared with ServiceOperationResource |
| `app/.../rest/ScalingResource.java` | **Modified** — delegate to ScalingService |
| `app/src/main/resources/db/app/migration/V6__cve_record.sql` | **New** — CVE table |

### casehub-ops (test files)

| File | Change |
|------|--------|
| `app/.../rest/CaseResourceTest.java` | **New** — tests for wired endpoints |
| `app/.../rest/ReconciliationResourceTest.java` | **New** — tests for wired endpoints |
| `app/.../rest/SecurityResourceTest.java` | **New** — tests for wired endpoints |
| `app/.../rest/ServiceOperationResourceTest.java` | **New** — tests for wired endpoints |
| `app/.../service/ApplicationEventBroadcasterTest.java` | **New** — ring buffer, filtering, gap event |
| `app/.../persistence/JpaCveStoreTest.java` | **New** — CRUD tests |
| `app/.../case_/CveResponseCaseDescriptorTest.java` | **New** — worker logic tests |
| `app/.../case_/ServiceUpgradeCaseDescriptorTest.java` | **New** — worker logic tests |
| `app/.../service/ApplicationLifecycleServiceTest.java` | Add tests for `updateServiceImage()` |
| `app/.../case_/CaseDefinitionRegistrarTest.java` | Verify real capabilities, not stubs |
| `app/.../service/CveStatusObserverTest.java` | **New** — observer correctly updates CveStore |
| `app/.../service/ScalingServiceTest.java` | **New** — extracted scaling logic tests |

### blocks-ui (new components)

| Directory | Package |
|-----------|---------|
| `components/service-card/` | `@casehubio/blocks-ui-service-card` |
| `components/cluster-panel/` | `@casehubio/blocks-ui-cluster-panel` |
| `components/reconciliation-status/` | `@casehubio/blocks-ui-reconciliation-status` |
| `components/dimension-dashboard/` | `@casehubio/blocks-ui-dimension-dashboard` |
| `components/topology-viewer/` | `@casehubio/blocks-ui-topology-viewer` |

Each component follows blocks-ui conventions: Lit 3.x, vitest, dual data mode
(endpoint/data props), `emitPagesEvent` for events, `DataSourceMixin` or
`SSEManager` for live data.

## Test Plan

### Case descriptor tests (follow existing pattern)

**CveResponseCaseDescriptorTest:**
- Case definition identity: correct namespace, name, version
- Assess worker — fixedInTag present + services found: produces `update-image`
- Assess worker — fixedInTag null: escalates
- Assess worker — no affected services: escalates
- Assess worker — missing required fields: `WorkerResult.failed()`
- Remediate worker — happy path: calls `updateServiceImage()`, writes output
- Remediate worker — lifecycle service throws: writes `.cveEscalationRequired`
- Verify worker: registers with convergence tracker
- Escalate worker: writes `.cveStatus = "escalated"` and summary

**ServiceUpgradeCaseDescriptorTest:**
- Case definition identity: correct namespace, name, version
- Assess worker — valid input: produces assessment
- Assess worker — unknown service: escalates
- Assess worker — missing fields: `WorkerResult.failed()`
- Execute worker — happy path: calls `updateServiceImage()`, writes output
- Execute worker — lifecycle service throws: writes `.upgradeEscalationRequired`
- Verify worker: registers with convergence tracker
- Escalate worker: writes `.upgradeStatus = "escalated"`

### REST endpoint tests

Each wired endpoint: happy path, not found, authorization (tenancy filter),
edge cases (empty results, missing data).

### ApplicationEventBroadcaster tests

- Subscribe/unsubscribe lifecycle
- Event filtering (all vs reconciliation-only)
- Ring buffer capacity and eviction
- Gap event on stale `Last-Event-ID`
- Concurrent subscribe from multiple clients

### ApplicationLifecycleService tests

- `updateServiceImage` — happy path: image updated, graph recompiled, loop updated
- `updateServiceImage` — unknown service: throws
- `updateServiceImage` — unknown application: throws
- `updateServiceImage` — DRAFT application (no case): updates entity only, no reconciliation

### blocks-ui component tests

Each component: renders with data prop, renders with empty data, emits events,
handles loading/error states. SSE components: subscribes on connect, unsubscribes
on disconnect. Topology viewer: converts TopologySnapshot to graph model.

## Known Limitations

1. **No convergence timeout** — same cross-cutting limitation as all other case types.
   If nodes never converge, cve-response and service-upgrade cases hang.

2. **No concurrent CVE dedup** — multiple CVEs for the same image on the same
   application spawn separate cases. Engine binding dedup is a platform concern
   (GE-20260608-1a56c3).

3. **CVE scanner integration is demonstrative** — `SecurityResource.scanCves()`
   accepts POST'd CVE events. Real integration would be a scanner webhook or
   polling adapter. The REST ingest demonstrates the pattern.

4. **Image field semantics** — `updateServiceImage` updates the full image
   reference (including tag). The service definition uses `image: "repo:tag"`
   format, not separate image/tag fields.
