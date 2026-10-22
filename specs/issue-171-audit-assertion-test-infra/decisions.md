## D1: Test tier scope

**Choice:** Both reconciliation and integration-live tiers
**Alternatives:**
- Reconciliation only — faster but leaves integration-live tests without audit verification
- Integration-live only — limited to levels with live tests, misses fast test tier
**Rationale:** Audit assertions should be verifiable at both levels — fast mocked tests for development speed, live tests for end-to-end confidence
**Trade-offs:** Slightly more infrastructure to maintain across two tiers
**Sources:** topology-tests/src/test/java/.../reconciliation/ReconciliationTestBase.java, topology-tests/src/test/java/.../integration/FleetObservabilityIntegrationTest.java
**Exploration:** quick
**Status:** captured

## D2: CDI scope

**Choice:** Pure Java test doubles — no CDI
**Alternatives:**
- CDI test profile — heavier, tests wiring but adds QuarkusTest dependency to topology-tests
- Layered (both) — two implementations of the same API, more maintenance
**Rationale:** ReconciliationTestBase is pure Java with no CDI. Consistency matters — inject test doubles into the executor pipeline the same way FailableNodeProvisioner and MockActualStateAdapter work
**Trade-offs:** Doesn't test CDI wiring of audit services; that's a concern for service/ module tests
**Sources:** ReconciliationTestBase.java (pure Java pattern), FailableNodeProvisioner.java
**Exploration:** quick
**Status:** captured

## D3: Integration-live audit approach

**Choice:** Extend CloudEvent capture pattern from L08 into reusable utility
**Alternatives:**
- Real ledger via ServiceCluster — high fidelity but significant infrastructure overhead
- Skip integration-live audit infra — leaves a gap
**Rationale:** L08's cloudEventSink pattern is already the natural audit seam. CloudEvents are the integration point between desiredstate runtime and all downstream audit consumers (ledger, RAS, case engine)
**Trade-offs:** Doesn't verify ledger persistence in live tests; trusts CloudEvent as the correct boundary
**Sources:** FleetObservabilityIntegrationTest.java (CopyOnWriteArrayList + eventsOfType pattern)
**Exploration:** quick
**Status:** captured

## D4: Module placement

**Choice:** testing/ module (casehub-ops-testing)
**Alternatives:**
- topology-tests/ directly — simpler but not reusable
- New audit-testing/ module — clean but unnecessary module overhead
**Rationale:** testing/ is the designated shared test fixtures module. Audit utilities are shared infrastructure needed by topology-tests and potentially future test modules
**Trade-offs:** testing/ currently has no Java sources (dependency aggregator POM only) — this adds the first real code
**Sources:** testing/pom.xml, CLAUDE.md (testing is test scope only)
**Exploration:** quick
**Status:** captured

## D5: API style

**Choice:** Simple utility class with query methods and static assertion helpers
**Alternatives:**
- Fluent AssertJ custom assertions — more discoverable but project doesn't use custom assertions
- Both (utility + AssertJ) — unnecessary complexity for the same result
**Rationale:** Matches project's existing test style. L08's eventsOfType() pattern is the established idiom. An AuditCapture object with query methods plus static helpers is consistent
**Trade-offs:** Less discoverable than fluent assertions; level authors need to know the utility methods
**Sources:** FleetObservabilityIntegrationTest.java (eventsOfType, countEventsOfType helpers)
**Exploration:** quick
**Status:** captured

## D6: Pipeline integration mechanism

**Choice:** Callback-based AuditCapture injected at construction
**Alternatives:**
- Observer pattern with in-memory event bus — cleaner separation but introduces new abstraction
- Decorator pattern wrapping NodeProvisioner/TransitionExecutor — captures at wrong level vs production
**Rationale:** cloudEventSink callback is already the production audit integration point in ReconciliationLoop. AuditCapture wraps existing callback/sink patterns without new abstractions. FailableNodeProvisioner's step outcome callback is the other natural injection point
**Trade-offs:** Tightly coupled to ReconciliationLoop.builder() API; if that changes, AuditCapture needs updating
**Sources:** ReconciliationLoop.builder().cloudEventSink(), NodeStepExecutor constructor (step outcome callback)
**Exploration:** quick
**Status:** captured
