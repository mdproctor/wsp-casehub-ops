## D1: Test tier scope

**Choice:** Reconciliation, integration-live, and live (K3s) tiers — compilation tier excluded
**Alternatives:**
- Reconciliation + integration-live only — original scope, misses live K8s tier which uses the same SimpleTransitionExecutor pipeline
- All four tiers — compilation tier produces no provisioning, so has no audit events to capture
- Reconciliation only — faster but leaves integration-live and live tests without audit verification
- Integration-live only — limited to levels with live tests, misses fast test tier
**Rationale:** Audit assertions should be verifiable at every tier that executes provisioning. Compilation tests (TopologyTestBase) produce DesiredStateGraph only — no transitions, no outcomes, no audit events. The remaining three tiers all execute transitions: reconciliation and live via SimpleTransitionExecutor directly, integration-live via ReconciliationLoop. The TransitionExecutor decorator mechanism (see D6) applies equally to reconciliation and live tiers since both construct SimpleTransitionExecutor the same way.
**Trade-offs:** Three tiers increases infrastructure surface, but the reconciliation and live tiers share the same injection mechanism (TransitionExecutor decorator), so the incremental cost of adding live is minimal
**Sources:** ReconciliationTestBase.java (SimpleTransitionExecutor), K3sTestBase.java (same pattern), FleetObservabilityIntegrationTest.java (ReconciliationLoop + cloudEventSink), compilation tests (no provisioning)
**Exploration:** quick
**Status:** revised — expanded from 2 tiers to 3, explicitly excluded compilation with rationale, added live K3s tier since it shares the same SimpleTransitionExecutor injection point

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
**Rationale:** testing/ is the established shared test fixtures module with 6 existing stub implementations (StubChannelStore, StubEndpointRegistry, StubDeploymentTarget, StubAgentRegistry, StubEvidenceCollector, StubChannelBindingStore). Audit test utilities are the same category — shared test doubles consumed by topology-tests and potentially future test modules
**Trade-offs:** Adds to the existing test fixtures module, increasing its surface area
**Sources:** testing/src/main/java/io/casehub/ops/testing/ (6 existing stubs), testing/pom.xml, ARC42STORIES.MD §5 (testing/ is test scope only)
**Exploration:** quick
**Status:** revised — corrected factual error: testing/ already has 6 Java source files, not "no Java sources"

## D5: API style

**Choice:** Simple utility class with query methods and static assertion helpers
**Alternatives:**
- Fluent AssertJ custom assertions — more discoverable via IDE autocomplete, compile-time type safety, but project has no existing custom assertion infrastructure
- Both (utility + AssertJ) — unnecessary complexity for the same result
**Rationale:** The project is small enough that utility methods suffice. FleetObservabilityIntegrationTest has private eventsOfType()/countEventsOfType() helpers — extracting these into shared infrastructure is the natural evolution. AssertJ custom assertions are worth introducing later if the assertion API grows significantly across issues #172 and #173, but premature for the initial infrastructure
**Trade-offs:** Less discoverable than fluent assertions; level authors need to know the utility methods. Reassess after levels 01-08 implementation reveals whether utility methods scale
**Sources:** FleetObservabilityIntegrationTest.java (eventsOfType, countEventsOfType — private helpers in one file, not a project-wide idiom)
**Exploration:** quick
**Status:** revised — corrected rationale: one local helper in one test file is not an "established idiom"; the real argument is project size and natural evolution from extracting existing helpers

## D6: Pipeline integration mechanism

**Choice:** Tier-specific injection: TransitionExecutor decorator for reconciliation/live tiers, cloudEventSink callback for integration-live tier
**Alternatives:**
- Single cloudEventSink mechanism for all tiers — impossible: reconciliation and live tiers use SimpleTransitionExecutor directly without ReconciliationLoop, so cloudEventSink is not available
- Observer pattern with in-memory event bus — introduces a new abstraction not present in the runtime
- NodeStepExecutor decorator — captures per-node outcomes but at a lower level than TransitionResult
- GlobalReconciliationListener — cycle-level hooks with typed desired/actual state access, but only available via ReconciliationLoop (same dependency gap), less granular than CloudEvents, zero existing implementations
**Rationale:** The three provisioning tiers have two distinct execution pipelines. Reconciliation (ReconciliationTestBase) and live (K3sTestBase) construct SimpleTransitionExecutor directly — no ReconciliationLoop, no cloudEventSink, no GlobalReconciliationListener. The injection point is the TransitionExecutor interface: a decorator wrapping SimpleTransitionExecutor captures TransitionResult (Map<NodeId, StepOutcome>) after each execution. Integration-live (FleetObservabilityIntegrationTest) uses ReconciliationLoop.builder().cloudEventSink() — the production audit boundary where CloudEvents are emitted for downstream consumers (ledger, RAS, work). AuditCapture provides a unified query API over both capture mechanisms.
**Trade-offs:** Two capture mechanisms increase implementation complexity, but they match the architectural reality — the tiers have different execution pipelines by design. The TransitionExecutor decorator is a standard pattern. The cloudEventSink callback is already demonstrated in FleetObservabilityIntegrationTest.
**Sources:** ReconciliationTestBase.executeTransition() (constructs SimpleTransitionExecutor directly), K3sTestBase.planAndExecute() (same pattern), FleetObservabilityIntegrationTest.setUp() (ReconciliationLoop.builder().cloudEventSink(capturedEvents::add)), NodeStepExecutor constructor (4th param is LifecycleStepExecutor for lifecycle hooks, NOT a step outcome callback), FailableNodeProvisioner (implements NodeProvisioner SPI — provision/deprovision, no callback)
**Exploration:** quick
**Status:** revised — fixed factual errors: (1) cloudEventSink is not available in reconciliation/live tiers (no ReconciliationLoop), (2) NodeStepExecutor 4th parameter is LifecycleStepExecutor (pre/post provision hooks), not a step outcome callback, (3) FailableNodeProvisioner has no callback — it implements NodeProvisioner. Established tier-specific injection mechanisms.

## D7: Audit verification scope

**Choice:** CloudEvent emission (integration-live) + transition outcomes (reconciliation/live) — NOT end-to-end ledger/RAS/work persistence
**Alternatives:**
- Full end-to-end: verify ledger entries, RAS events, and work module audit records from topology tests — requires CDI container, foundation module dependencies, database state inspection
- CloudEvent-only: verify only CloudEvent emission — misses transition outcome verification for tiers without ReconciliationLoop
- Transport + downstream: verify CloudEvents AND mock downstream consumers — adds mock complexity without testing real CDI wiring
**Rationale:** CloudEvents are the boundary signal between the desiredstate runtime and downstream audit consumers (ledger, RAS, work). The topology tests verify the desiredstate side of this boundary. Verifying that downstream consumers correctly process CloudEvents requires a CDI-wired application context with real ledger interceptors (@Audited → AuditedInterceptor), RAS ganglion bindings, and work module integration — this is a service/ module integration test concern, not a topology test concern. The topology tests are pure Java by design (ReconciliationTestBase, K3sTestBase) — no CDI container, no Quarkus test profile. The deployment-topology-tests spec confirms this: Level 02 verifies "register payloads include audit trail configuration" (the signal), not "ledger received entries" (the downstream processing).
**Trade-offs:** Does not prove end-to-end audit trail existence at the topology test level. Trusts the CloudEvent boundary — if the right events are emitted with the right data, downstream consumers will process them correctly. This is validated by service/ module tests.
**Sources:** ARC42STORIES.MD §2 Constraints ("testing/ is test-scope only — never compile or runtime"), ReconciliationTestBase.java (pure Java, no CDI), deployment-topology-tests spec (Level 02 scope), parent epic #163 step 4 ("Assert audit — verify ledger/work/RAS recorded the operation" — applies across the full test pyramid, not exclusively to topology tests)
**Exploration:** surfaced by review
**Status:** captured — implicit decision made explicit. The parent epic's "verify ledger/work/RAS" is satisfied across the test pyramid: topology tests verify signal production, service tests verify signal consumption.

## D8: Assertion semantics for level authors

**Choice:** Assert event types and key data fields (nodeId, tenancyId, faultType) — not event ordering or exact payload structure
**Alternatives:**
- Assert only event presence (any event was emitted) — too weak, doesn't verify correctness
- Assert exact payload structure (full JSON comparison) — too brittle, breaks on any field addition in ReconciliationEventEmitter
- Assert event ordering — brittle and unnecessary: ReconciliationLoop controls ordering, not the level author's topology
- Assert event count exactly — brittle: fault recovery may produce variable event counts depending on timing
**Rationale:** Level authors need to verify that their topology produces the correct audit trail — the right events for the right nodes. Event TYPE (RECONCILIATION_COMPLETED, NODE_FAULTED, NODE_RECOVERED) confirms the pipeline recognized the condition. Key DATA fields (nodeId, tenancyId) confirm the event applies to the right entity. FaultType confirms the right failure classification. Exact payload structure and ordering are runtime implementation details that level authors should not couple to. For the reconciliation tier (TransitionResult capture), the equivalent is: assert outcome types (Succeeded, Failed, AlreadyConverged) per node ID.
**Trade-offs:** Permits weak assertions if level authors only check event types without data fields. Mitigated by providing assertion helpers that encourage specific field checks (e.g. assertEventOfType(type).withNodeId(id)).
**Sources:** FleetObservabilityIntegrationTest.java (existing assertions: eventsOfType(TYPE).isNotEmpty(), events contain specific TYPE constants), DesiredStateEventTypes (event type constants), NodeFaultedData/ReconciliationCompletedData (structured event data with nodeId, tenancyId, faultType fields)
**Exploration:** surfaced by review
**Status:** captured

## D9: Test fidelity gradient

**Choice:** Reconciliation/live tiers verify outcome production (StepOutcome types per node), integration-live tier verifies CloudEvent emission (event types + data fields)
**Alternatives:**
- Uniform assertions across all tiers (same AuditCapture API, same assertion style) — conflates signal production with signal delivery, forces the reconciliation tier to construct CloudEvents it doesn't naturally produce
- Only integration-live tier has audit assertions (reconciliation/live skip audit) — leaves fast tests without audit verification, defeats the purpose of a test pyramid
**Rationale:** The reconciliation and live tiers execute transitions via SimpleTransitionExecutor and produce TransitionResult (Map<NodeId, StepOutcome>). These tiers verify that the execution pipeline PRODUCES the right outcomes — Succeeded for clean provision, Failed for fault injection, AlreadyConverged for idempotent re-runs. The integration-live tier runs ReconciliationLoop which emits CloudEvents via cloudEventSink. This tier verifies that outcomes are EMITTED as properly formed CloudEvents with correct event types and data. The fidelity gradient matches the tier purposes: fast mocked tests (reconciliation) verify behavior correctness, slow real-service tests (integration-live) verify signal emission, real-infrastructure tests (live) verify provisioning against actual systems.
**Trade-offs:** Two assertion vocabularies (StepOutcome assertions vs CloudEvent assertions) increase learning curve. AuditCapture provides a unified query API that abstracts this, but level authors still need to understand which assertions are meaningful at their tier.
**Sources:** SimpleTransitionExecutor.execute() (returns TransitionResult), ReconciliationLoop (emits CloudEvents via ReconciliationEventEmitter → cloudEventSink), existing test patterns (ReconciliationTestBase asserts TransitionResult outcomes, FleetObservabilityIntegrationTest asserts CloudEvent types)
**Exploration:** surfaced by review
**Status:** captured
