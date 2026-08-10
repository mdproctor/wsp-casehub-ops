# Operational Efficiency — Lazy Reconstruction Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #31 — Operational efficiency — partial branch loading, eviction, memory management
**Issue group:** #29, #30, #31, #34, #37, #38, #47

**Goal:** Enable lazy reconstruction of service lifecycle cases from engine context after JVM restart, with per-dimension loading granularity and thread-safe concurrent access.

**Architecture:** OperationalDimension gains a loaded/not-loaded state with per-dimension ReadWriteLock. ServiceCaseContext gets a reconstruction factory that creates dimensions with persisted statuses but deferred activeResponses loading. ServiceCaseRegistry gains getOrReconstruct() for event-driven discovery. The persistence foundation (service metadata, composite status, activeResponseIds to engine context) fills gaps from #30's implementation that reconstruction depends on.

**Tech Stack:** Java 21, JUnit 5, ConcurrentHashMap, ReentrantReadWriteLock, CopyOnWriteArrayList

## Global Constraints

- All changes in ops-api (`io.casehub.ops.api.lifecycle`) and ops-app (`io.casehub.ops.app.lifecycle`). No new modules, packages, or types.
- No engine changes — ops-only (D6).
- DimensionSection is a live view over engine context. Section reads/writes are always available. Only `activeResponses` and `status` need reconstruction.
- IntelliJ MCP for all Java edits. `ide_insert_member` for new methods, `ide_replace_member` for modifications, `ide_edit_member` for field/signature changes.
- All tests use HashMap-backed ContextWriter/ContextReader — same pattern as existing tests.
- Build: `mvn --batch-mode -o test -pl api` for api tests, `mvn --batch-mode -o test -pl app` for app tests.

---

### Task 1: OperationalDimension — Thread Safety + Loading State

**Files:**
- Modify: `api/src/main/java/io/casehub/ops/api/lifecycle/OperationalDimension.java`
- Modify: `api/src/test/java/io/casehub/ops/api/lifecycle/OperationalDimensionTest.java`

**Interfaces:**
- Produces: `OperationalDimension.load(DimensionSection.ContextReader reader)`, `isLoaded()`, thread-safe `addResponse`/`removeResponse`/`updateStatus`/`activeResponses`

- [ ] **Step 1: Write failing tests for loading state**

Add to `OperationalDimensionTest.java`:

```java
@Test
void freshDimensionIsLoaded() {
    var dim = createDimension(DimensionType.HEALTH_MONITORING, HealthStatus.HEALTHY);
    assertTrue(dim.isLoaded());
}

@Test
void loadPopulatesActiveResponsesFromReader() {
    var store = new HashMap<String, Object>();
    var section = new DimensionSection(DimensionType.HEALTH_MONITORING, store::put, store::get);
    var dim = new OperationalDimension(DimensionType.HEALTH_MONITORING, HealthStatus.HEALTHY,
            section, List.of(), false);

    var refId = UUID.randomUUID();
    store.put("health.activeResponseIds", List.of(
            Map.<String, Object>of("caseId", refId.toString(),
                    "bindingName", "health:incident-response",
                    "createdAt", "2026-08-01T10:00:00Z")));

    assertFalse(dim.isLoaded());
    dim.load(store::get);
    assertTrue(dim.isLoaded());
    assertEquals(1, dim.activeResponses().size());
    assertEquals(refId, dim.activeResponses().get(0).caseId());
    assertEquals("health:incident-response", dim.activeResponses().get(0).bindingName());
}

@Test
void loadWithNoPersistedDataSetsLoadedWithEmptyResponses() {
    var store = new HashMap<String, Object>();
    var section = new DimensionSection(DimensionType.SECURITY, store::put, store::get);
    var dim = new OperationalDimension(DimensionType.SECURITY, SecurityStatus.CLEAR,
            section, List.of(), false);

    dim.load(store::get);
    assertTrue(dim.isLoaded());
    assertTrue(dim.activeResponses().isEmpty());
}

@Test
void loadIsIdempotent() {
    var store = new HashMap<String, Object>();
    var section = new DimensionSection(DimensionType.HEALTH_MONITORING, store::put, store::get);
    var dim = new OperationalDimension(DimensionType.HEALTH_MONITORING, HealthStatus.HEALTHY,
            section, List.of(), false);

    store.put("health.activeResponseIds", List.of(
            Map.<String, Object>of("caseId", UUID.randomUUID().toString(),
                    "bindingName", "health:incident-response",
                    "createdAt", "2026-08-01T10:00:00Z")));

    dim.load(store::get);
    int countAfterFirstLoad = dim.activeResponses().size();
    dim.load(store::get);
    assertEquals(countAfterFirstLoad, dim.activeResponses().size());
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn --batch-mode -o test -pl api -Dtest=OperationalDimensionTest`
Expected: compilation failure — no `isLoaded()` method, no 5-arg constructor

- [ ] **Step 3: Implement OperationalDimension changes**

Replace the full class body of `OperationalDimension` using `ide_replace_member` for each member:

1. Add imports: `java.util.concurrent.CopyOnWriteArrayList`, `java.util.concurrent.locks.ReentrantReadWriteLock`, `java.util.concurrent.locks.ReadWriteLock`, `java.time.Instant`, `java.util.Map`

2. Change field `activeResponses` from `ArrayList<CaseRef>` to `CopyOnWriteArrayList<CaseRef>`

3. Add new fields:
```java
private volatile boolean loaded;
private final ReadWriteLock lock = new ReentrantReadWriteLock();
```

4. Add second constructor for unloaded dimensions:
```java
public OperationalDimension(DimensionType type, DimensionStatus status,
                            DimensionSection section, List<GanglionBinding> subscriptions,
                            boolean loaded) {
    this.type = type;
    this.status = status;
    this.section = section;
    this.activeResponses = new CopyOnWriteArrayList<>();
    this.subscriptions = List.copyOf(subscriptions);
    this.loaded = loaded;
}
```

5. Modify existing constructor to delegate: call `this(type, status, section, subscriptions, true)`

6. Add `isLoaded()`:
```java
public boolean isLoaded() { return loaded; }
```

7. Add `load(ContextReader)`:
```java
@SuppressWarnings("unchecked")
public void load(DimensionSection.ContextReader reader) {
    lock.writeLock().lock();
    try {
        if (loaded) return;
        Object raw = reader.read(type.contextPrefix() + "activeResponseIds");
        if (raw instanceof List<?> list) {
            for (Object entry : list) {
                if (entry instanceof Map<?, ?> map) {
                    activeResponses.add(new CaseRef(
                            UUID.fromString((String) map.get("caseId")),
                            (String) map.get("bindingName"),
                            Instant.parse((String) map.get("createdAt"))));
                }
            }
        }
        loaded = true;
    } finally {
        lock.writeLock().unlock();
    }
}
```

8. Wrap `updateStatus` with write lock:
```java
public void updateStatus(DimensionStatus newStatus) {
    lock.writeLock().lock();
    try {
        this.status = newStatus;
    } finally {
        lock.writeLock().unlock();
    }
}
```

9. Wrap `addResponse` with write lock:
```java
public void addResponse(CaseRef ref) {
    lock.writeLock().lock();
    try {
        activeResponses.add(ref);
    } finally {
        lock.writeLock().unlock();
    }
}
```

10. Wrap `removeResponse` with write lock:
```java
public void removeResponse(UUID caseId) {
    lock.writeLock().lock();
    try {
        activeResponses.removeIf(r -> r.caseId().equals(caseId));
    } finally {
        lock.writeLock().unlock();
    }
}
```

11. Wrap `activeResponses()` with read lock:
```java
public List<CaseRef> activeResponses() {
    lock.readLock().lock();
    try {
        return Collections.unmodifiableList(activeResponses);
    } finally {
        lock.readLock().unlock();
    }
}
```

- [ ] **Step 4: Run all api tests to verify they pass**

Run: `mvn --batch-mode -o test -pl api`
Expected: all tests pass (existing + new)

- [ ] **Step 5: Run app tests to check for breakage**

Run: `mvn --batch-mode -o test -pl app`
Expected: all tests pass — the new constructor is additive, existing 4-arg constructor still works

- [ ] **Step 6: Commit**

```bash
git add api/src/main/java/io/casehub/ops/api/lifecycle/OperationalDimension.java api/src/test/java/io/casehub/ops/api/lifecycle/OperationalDimensionTest.java
git commit -m "feat(#31): OperationalDimension — thread safety + loading state

ArrayList → CopyOnWriteArrayList, per-dimension ReadWriteLock,
load(ContextReader) for lazy reconstruction from engine context.

Refs #31"
```

---

### Task 2: DimensionType — resolveStatus() + defaultStatus()

**Files:**
- Modify: `api/src/main/java/io/casehub/ops/api/lifecycle/DimensionType.java`
- Modify: `api/src/test/java/io/casehub/ops/api/lifecycle/DimensionTypeTest.java`

**Interfaces:**
- Produces: `DimensionType.resolveStatus(String name)` returns `DimensionStatus`, `DimensionType.defaultStatus()` returns `DimensionStatus`

- [ ] **Step 1: Write failing tests**

Add to `DimensionTypeTest.java`:

```java
@Test
void resolveStatusForEachDimension() {
    assertEquals(HealthStatus.DOWN, DimensionType.HEALTH_MONITORING.resolveStatus("DOWN"));
    assertEquals(ConfigurationDriftStatus.DRIFTED, DimensionType.CONFIGURATION_DRIFT.resolveStatus("DRIFTED"));
    assertEquals(ComplianceStatus.NON_COMPLIANT, DimensionType.COMPLIANCE.resolveStatus("NON_COMPLIANT"));
    assertEquals(ScalingStatus.SCALING, DimensionType.SCALING.resolveStatus("SCALING"));
    assertEquals(ChangeManagementStatus.ROLLBACK, DimensionType.CHANGE_MANAGEMENT.resolveStatus("ROLLBACK"));
    assertEquals(SecurityStatus.BREACH_DETECTED, DimensionType.SECURITY.resolveStatus("BREACH_DETECTED"));
    assertEquals(MaintenanceStatus.OVERDUE, DimensionType.MAINTENANCE.resolveStatus("OVERDUE"));
    assertEquals(ProblemManagementStatus.PATTERN_DETECTED, DimensionType.PROBLEM_MANAGEMENT.resolveStatus("PATTERN_DETECTED"));
    assertEquals(DecommissionStatus.COMPLETED, DimensionType.DECOMMISSION.resolveStatus("COMPLETED"));
}

@Test
void resolveStatusThrowsForInvalidName() {
    assertThrows(IllegalArgumentException.class,
            () -> DimensionType.HEALTH_MONITORING.resolveStatus("NONEXISTENT"));
}

@Test
void defaultStatusForEachDimension() {
    assertEquals(HealthStatus.HEALTHY, DimensionType.HEALTH_MONITORING.defaultStatus());
    assertEquals(ConfigurationDriftStatus.IN_SYNC, DimensionType.CONFIGURATION_DRIFT.defaultStatus());
    assertEquals(ComplianceStatus.COMPLIANT, DimensionType.COMPLIANCE.defaultStatus());
    assertEquals(ScalingStatus.OPTIMAL, DimensionType.SCALING.defaultStatus());
    assertEquals(ChangeManagementStatus.NO_ACTIVITY, DimensionType.CHANGE_MANAGEMENT.defaultStatus());
    assertEquals(SecurityStatus.CLEAR, DimensionType.SECURITY.defaultStatus());
    assertEquals(MaintenanceStatus.NO_ACTIVITY, DimensionType.MAINTENANCE.defaultStatus());
    assertEquals(ProblemManagementStatus.NO_KNOWN_PROBLEMS, DimensionType.PROBLEM_MANAGEMENT.defaultStatus());
    assertEquals(DecommissionStatus.NOT_PLANNED, DimensionType.DECOMMISSION.defaultStatus());
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn --batch-mode -o test -pl api -Dtest=DimensionTypeTest`
Expected: compilation failure — no `resolveStatus` or `defaultStatus` methods

- [ ] **Step 3: Implement DimensionType changes**

Replace `DimensionType` enum body. Each constant gains `statusClass` (the enum class implementing `DimensionStatus`) and `defaultStatus` (the healthy/nominal value):

```java
HEALTH_MONITORING("health.", HealthStatus.class, HealthStatus.HEALTHY),
CONFIGURATION_DRIFT("drift.", ConfigurationDriftStatus.class, ConfigurationDriftStatus.IN_SYNC),
COMPLIANCE("compliance.", ComplianceStatus.class, ComplianceStatus.COMPLIANT),
SCALING("scaling.", ScalingStatus.class, ScalingStatus.OPTIMAL),
CHANGE_MANAGEMENT("change.", ChangeManagementStatus.class, ChangeManagementStatus.NO_ACTIVITY),
SECURITY("security.", SecurityStatus.class, SecurityStatus.CLEAR),
MAINTENANCE("maintenance.", MaintenanceStatus.class, MaintenanceStatus.NO_ACTIVITY),
PROBLEM_MANAGEMENT("problems.", ProblemManagementStatus.class, ProblemManagementStatus.NO_KNOWN_PROBLEMS),
DECOMMISSION("decommission.", DecommissionStatus.class, DecommissionStatus.NOT_PLANNED);
```

Add fields and methods:
```java
private final String contextPrefix;
private final Class<? extends Enum<?>> statusClass;
private final DimensionStatus defaultStatus;

DimensionType(String contextPrefix, Class<? extends Enum<?>> statusClass, DimensionStatus defaultStatus) {
    this.contextPrefix = contextPrefix;
    this.statusClass = statusClass;
    this.defaultStatus = defaultStatus;
}

public String contextPrefix() { return contextPrefix; }

public DimensionStatus defaultStatus() { return defaultStatus; }

@SuppressWarnings({"unchecked", "rawtypes"})
public DimensionStatus resolveStatus(String name) {
    return (DimensionStatus) Enum.valueOf((Class<Enum>) statusClass, name);
}
```

- [ ] **Step 4: Run api tests to verify they pass**

Run: `mvn --batch-mode -o test -pl api`
Expected: all tests pass

- [ ] **Step 5: Commit**

```bash
git add api/src/main/java/io/casehub/ops/api/lifecycle/DimensionType.java api/src/test/java/io/casehub/ops/api/lifecycle/DimensionTypeTest.java
git commit -m "feat(#31): DimensionType.resolveStatus() + defaultStatus()

Each DimensionType constant now carries its status enum class and
default healthy status. Eliminates switch statements for status
resolution — the type owns its vocabulary.

Refs #31"
```

---

### Task 3: ServiceCaseContext — Reconstruction Factory

**Files:**
- Modify: `api/src/main/java/io/casehub/ops/api/lifecycle/ServiceCaseContext.java`
- Modify: `api/src/test/java/io/casehub/ops/api/lifecycle/ServiceCaseContextTest.java`

**Interfaces:**
- Consumes: `DimensionType.defaultStatus()`, `OperationalDimension(type, status, section, subscriptions, loaded)`
- Produces: `ServiceCaseContext.createForReconstruction(serviceId, serviceName, category, deployedAt, metadata, persistedStatuses, writer, reader)`

- [ ] **Step 1: Write failing tests**

Add to `ServiceCaseContextTest.java`:

```java
@Test
void createForReconstructionUsesPersistedStatuses() {
    var store = new HashMap<String, Object>();
    var statuses = new EnumMap<DimensionType, DimensionStatus>(DimensionType.class);
    statuses.put(DimensionType.HEALTH_MONITORING, HealthStatus.DOWN);
    statuses.put(DimensionType.SECURITY, SecurityStatus.VULNERABILITY_DETECTED);
    // remaining dimensions use defaults (not in map)

    var ctx = ServiceCaseContext.createForReconstruction(
            "order-api", "Order API", ManagedServiceCategory.APPLICATION,
            Instant.parse("2026-08-01T10:00:00Z"), Map.of(),
            statuses, store::put, store::get);

    assertEquals(HealthStatus.DOWN, ctx.dimensions().get(DimensionType.HEALTH_MONITORING).status());
    assertEquals(SecurityStatus.VULNERABILITY_DETECTED, ctx.dimensions().get(DimensionType.SECURITY).status());
    assertEquals(ComplianceStatus.COMPLIANT, ctx.dimensions().get(DimensionType.COMPLIANCE).status());
}

@Test
void createForReconstructionDimensionsAreNotLoaded() {
    var store = new HashMap<String, Object>();
    var ctx = ServiceCaseContext.createForReconstruction(
            "order-api", "Order API", ManagedServiceCategory.APPLICATION,
            Instant.parse("2026-08-01T10:00:00Z"), Map.of(),
            Map.of(), store::put, store::get);

    for (var dim : ctx.dimensions().values()) {
        assertFalse(dim.isLoaded());
    }
}

@Test
void createForReconstructionPreservesMetadata() {
    var store = new HashMap<String, Object>();
    var ctx = ServiceCaseContext.createForReconstruction(
            "order-api", "Order API", ManagedServiceCategory.APPLICATION,
            Instant.parse("2026-08-01T10:00:00Z"), Map.of("cluster", "prod"),
            Map.of(), store::put, store::get);

    assertEquals("order-api", ctx.serviceId());
    assertEquals("Order API", ctx.serviceName());
    assertEquals(ManagedServiceCategory.APPLICATION, ctx.category());
    assertEquals(Instant.parse("2026-08-01T10:00:00Z"), ctx.deployedAt());
    assertEquals("prod", ctx.metadata().get("cluster"));
}

@Test
void createForReconstructionToServiceHealthUsesPersistedStatuses() {
    var store = new HashMap<String, Object>();
    var statuses = new EnumMap<DimensionType, DimensionStatus>(DimensionType.class);
    statuses.put(DimensionType.HEALTH_MONITORING, HealthStatus.DOWN);

    var ctx = ServiceCaseContext.createForReconstruction(
            "order-api", "Order API", ManagedServiceCategory.APPLICATION,
            Instant.parse("2026-08-01T10:00:00Z"), Map.of(),
            statuses, store::put, store::get);

    var health = ctx.toServiceHealth();
    assertEquals(Severity.CRITICAL, health.overallSeverity());
    assertEquals(HealthStatus.DOWN, health.dimensions().get(DimensionType.HEALTH_MONITORING));
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn --batch-mode -o test -pl api -Dtest=ServiceCaseContextTest`
Expected: compilation failure — no `createForReconstruction` method

- [ ] **Step 3: Implement createForReconstruction**

Add to `ServiceCaseContext.java`:

```java
public static ServiceCaseContext createForReconstruction(
        String serviceId, String serviceName,
        ManagedServiceCategory category, Instant deployedAt,
        Map<String, Object> metadata,
        Map<DimensionType, DimensionStatus> persistedStatuses,
        DimensionSection.ContextWriter writer,
        DimensionSection.ContextReader reader) {
    var dims = new EnumMap<DimensionType, OperationalDimension>(DimensionType.class);
    for (DimensionType type : DimensionType.values()) {
        var section = new DimensionSection(type, writer, reader);
        DimensionStatus status = persistedStatuses.getOrDefault(type, type.defaultStatus());
        dims.put(type, new OperationalDimension(type, status, section, List.of(), false));
    }
    return new ServiceCaseContext(serviceId, serviceName, category, dims, deployedAt, metadata);
}
```

Also update `create()` to use `DimensionType.defaultStatus()`:

Replace `dims.put(type, new OperationalDimension(type, defaultStatus(type), section, List.of()));`
with `dims.put(type, new OperationalDimension(type, type.defaultStatus(), section, List.of()));`

Remove the private `defaultStatus(DimensionType)` method and its 9 status imports (they're now in DimensionType).

- [ ] **Step 4: Run all api tests to verify they pass**

Run: `mvn --batch-mode -o test -pl api`
Expected: all tests pass

- [ ] **Step 5: Run app tests to check for breakage**

Run: `mvn --batch-mode -o test -pl app`
Expected: all tests pass — `create()` behavior unchanged

- [ ] **Step 6: Commit**

```bash
git add api/src/main/java/io/casehub/ops/api/lifecycle/ServiceCaseContext.java api/src/test/java/io/casehub/ops/api/lifecycle/ServiceCaseContextTest.java
git commit -m "feat(#31): ServiceCaseContext.createForReconstruction()

Factory for creating unloaded contexts with persisted statuses after
restart. Dimensions start not-loaded — activeResponses populated lazily
via OperationalDimension.load(). Migrate create() to use
DimensionType.defaultStatus().

Refs #31"
```

---

### Task 4: DimensionStatusService — Status Persistence + Migrate resolveStatus

**Files:**
- Modify: `app/src/main/java/io/casehub/ops/app/lifecycle/DimensionStatusService.java`
- Modify: `app/src/test/java/io/casehub/ops/app/lifecycle/DimensionStatusServiceTest.java`

**Interfaces:**
- Consumes: `DimensionType.resolveStatus(String)`, `OperationalDimension.section().put()`
- Produces: `recompute()` now persists composite status to `<prefix>status` in engine context

- [ ] **Step 1: Write failing test for status persistence**

Add to `DimensionStatusServiceTest.java`:

```java
@Test
void recomputePersistsStatusToSection() {
    var store = new HashMap<String, Object>();
    var dim = createDimension(DimensionType.HEALTH_MONITORING, HealthStatus.HEALTHY, store);

    dim.section().put("condition", "DOWN");
    service.recompute(dim);

    assertEquals("DOWN", store.get("health.status"));
}

@Test
void recomputeWithResponsePersistsResponseStatus() {
    var store = new HashMap<String, Object>();
    var dim = createDimension(DimensionType.HEALTH_MONITORING, HealthStatus.HEALTHY, store);

    dim.section().put("condition", "DOWN");
    dim.addResponse(new CaseRef(UUID.randomUUID(), "health:incident-response", Instant.now()));
    service.recompute(dim);

    assertEquals("REMEDIATING", store.get("health.status"));
}
```

Update the existing `createDimension` helper to accept a store (or add an overload):

```java
private OperationalDimension createDimension(DimensionType type, DimensionStatus initialStatus, Map<String, Object> store) {
    var section = new DimensionSection(type, store::put, store::get);
    return new OperationalDimension(type, initialStatus, section, List.of());
}
```

Migrate the existing `createDimension(DimensionType, DimensionStatus)` to use the new overload:
```java
private OperationalDimension createDimension(DimensionType type, DimensionStatus initialStatus) {
    return createDimension(type, initialStatus, new HashMap<>());
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn --batch-mode -o test -pl app -Dtest=DimensionStatusServiceTest`
Expected: FAIL — `store.get("health.status")` is null (recompute doesn't write to section yet)

- [ ] **Step 3: Implement status persistence in recompute()**

Modify `DimensionStatusService.recompute()`:

```java
public void recompute(OperationalDimension dimension) {
    DimensionStatus condition = readCondition(dimension);
    boolean hasActiveResponses = !dimension.activeResponses().isEmpty();
    boolean conditionHealthy = condition.severity() == Severity.OK;

    DimensionStatus newStatus;
    if (conditionHealthy || !hasActiveResponses) {
        newStatus = condition;
    } else {
        newStatus = responseStatus(dimension.type());
    }
    dimension.updateStatus(newStatus);
    dimension.section().put("status", ((Enum<?>) newStatus).name());
}
```

Also replace the private `resolveStatus` method body with:
```java
private DimensionStatus resolveStatus(DimensionType type, String name) {
    return type.resolveStatus(name);
}
```

- [ ] **Step 4: Run all app tests to verify they pass**

Run: `mvn --batch-mode -o test -pl app`
Expected: all tests pass

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/io/casehub/ops/app/lifecycle/DimensionStatusService.java app/src/test/java/io/casehub/ops/app/lifecycle/DimensionStatusServiceTest.java
git commit -m "feat(#31): DimensionStatusService persists composite status

recompute() writes status to section after every computation — enables
reconstruction of correct status after restart. Migrate resolveStatus
to DimensionType.resolveStatus().

Refs #31"
```

---

### Task 5: ServiceCaseRegistry — getOrReconstruct + Metadata Persistence + serviceId Index

**Files:**
- Modify: `app/src/main/java/io/casehub/ops/app/lifecycle/ServiceCaseRegistry.java`
- Modify: `app/src/test/java/io/casehub/ops/app/lifecycle/ServiceCaseRegistryTest.java`

**Interfaces:**
- Consumes: `ServiceCaseContext.createForReconstruction(...)`, `DimensionType.resolveStatus(String)`, `DimensionType.defaultStatus()`
- Produces: `getOrReconstruct(UUID, ContextWriter, ContextReader)`, `getByServiceId(String)`, `register()` writes metadata to engine context

- [ ] **Step 1: Write failing tests**

Add to `ServiceCaseRegistryTest.java`:

```java
@Test
void registerWritesMetadataToEngineContext() {
    var store = new HashMap<String, Object>();
    UUID caseId = UUID.randomUUID();

    registry.register(caseId, "order-api", "Order API",
            ManagedServiceCategory.APPLICATION, Map.of("cluster", "prod"),
            store::put, store::get);

    assertEquals("order-api", store.get("service.serviceId"));
    assertEquals("Order API", store.get("service.serviceName"));
    assertEquals("APPLICATION", store.get("service.category"));
    assertNotNull(store.get("service.deployedAt"));
}

@Test
void getByServiceIdReturnsContext() {
    var store = new HashMap<String, Object>();
    UUID caseId = UUID.randomUUID();

    registry.register(caseId, "order-api", "Order API",
            ManagedServiceCategory.APPLICATION, Map.of(),
            store::put, store::get);

    assertNotNull(registry.getByServiceId("order-api"));
    assertEquals("order-api", registry.getByServiceId("order-api").serviceId());
}

@Test
void getByServiceIdReturnsNullForUnknown() {
    assertNull(registry.getByServiceId("unknown"));
}

@Test
void deregisterRemovesFromServiceIndex() {
    var store = new HashMap<String, Object>();
    UUID caseId = UUID.randomUUID();

    registry.register(caseId, "order-api", "Order API",
            ManagedServiceCategory.APPLICATION, Map.of(),
            store::put, store::get);
    registry.deregister(caseId);

    assertNull(registry.getByServiceId("order-api"));
}

@Test
void getOrReconstructCreatesUnloadedContext() {
    var store = new HashMap<String, Object>();
    UUID caseId = UUID.randomUUID();

    store.put("service.serviceId", "order-api");
    store.put("service.serviceName", "Order API");
    store.put("service.category", "APPLICATION");
    store.put("service.deployedAt", "2026-08-01T10:00:00Z");
    store.put("health.status", "DOWN");

    var ctx = registry.getOrReconstruct(caseId, store::put, store::get);

    assertNotNull(ctx);
    assertEquals("order-api", ctx.serviceId());
    assertEquals("Order API", ctx.serviceName());
    assertEquals(ManagedServiceCategory.APPLICATION, ctx.category());
    assertEquals(HealthStatus.DOWN, ctx.dimensions().get(DimensionType.HEALTH_MONITORING).status());
    assertFalse(ctx.dimensions().get(DimensionType.HEALTH_MONITORING).isLoaded());
}

@Test
void getOrReconstructReturnsExistingContext() {
    var store = new HashMap<String, Object>();
    UUID caseId = UUID.randomUUID();

    registry.register(caseId, "order-api", "Order API",
            ManagedServiceCategory.APPLICATION, Map.of(),
            store::put, store::get);

    var ctx = registry.getOrReconstruct(caseId, store::put, store::get);
    assertTrue(ctx.dimensions().get(DimensionType.HEALTH_MONITORING).isLoaded());
}

@Test
void getOrReconstructReturnsNullForNonServiceCase() {
    var store = new HashMap<String, Object>();
    var ctx = registry.getOrReconstruct(UUID.randomUUID(), store::put, store::get);
    assertNull(ctx);
}

@Test
void getOrReconstructPopulatesServiceIndex() {
    var store = new HashMap<String, Object>();
    UUID caseId = UUID.randomUUID();

    store.put("service.serviceId", "order-api");
    store.put("service.serviceName", "Order API");
    store.put("service.category", "APPLICATION");
    store.put("service.deployedAt", "2026-08-01T10:00:00Z");

    registry.getOrReconstruct(caseId, store::put, store::get);
    assertNotNull(registry.getByServiceId("order-api"));
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn --batch-mode -o test -pl app -Dtest=ServiceCaseRegistryTest`
Expected: compilation failure — no `getOrReconstruct`, `getByServiceId` methods

- [ ] **Step 3: Implement ServiceCaseRegistry changes**

Replace `ServiceCaseRegistry` class body:

```java
@ApplicationScoped
public class ServiceCaseRegistry {

    private final ConcurrentHashMap<UUID, ServiceCaseContext> contexts = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<String, UUID> serviceIndex = new ConcurrentHashMap<>();

    public void register(UUID caseId, String serviceId, String serviceName,
                         ManagedServiceCategory category, Map<String, Object> metadata,
                         DimensionSection.ContextWriter writer, DimensionSection.ContextReader reader) {
        writer.write("service.serviceId", serviceId);
        writer.write("service.serviceName", serviceName);
        writer.write("service.category", category.name());
        writer.write("service.deployedAt", Instant.now().toString());

        var ctx = ServiceCaseContext.create(serviceId, serviceName, category, metadata, writer, reader);
        contexts.put(caseId, ctx);
        serviceIndex.put(serviceId, caseId);
    }

    public ServiceCaseContext get(UUID caseId) {
        return contexts.get(caseId);
    }

    public ServiceCaseContext getByServiceId(String serviceId) {
        UUID caseId = serviceIndex.get(serviceId);
        return caseId != null ? contexts.get(caseId) : null;
    }

    public ServiceCaseContext getOrReconstruct(UUID caseId,
                                               DimensionSection.ContextWriter writer,
                                               DimensionSection.ContextReader reader) {
        ServiceCaseContext existing = contexts.get(caseId);
        if (existing != null) return existing;

        String serviceId = (String) reader.read("service.serviceId");
        if (serviceId == null) return null;

        String serviceName = (String) reader.read("service.serviceName");
        String categoryName = (String) reader.read("service.category");
        String deployedAtStr = (String) reader.read("service.deployedAt");

        ManagedServiceCategory category = ManagedServiceCategory.valueOf(categoryName);
        Instant deployedAt = Instant.parse(deployedAtStr);

        var statuses = new EnumMap<DimensionType, DimensionStatus>(DimensionType.class);
        for (DimensionType type : DimensionType.values()) {
            String statusName = (String) reader.read(type.contextPrefix() + "status");
            if (statusName != null) {
                statuses.put(type, type.resolveStatus(statusName));
            }
        }

        var ctx = ServiceCaseContext.createForReconstruction(
                serviceId, serviceName, category, deployedAt, Map.of(),
                statuses, writer, reader);

        ServiceCaseContext winner = contexts.putIfAbsent(caseId, ctx);
        if (winner != null) return winner;

        serviceIndex.put(serviceId, caseId);
        return ctx;
    }

    public void deregister(UUID caseId) {
        ServiceCaseContext ctx = contexts.remove(caseId);
        if (ctx != null) {
            serviceIndex.remove(ctx.serviceId());
        }
    }
}
```

Add imports: `java.time.Instant`, `java.util.EnumMap`, `io.casehub.ops.api.lifecycle.DimensionStatus`, `io.casehub.ops.api.lifecycle.DimensionType`

- [ ] **Step 4: Run all app tests to verify they pass**

Run: `mvn --batch-mode -o test -pl app`
Expected: all tests pass (existing registry tests + new tests + bridge/status tests unaffected)

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/io/casehub/ops/app/lifecycle/ServiceCaseRegistry.java app/src/test/java/io/casehub/ops/app/lifecycle/ServiceCaseRegistryTest.java
git commit -m "feat(#31): ServiceCaseRegistry.getOrReconstruct() + metadata persistence

register() writes service identity to engine context. getOrReconstruct()
creates unloaded contexts from persisted state on first access after
restart. serviceId index enables service-oriented lookups.

Refs #31"
```

---

### Task 6: ServiceDetectionBridge — Load-on-Demand + activeResponseIds Persistence

**Files:**
- Modify: `app/src/main/java/io/casehub/ops/app/lifecycle/ServiceDetectionBridge.java`
- Modify: `app/src/test/java/io/casehub/ops/app/lifecycle/ServiceDetectionBridgeTest.java`

**Interfaces:**
- Consumes: `OperationalDimension.isLoaded()`, `OperationalDimension.load(ContextReader)`, `ServiceCaseRegistry.getOrReconstruct(UUID, ContextWriter, ContextReader)`
- Produces: Detection on non-loaded dimension triggers load. activeResponseIds written to section on child case lifecycle events.

- [ ] **Step 1: Write failing tests for load-on-demand**

Add to `ServiceDetectionBridgeTest.java`:

```java
@Test
void detectionOnNonLoadedDimensionLoadsItFirst() {
    var store = new HashMap<String, Object>();
    UUID caseId = UUID.randomUUID();

    store.put("service.serviceId", "order-api");
    store.put("service.serviceName", "Order API");
    store.put("service.category", "APPLICATION");
    store.put("service.deployedAt", "2026-08-01T10:00:00Z");
    store.put("health.activeResponseIds", List.of(
            Map.<String, Object>of("caseId", UUID.randomUUID().toString(),
                    "bindingName", "health:incident-response",
                    "createdAt", "2026-08-01T10:00:00Z")));

    bridge.registerBindings(caseId, List.of(
            new GanglionBinding("heartbeat-failure", DimensionType.HEALTH_MONITORING,
                    "serviceDown", HealthStatus.DOWN)));

    bridge.onDetection("heartbeat-failure", caseId, store::put, store::get, Map.of());

    var ctx = registry.getOrReconstruct(caseId, store::put, store::get);
    var dim = ctx.dimensions().get(DimensionType.HEALTH_MONITORING);
    assertTrue(dim.isLoaded());
    assertEquals(1, dim.activeResponses().size());
    assertEquals(HealthStatus.DOWN, dim.status());
}
```

- [ ] **Step 2: Write failing test for activeResponseIds persistence**

```java
@Test
void activeResponseIdPersistedOnAdd() {
    UUID caseId = registerService();
    bridge.registerBindings(caseId, List.of(
            new GanglionBinding("heartbeat-failure", DimensionType.HEALTH_MONITORING,
                    "serviceDown", HealthStatus.DOWN)));

    bridge.onDetection("heartbeat-failure", caseId, Map.of());

    var ctx = registry.get(caseId);
    var dim = ctx.dimensions().get(DimensionType.HEALTH_MONITORING);
    var ref = new CaseRef(UUID.randomUUID(), "health:incident-response", Instant.now());
    bridge.addResponseAndPersist(caseId, DimensionType.HEALTH_MONITORING, ref);

    assertEquals(1, dim.activeResponses().size());
    assertNotNull(store.get("health.activeResponseIds"));
}

@Test
void activeResponseIdPersistedOnRemove() {
    UUID caseId = registerService();
    var ref = new CaseRef(UUID.randomUUID(), "health:incident-response", Instant.now());
    var ctx = registry.get(caseId);
    var dim = ctx.dimensions().get(DimensionType.HEALTH_MONITORING);
    bridge.addResponseAndPersist(caseId, DimensionType.HEALTH_MONITORING, ref);

    bridge.removeResponseAndPersist(caseId, DimensionType.HEALTH_MONITORING, ref.caseId());

    assertTrue(dim.activeResponses().isEmpty());
    var persisted = store.get("health.activeResponseIds");
    assertNotNull(persisted);
    assertTrue(((List<?>) persisted).isEmpty());
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn --batch-mode -o test -pl app -Dtest=ServiceDetectionBridgeTest`
Expected: compilation failure — no `onDetection` overload with writer/reader, no `addResponseAndPersist`/`removeResponseAndPersist`

- [ ] **Step 4: Implement ServiceDetectionBridge changes**

Add new `onDetection` overload that accepts writer/reader for getOrReconstruct:

```java
public void onDetection(String situationType, UUID caseId,
                        DimensionSection.ContextWriter writer,
                        DimensionSection.ContextReader reader,
                        Map<String, Object> detectionData) {
    var ctx = registry.getOrReconstruct(caseId, writer, reader);
    if (ctx == null) return;

    var bindings = bindingsMap.getOrDefault(caseId, List.of());
    for (var binding : bindings) {
        if (binding.situationType().equals(situationType)) {
            var dimension = ctx.dimensions().get(binding.dimension());

            if (!dimension.isLoaded()) {
                dimension.load(reader);
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

Keep the existing `onDetection(String, UUID, Map)` delegating to `registry.get()` (backward compatible for callers that already registered the service).

Add activeResponseIds persistence methods:

```java
public void addResponseAndPersist(UUID caseId, DimensionType dimType, CaseRef ref) {
    var ctx = registry.get(caseId);
    if (ctx == null) return;
    var dimension = ctx.dimensions().get(dimType);
    dimension.addResponse(ref);
    persistActiveResponseIds(dimension);
    statusService.recompute(dimension);
}

public void removeResponseAndPersist(UUID caseId, DimensionType dimType, UUID childCaseId) {
    var ctx = registry.get(caseId);
    if (ctx == null) return;
    var dimension = ctx.dimensions().get(dimType);
    dimension.removeResponse(childCaseId);
    persistActiveResponseIds(dimension);
    statusService.recompute(dimension);
}

private void persistActiveResponseIds(OperationalDimension dimension) {
    List<Map<String, Object>> serialized = dimension.activeResponses().stream()
            .map(r -> Map.<String, Object>of(
                    "caseId", r.caseId().toString(),
                    "bindingName", r.bindingName(),
                    "createdAt", r.createdAt().toString()))
            .toList();
    dimension.section().put("activeResponseIds", serialized);
}
```

- [ ] **Step 5: Run all app tests to verify they pass**

Run: `mvn --batch-mode -o test -pl app`
Expected: all tests pass

- [ ] **Step 6: Run full build**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS

- [ ] **Step 7: Commit**

```bash
git add app/src/main/java/io/casehub/ops/app/lifecycle/ServiceDetectionBridge.java app/src/test/java/io/casehub/ops/app/lifecycle/ServiceDetectionBridgeTest.java
git commit -m "feat(#31): ServiceDetectionBridge — load-on-demand + activeResponseIds persistence

Detection on non-loaded dimension triggers lazy reconstruction before
recompute. addResponseAndPersist/removeResponseAndPersist write
activeResponseIds to engine context for reconstruction after restart.

Refs #31"
```
