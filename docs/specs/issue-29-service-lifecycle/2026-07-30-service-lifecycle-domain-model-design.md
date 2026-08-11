# Service Lifecycle Domain Model — Design Spec

**Issue:** casehubio/casehub-ops#30 (parent: #29)
**Date:** 2026-07-30
**Status:** Draft

---

## 1. Problem Statement

A deployed service has ongoing operational concerns — health, compliance, scaling, security, maintenance, drift, upgrades, recurring problems, and eventual decommission. Today these concerns are managed by separate tools with no shared identity, no memory, and no coordination. A CPU spike that's also a compliance violation that's also a scaling signal gets treated as three independent events.

CaseHub's Case abstraction can model a service as a long-lived case: deployed → WAITING → decommissioned. The question is what structure the case needs to carry nine operational dimensions simultaneously, with per-dimension status, memory (incident history, scaling patterns, compliance evidence), and concurrent child case management.

## 2. Core Concept

A service IS a long-lived case. It opens on deploy, sits in WAITING, and closes when decommission completes. Nine operational dimensions describe its ongoing health. Each dimension has its own status vocabulary, its own blackboard section, and can spawn multiple concurrent child cases.

The unique value: **single operational identity with memory.** The 3rd incident on the same service escalates differently than the 1st. A CVE detection that also violates SOC2 triggers responses in both the security and compliance dimensions simultaneously. An operator sees one screen per service showing all nine dimensions — not nine disconnected tools.

## 3. Design Decisions

| # | Decision | Choice | Rationale |
|---|---|---|---|
| D1 | Scenario coverage | All 9 operational scenarios | The domain model should accommodate all scenarios even if child case implementations come in batches |
| D2 | Status model | Per-dimension status vocabularies | Each dimension's vocabulary carries domain-specific meaning. Severity (OK/INFO/WARNING/CRITICAL) is derived for aggregation. |
| D3 | Dimension presence | All 9 always present on every service | No configuration. Dimensions without active monitoring show their default healthy status (HEALTHY, IN_SYNC, COMPLIANT, etc.) — there is no distinct N/A enum value. Uniform model, uniform UI. |
| D4 | Blackboard structure | Sectioned by dimension | Each dimension owns a named section. Sections are independently hydratable (foundation for #31 partial loading). Write isolation: dimensions only write to their own section. Cross-dimension reads allowed (problem management reads health history). |
| D5 | Dimension status composition | Condition + response composite | Status reflects both the underlying condition (from monitoring) AND whether an active response case exists. Distinguishes "broken and nobody knows" from "broken and being fixed." |
| D6 | Child case concurrency | Multiple concurrent per dimension | A dimension can have N active child cases. Dimension status = worst-of-N active responses combined with condition. |
| D7 | Ganglion → dimension mapping | Ganglia independent, dimensions subscribe | Ganglia emit detections without knowing about dimensions. The service case declares bindings: detection type → dimension(s). One detection can feed multiple dimensions (CPU spike → health + scaling). |
| D8 | Decommission model | Uniform dimension | Decommission is dimension #9 with its own status vocabulary. Completion triggers the service case's completion criterion — the only dimension whose completion ends the parent case. |
| D9 | Architectural approach | Dimensions as first-class types | Not child cases (over-engineers), not views over existing primitives (under-engineers). `OperationalDimension` is a purpose-built type in `ops-api` — lightweight, explicit, maps 1:1 to UI. |

## 4. Domain Model

### 4.1 Type Hierarchy

```
ops-api/io.casehub.ops.api.lifecycle
├── DimensionType (enum: 9 values)
├── DimensionStatus (interface: severity(), label(), isTerminal())
├── Severity (enum: OK, INFO, WARNING, CRITICAL)
├── OperationalDimension (type + status + section + activeResponses + subscriptions)
├── DimensionSection (typed blackboard view per dimension)
├── ServiceCaseContext (serviceId + serviceType + dimensions map + metadata)
├── GanglionBinding (situationType + dimension + contextKey + conditionStatus)
├── CaseRef (reference to an active child case)
├── ManagedServiceCategory (enum: APPLICATION, INFRASTRUCTURE, PLATFORM, IOT, COMPLIANCE)
└── status/
    ├── HealthStatus (HEALTHY, DEGRADED, DOWN, INVESTIGATING, REMEDIATING)
    ├── ConfigurationDriftStatus (IN_SYNC, DRIFTED, RECONCILING, RECONCILIATION_FAILED)
    ├── ComplianceStatus (COMPLIANT, STALE_EVIDENCE, NON_COMPLIANT, REMEDIATING, UNAVAILABLE)
    ├── ScalingStatus (OPTIMAL, UNDER_PROVISIONED, OVER_PROVISIONED, SCALING, SCALE_FAILED)
    ├── ChangeManagementStatus (NO_ACTIVITY, UPGRADE_AVAILABLE, CANARY, ROLLING_OUT, ROLLBACK, CHANGE_FAILED)
    ├── SecurityStatus (CLEAR, VULNERABILITY_DETECTED, PATCHING, BREACH_DETECTED, INVESTIGATING)
    ├── MaintenanceStatus (NO_ACTIVITY, SCHEDULED, IN_PROGRESS, OVERDUE, FAILED)
    ├── ProblemManagementStatus (NO_KNOWN_PROBLEMS, PATTERN_DETECTED, INVESTIGATING, WORKAROUND_APPLIED, ROOT_CAUSE_FIXED)
    └── DecommissionStatus (NOT_PLANNED, SCHEDULED, IN_PROGRESS, BLOCKED, COMPLETED)
```

### 4.2 Core Types

**DimensionStatus interface:**

```java
public interface DimensionStatus {
    Severity severity();
    String label();
    boolean isTerminal();  // true only for COMPLETED on DecommissionStatus
}
```

All nine status enums implement this interface. Each enum constant carries a fixed `Severity` mapping. The `severity()` method enables cross-dimension aggregation without losing per-dimension semantics.

**OperationalDimension:**

```java
public class OperationalDimension {
    private final DimensionType type;
    private DimensionStatus status;
    private final DimensionSection section;
    private final List<CaseRef> activeResponses;
    private final List<GanglionBinding> subscriptions;

    public Severity severity() { return status.severity(); }
}
```

**ServiceCaseContext:**

```java
public class ServiceCaseContext {
    private final String serviceId;
    private final String serviceName;
    private final ManagedServiceCategory category;
    private final Map<DimensionType, OperationalDimension> dimensions;  // always 9 entries
    private final Instant deployedAt;
    private final Map<String, Object> metadata;
}
```

**ServiceHealth (aggregate for UI):**

```java
public record ServiceHealth(
    String serviceId,
    String serviceName,
    Map<DimensionType, DimensionStatus> dimensions,
    Severity overallSeverity  // worst-of-9
) {}
```

### 4.3 DimensionSection

Each dimension owns a typed, namespaced view into the service case's engine context. `DimensionSection` projects through `CaseHubRuntime.signal()` and `query()` — it is not a separate store. The engine's existing context persistence handles durability.

```java
public class DimensionSection {
    private final DimensionType type;
    private final String contextPrefix;  // e.g. "health.", "security."

    <T> T get(String key, Class<T> type);  // delegates to runtime.query(caseId, contextPrefix + key)
    void put(String key, Object value);     // delegates to runtime.signal(caseId, contextPrefix + key, value)
    Instant lastUpdated();
}
```

Write isolation: a dimension's handlers only write to their own section (enforced by `contextPrefix`). Cross-dimension reads are permitted — problem management reads health context to detect incident patterns. Section operations delegate to the engine's context store; `DimensionSection` is a typed, namespaced view, not an independent persistence layer.

### 4.4 Status Computation

`DimensionStatusService` computes each dimension's composite status:

| Condition | Active responses | Result |
|---|---|---|
| Unhealthy | None | Condition status (DEGRADED, DRIFTED, NON_COMPLIANT, etc.) |
| Unhealthy | 1+ active | Response status (REMEDIATING, INVESTIGATING, RECONCILING, etc.) |
| Healthy | 1+ still open | Healthy status (condition is primary signal) |
| Healthy | None | Healthy status (HEALTHY, IN_SYNC, COMPLIANT, etc.) |

When multiple child cases are active, the dimension's response status is worst-of-N across the active cases' statuses. Ordering uses `Severity` as primary key (CRITICAL > WARNING > INFO > OK) and enum declaration order as tiebreaker within the same severity. Each status enum declares condition statuses first and response statuses after, making the ordering fully deterministic.

**Condition statuses** are the base states reflecting what monitoring observes (HEALTHY, DEGRADED, DOWN, IN_SYNC, DRIFTED, COMPLIANT, NON_COMPLIANT, etc.). **Response statuses** are the states indicating active remediation (REMEDIATING, INVESTIGATING, RECONCILING, PATCHING, SCALING, etc.). The `DimensionStatusService` selects a single status to display: when an active response case exists, the response status is shown (telling the operator "this is broken AND being fixed"); when no response exists, the condition status is shown (telling the operator "this is broken and nobody is acting"). This single-value model is the D5 composite — the selected status encodes both condition and response information without requiring the UI to render two separate fields.

**Condition derivation:** `GanglionBinding` declares a `conditionStatus` field — the `DimensionStatus` value that each detection type implies (e.g., situationType `"heartbeat-failure"` → `HealthStatus.DOWN`). When `ServiceDetectionBridge` processes a detection, it reads the `conditionStatus` from the matching binding and writes it to `dimensions.<type>.condition` in the engine context via `DimensionSection`. The bridge is generic — it reads the status from the binding declaration, not from hardcoded vocabulary knowledge. Recovery follows the same path: a `"heartbeat-ok"` binding maps to `HealthStatus.HEALTHY`. Bindings without a `conditionStatus` write detection data only (the dimension's condition remains unchanged).

**Engine context write scope:** `DimensionStatusService` writes all 9 dimension statuses to the engine context under `dimensions.<type>.status` on every recomputation. This enables completion predicates (decommission triggers case completion), restart reconstruction (status is durable in engine context), and future cross-dimension bindings. Feedback loop prevention: engine bindings trigger on **detection data keys** (`.health.serviceDown`, `.security.cveDetected`, etc.), never on status keys. The `dimensions.*` namespace is reserved for `DimensionStatusService` writes — no binding's `ContextChangeTrigger` evaluates against it.

### 4.5 Runtime Lifecycle

**ServiceCaseRegistry** (`@ApplicationScoped`, `app` module) manages `ServiceCaseContext` instances in memory, keyed by engine case ID. Follows the same pattern as `DriftSignalBridge` (which uses a `ConcurrentHashMap<String, AppRegistration>` with explicit `registerApplication()`/`deregisterApplication()` calls).

```java
@ApplicationScoped
public class ServiceCaseRegistry {
    private final ConcurrentHashMap<UUID, ServiceCaseContext> contexts = new ConcurrentHashMap<>();

    public void register(UUID caseId, String serviceId, ManagedServiceCategory category, Map<String, Object> metadata);
    public ServiceCaseContext get(UUID caseId);
    public ServiceCaseContext getOrReconstruct(UUID caseId, CaseHubRuntime runtime);
    public void deregister(UUID caseId);
}
```

**Creation:** On deploy (§7.1), `ApplicationLifecycleService` starts the engine case and calls `ServiceCaseRegistry.register()`. The registry creates a `ServiceCaseContext` with 9 dimensions at default status.

**Lookup:** When a `CloudEvent` arrives, `ServiceDetectionBridge` calls `ServiceCaseRegistry.get(caseId)` to locate the `ServiceCaseContext`. The case ID is extracted from the event's correlation data.

**Restart reconstruction:** Service cases live for months or years. On application restart, in-memory state is lost. The registry reconstructs on first access via `getOrReconstruct(caseId, runtime)`:

1. **Condition status:** `runtime.query(caseId, "dimensions.<type>.condition")` — persisted by the bridge on every detection
2. **Composite status:** `runtime.query(caseId, "dimensions.<type>.status")` — persisted by `DimensionStatusService` on every recomputation
3. **Active responses:** `runtime.query(caseId, "dimensions.<type>.activeResponseIds")` — a bounded list of child case IDs, maintained by the bridge alongside `CaseRef` additions/removals
4. **Metadata:** `runtime.query(caseId, "service.*")` — serviceId, category, deployedAt written on creation

All mutable `OperationalDimension` state is projected to the engine context as a matter of course (condition writes by the bridge, status writes by `DimensionStatusService`, response ID tracking by the bridge). Restart reconstruction reads it back. The engine context is the durable store; `ServiceCaseContext` and `OperationalDimension` are the runtime projection.

## 5. Nine Dimensions

Each dimension declares context keys stored in the engine's case context (via `DimensionSection`). Context keys hold bounded coordination data needed by bindings and workers. Operational metrics (uptimePercentage, costPerDay, complianceScore) and full event histories live in external systems (monitoring, compliance database, CMDB); dimensions query them on demand, not store them. History relevant to coordination (e.g., consecutive failure counts for escalation) is bounded — sliding windows or counters, never unbounded arrays.

### 5.1 HEALTH_MONITORING

**Purpose:** Is the service alive and performing within expectations?

**Status:** HEALTHY (OK) · DEGRADED (WARNING) · DOWN (CRITICAL) · INVESTIGATING (INFO) · REMEDIATING (WARNING)

**Typical ganglia:** heartbeat-check, metrics-trend (latency, error rate), log-anomaly, dependency-health

**Typical child cases:** IncidentResponse, MemoryLeakInvestigation, DependencyFailureResponse, AutoRestart

**Context keys:** `lastCheckTime`, `consecutiveFailures`, `lastIncidentId`, `lastIncidentDuration`, `incidentCount` — incident history queried from engine's episodic memory; uptimePercentage from monitoring

**Value:** Incident count enables escalation policies. The 3rd crash in a week triggers a different response than the 1st. Full incident history available via episodic memory, not stored in context.

### 5.2 CONFIGURATION_DRIFT

**Purpose:** Does the actual state match the declared state?

**Status:** IN_SYNC (OK) · DRIFTED (WARNING) · RECONCILING (INFO) · RECONCILIATION_FAILED (CRITICAL)

**Typical ganglia:** config-drift (spec hash comparison), manual-change-detected (audit log watcher)

**Typical child cases:** AutoReconciliation, ManualDriftReview

**Context keys:** `lastReconcileTime`, `lastKnownSpecHash`, `reconcileAttemptCount`, `consecutiveDriftCount`, `chronicDrifter` — drift history in episodic memory

**Value:** Integrates with casehub-desiredstate. Chronic drifters (>3x/week) get a different response — manual review instead of auto-reconcile.

### 5.3 COMPLIANCE

**Purpose:** Does the service meet its regulatory obligations right now?

**Status:** COMPLIANT (OK) · STALE_EVIDENCE (WARNING) · NON_COMPLIANT (CRITICAL) · REMEDIATING (WARNING) · UNAVAILABLE (WARNING)

**Typical ganglia:** evidence-stale, control-violation, framework-change

**Typical child cases:** EvidenceRecollection, ComplianceRemediation, AuditPreparation

**Context keys:** `lastEvidenceCollection`, `auditDueDate`, `activeViolationCount`, `frameworkIds[]` — control results and scores live in `CompliancePostureService`; dimension references by framework ID

**Value:** Bridges `CompliancePostureService` into the service lifecycle. A service is "running AND compliant" or "running AND non-compliant" — both visible simultaneously.

### 5.4 SCALING

**Purpose:** Is the service right-sized for its current load?

**Status:** OPTIMAL (OK) · UNDER_PROVISIONED (WARNING) · OVER_PROVISIONED (INFO) · SCALING (INFO) · SCALE_FAILED (CRITICAL)

**Typical ganglia:** cpu-threshold, memory-threshold, queue-depth, request-latency-trend, cost-anomaly

**Typical child cases:** ScaleOut, ScaleIn, CapacityReview

**Context keys:** `currentReplicas`, `targetReplicas`, `scalingEventsThisWeek`, `lastScaleDirection` — resource utilization and cost from monitoring; scaling history in episodic memory

**Value:** OVER_PROVISIONED is INFO (wasting money, not at risk). Frequent auto-scaling feeds problem management — a service that scales 10x daily has a sizing problem.

### 5.5 CHANGE_MANAGEMENT

**Purpose:** Is a change in progress, and is it going well?

**Status:** NO_ACTIVITY (OK) · UPGRADE_AVAILABLE (INFO) · CANARY (INFO) · ROLLING_OUT (INFO) · ROLLBACK (WARNING) · CHANGE_FAILED (CRITICAL)

**Typical ganglia:** version-check, canary-health, rollout-progress

**Typical child cases:** CanaryDeployment, RollingUpgrade, Rollback, PatchApplication, BreakingChangeMigration

**Context keys:** `currentVersion`, `targetVersion`, `rolloutProgress`, `rollbackCount`, `lastChangeDate` — change history in episodic memory

**Value:** Rollback count feeds risk assessment for future changes. The deployment pipeline's state is part of the service's identity.

### 5.6 SECURITY

**Purpose:** Is the service free of known vulnerabilities and security concerns?

**Status:** CLEAR (OK) · VULNERABILITY_DETECTED (WARNING) · PATCHING (INFO) · BREACH_DETECTED (CRITICAL) · INVESTIGATING (WARNING)

**Typical ganglia:** cve-scanner, anomaly-detector, secret-rotation-due, penetration-test-finding

**Typical child cases:** VulnerabilityPatch, SecretRotation, BreachInvestigation, AnomalyReview, PenTestRemediation

**Context keys:** `openVulnerabilityCount`, `lastScanTime`, `secretRotationDue`, `activeBreachId` — vulnerability details and patch/breach history in external security scanner and episodic memory

**Value:** CVE detection as part of service identity. A CVE that also violates SOC2 triggers both security and compliance dimensions — one detection, two responses.

### 5.7 MAINTENANCE

**Purpose:** Are scheduled operational tasks on track?

**Status:** NO_ACTIVITY (OK) · SCHEDULED (INFO) · IN_PROGRESS (INFO) · OVERDUE (WARNING) · FAILED (CRITICAL)

**Typical ganglia:** maintenance-due, backup-verification, dr-drill-due, certificate-expiry

**Typical child cases:** ScheduledMaintenance, BackupTest, DRDrill, CertificateRenewal

**Context keys:** `nextScheduledWindow`, `lastMaintenanceDate`, `overdueItemCount`, `backupLastVerified`, `drLastTested` — maintenance history in episodic memory

**Value:** Overdue maintenance is a leading indicator of incidents — visible before the incident happens. Most monitoring tools don't track "when was the last backup test?"

### 5.8 PROBLEM_MANAGEMENT

**Purpose:** Are there recurring patterns that need permanent fixes?

**Status:** NO_KNOWN_PROBLEMS (OK) · PATTERN_DETECTED (WARNING) · INVESTIGATING (INFO) · WORKAROUND_APPLIED (WARNING) · ROOT_CAUSE_FIXED (OK)

**Typical ganglia:** incident-pattern (reads health history), scaling-pattern (reads scaling history), drift-pattern (reads drift history)

**Typical child cases:** RootCauseInvestigation, PermanentFix, WorkaroundApplication

**Context keys:** `activeProblemsCount`, `lastPatternDetectedType`, `hasWorkaround` — known problems and correlations queried from cross-dimension episodic memory; pattern detection rules are ganglion configuration, not context data

**Value:** The meta-dimension. Automates ITIL problem management — pattern-detection ganglia read cross-dimension history and fire when thresholds are crossed. WORKAROUND_APPLIED is WARNING (not OK) because a workaround is not a fix.

### 5.9 DECOMMISSION

**Purpose:** Is the service planned for retirement, and is the process on track?

**Status:** NOT_PLANNED (OK) · SCHEDULED (INFO) · IN_PROGRESS (WARNING) · BLOCKED (CRITICAL) · COMPLETED (OK, terminal)

**Typical ganglia:** decommission-schedule, dependency-check, data-migration-progress, traffic-monitor

**Typical child cases:** DataMigration, DependencyCutover, ResourceCleanup, AuditArchival, StranglerFigCutover

**Context keys:** `plannedDate`, `migrationProgress`, `dataArchivalStatus`, `blockingDependencyCount`, `trafficCount` — full dependency graph queried from CMDB on demand

**Value:** BLOCKED means "we planned to decommission but something still depends on this service." The dependency graph prevents the most common decommission failure. COMPLETED triggers service case closure.

## 6. Integration Points

### 6.1 RAS / Ganglion Integration

Ganglia are independent detection units. The service case declares which detections feed which dimensions via two complementary binding layers:

1. **`GanglionBinding`** (ops-api) — maps a RAS detection type to a dimension and context key. Configures the `ServiceDetectionBridge`: which dimension's context section to write to when a detection arrives. This is the ops-layer concern.

2. **Engine `Binding`** (engine-api) — maps a context change to a child case. Configures the case engine: what child case to spawn when a context key changes. This is the engine-layer concern.

**Detection flow:**

```
CloudEvent stream
  → Ganglion (heartbeat, metrics-trend, cve-scanner, etc.)
    → DetectionResult
      → SituationChangeEvent (CDI event)
        → ServiceDetectionBridge [uses GanglionBinding to select dimension + context key]
          → writes to dimension's context section via DimensionSection.put()
            → engine Binding evaluates context change → child case spawned
```

**ServiceDetectionBridge** (`app` module) observes `SituationChangeEvent`, uses `GanglionBinding` to map `(situationType)` → `(dimension, contextKey)`, writes to the target dimension's context section via `DimensionSection.put()`, which delegates to `CaseHubRuntime.signal()`. The engine's `Binding` then evaluates the context change and spawns child cases as configured.

**One detection, multiple dimensions:** A `cve-detected` ganglion fires once. The bridge writes to both `security.vulnerabilityFound` and `compliance.controlViolation`, potentially triggering child cases in both dimensions.

**Dynamic situation registration:** On deploy, register RAS situation definitions for the service's monitoring needs. On decommission, deregister. Uses RAS dynamic registration API (casehub-ras#6).

### 6.2 Desiredstate Integration

The desiredstate `ReconciliationLoop` and the service case are co-managed. The reconciliation loop handles topology (agents, channels, case types). The service case handles operational lifecycle.

Desiredstate drift events feed the CONFIGURATION_DRIFT dimension via the existing `DriftSignalBridge` pattern: `ReconciliationLoop` detects drift → emits `CloudEvent` (type `NODE_DRIFTED`) → `DriftSignalBridge.onCloudEvent(@ObservesAsync CloudEvent)` observes it → signals the engine case context via `CaseHubRuntime.signal()`. This is the same pattern already implemented for application-level drift detection.

### 6.3 Compliance Integration

The existing `CompliancePostureService` feeds the COMPLIANCE dimension. Evidence staleness and control violations are ganglion detections that update the compliance section and trigger EvidenceRecollection or ComplianceRemediation child cases.

### 6.4 Existing App Module

`ApplicationCaseDescriptor` already stubs six child-case bindings (drift, CVE, upgrade, incident, scaling, compliance). These become dimension-tagged bindings in the new `ServiceCaseDescriptor`. `ApplicationLifecycleService` gains service case creation at deploy time.

## 7. Service Case Lifecycle

### 7.1 Creation (Deploy)

```
deploy("order-api", goals)
  1. Start ReconciliationLoop (existing)
  2. CaseHubRuntime.startCase() with ServiceCaseDescriptor
     - 9 dimensions initialized at default status
     - Context sections empty
     - Metadata: cluster, namespace, version
  3. Register ganglion situations for this service (RAS)
  4. Case enters WAITING
```

### 7.2 Steady State (WAITING)

Case sits in WAITING indefinitely. All activity through ganglion detections → context writes → binding evaluation → child case creation. The case itself never transitions to RUNNING.

### 7.3 Signal Sources

| Source | Example | Mechanism |
|---|---|---|
| RAS ganglia | Health check failure | SituationChangeEvent → bridge → context change |
| Desiredstate | Drift detected | CloudEvent (NODE_DRIFTED) → DriftSignalBridge → context change |
| Human operator | Schedule maintenance | CaseHubRuntime.signalCase() via REST |
| CI/CD pipeline | New version available | Webhook → signal → change management |
| External scanner | CVE database update | Webhook → signal → security |

### 7.4 Completion (Decommission)

Case completion criterion: when decommission dimension reaches COMPLETED. The integration path: `DimensionStatusService` writes status changes back to the engine's case context via `CaseHubRuntime.signal(caseId, "dimensions.decommission.status", "COMPLETED")`. The `ServiceCaseDescriptor` declares a `PredicateBasedCompletion` with expression `.dimensions.decommission.status == "COMPLETED"`, which the engine evaluates against its case context. This is a one-way projection — `OperationalDimension` is authoritative, the engine context receives a signal for predicate evaluation. Service case transitions WAITING → COMPLETED (SUCCESS). Post-completion: deregister ganglion situations, stop ReconciliationLoop, retain case and history in ledger for audit.

### 7.5 Suspension and Cancellation

**SUSPENDED:** Service deliberately taken offline (planned maintenance, capacity reduction, etc.).

- **Ganglion suppression:** `ServiceDetectionBridge` checks suspension status before forwarding detections. Events during suspension are discarded, not queued — stale events on resume would trigger false responses.
- **Dimension freeze:** Dimension statuses are preserved at their pre-suspension values. In-progress child cases continue to completion (they represent work already started — killing them mid-flight loses work). No new child cases are created.
- **Resume:** Status returns to WAITING. Ganglia resume firing. Dimensions recompute status from fresh signals.

**CANCELLED:** Service abandoned without proper decommission (team disbanded, project killed, etc.).

- **Approval:** Uses the existing `ApprovalEvaluator` workflow — a CANCELLED transition requires human approval via the same mechanism as plan approvals.
- **Child case cleanup:** Active child cases across all dimensions are cancelled (engine `CaseHubRuntime.cancelCase()`). Each cancellation is recorded in the audit trail.
- **Audit:** The service case transitions to COMPLETED (CANCELLED) with a mandatory `cancellationReason` in the case context. The case and all history are retained in the ledger.

## 8. Child Case Integration

### 8.1 Binding Model

Bindings use the existing `Binding.Builder` API (same pattern as `ApplicationCaseDescriptor`). Dimension tagging is encoded in the binding name prefix — no engine-api changes required.

```java
// In ServiceCaseDescriptor — bindings grouped by dimension
private static List<Binding> healthBindings() {
    return List.of(
            Binding.builder()
                    .name("health:incident-response")
                    .on(new ContextChangeTrigger(".health.serviceDown"))
                    .subCase(SubCase.builder()
                            .namespace("ops").name("incident-response").version("1.0")
                            .inputMapping(".health.incidentData")
                            .waitForCompletion(false)
                            .build())
                    .build()
    );
}
```

`ServiceCaseDescriptor` maintains a static `Map<DimensionType, List<String>> DIMENSION_BINDINGS` mapping each dimension to its binding names (e.g., `HEALTH_MONITORING → ["health:incident-response", "health:auto-restart"]`). This mapping lives in the ops layer — the engine sees flat bindings; the ops layer knows which dimension each binding belongs to.

### 8.2 Child Case Lifecycle

1. Binding fires → engine creates child case
2. `ServiceDetectionBridge` observes the child case creation event (`@ObservesAsync CloudEvent` with case-created type), matches the binding name to a dimension via `DIMENSION_BINDINGS`, adds `CaseRef` to `OperationalDimension.activeResponses`
3. `DimensionStatusService` recomputes composite status
4. Child case runs (workers, signals, human approval)
5. Child case completes → engine emits case-completed `CloudEvent`
6. `ServiceDetectionBridge` observes completion, removes `CaseRef` from `OperationalDimension.activeResponses`
7. Dimension status recomputed — returns to healthy if condition resolved

**Synchronization mechanism:** The bridge observes engine lifecycle events via CDI async events (same pattern as `DriftSignalBridge.onCloudEvent(@ObservesAsync CloudEvent)`). The engine emits `CloudEvent`s for case state changes; the bridge filters for child cases belonging to this service case and updates the corresponding dimension's `activeResponses`.

**Durability:** On every `CaseRef` addition/removal, the bridge writes `dimensions.<type>.activeResponseIds` to the engine context — a bounded list of active child case IDs. This enables restart reconstruction (§4.5) without requiring the bridge to query the engine's internal parent-child relationships.

### 8.3 Concurrency

Multiple child cases per dimension. Status = worst-of-N active responses combined with condition.

## 9. Module Placement

| Type / Service | Module | Package |
|---|---|---|
| DimensionType, DimensionStatus, Severity | ops-api | io.casehub.ops.api.lifecycle |
| OperationalDimension, DimensionSection | ops-api | io.casehub.ops.api.lifecycle |
| ServiceCaseContext, ServiceHealth | ops-api | io.casehub.ops.api.lifecycle |
| GanglionBinding, CaseRef, ManagedServiceCategory | ops-api | io.casehub.ops.api.lifecycle |
| HealthStatus ... DecommissionStatus (9 enums) | ops-api | io.casehub.ops.api.lifecycle.status |
| ServiceCaseDescriptor | app | io.casehub.ops.app.lifecycle |
| ServiceCaseRegistry | app | io.casehub.ops.app.lifecycle |
| DimensionStatusService | app | io.casehub.ops.app.lifecycle |
| ServiceDetectionBridge | app | io.casehub.ops.app.lifecycle |
| Child case descriptors | app | io.casehub.ops.app.lifecycle.cases |

## 10. Scope Boundaries

**In scope for #30 (this design):**
- All types in §4 (domain model)
- Nine dimension status enums (§5)
- ServiceCaseDescriptor with dimension-tagged bindings
- DimensionStatusService (composite status computation)
- ServiceCaseContext initialization on deploy
- Case lifecycle (create, WAITING, completion criterion)

**In scope for #30 but stubbed:**
- Child case descriptors — binding declarations exist, workers are no-ops
- Ganglion bindings — declared in ServiceCaseDescriptor but no RAS integration yet (#47)
- ServiceDetectionBridge — skeleton only, pending casehub-ras#6

**Deferred:**
- #34 — IncidentResponse child case full implementation
- #37 — ComplianceRemediation child case full implementation
- #31 — Partial branch loading / DimensionSection hydration optimization
- #47 — RAS ganglion integration (blocked by casehub-ras#6)
- #38 — UI screens

## 11. Cross-Repo Dependencies

| Dependency | Status | Impact |
|---|---|---|
| casehub-engine CaseStatus.WAITING | Exists | No engine changes needed for basic lifecycle |
| casehub-engine CaseDefinition bindings | Exists | No engine-api change needed — dimension tagging via binding name convention in ops layer |
| casehub-engine BlackboardRegistry | Exists | DimensionSection delegates to engine's case context via CaseHubRuntime.signal()/query() |
| casehub-ras#6 (dynamic situation registration) | CLOSED (design deferred) | Blocks #47, not #30 |
| casehub-desiredstate EventSource | Exists | Drift events feed CONFIGURATION_DRIFT dimension |

## 12. Open Questions

1. ~~**Binding dimension metadata:**~~ **RESOLVED.** Dimension tagging uses binding name prefix convention (e.g., `"health:incident-response"`). `ServiceCaseDescriptor` maintains a `Map<DimensionType, List<String>>` mapping dimensions to binding names. No engine-api change needed — this follows the existing pattern in `ApplicationCaseDescriptor` where bindings are grouped by concern.

2. **DimensionSection persistence:** For #31 (partial loading), sections need independent persistence. Since `DimensionSection` delegates to the engine's case context (via `CaseHubRuntime.signal()/query()`), partial loading depends on the engine's context store supporting selective key-range loading. This is a #31 concern.

3. **ServiceHealth query API:** How does the UI query `ServiceHealth` for all services? REST endpoint in `app/`? Or a dedicated query service? This is a #38 concern but the API shape affects the domain model.