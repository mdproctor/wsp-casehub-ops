## D1: Test class structure

**Choice:** New test class per module — `DeploymentReconciliationIntegrationTest` and `ComplianceReconciliationIntegrationTest`
**Alternatives:**
- Extend existing `DeploymentLifecycleIntegrationTest` — blurs SPI-level and cycle-level concerns in one class
- Both (extend + new) — unnecessary duplication
**Rationale:** Existing tests verify SPI implementations work individually (provision, drift check, event source). Reconciliation tests verify the full planner-in-the-loop cycle. Different concerns, different classes.
**Trade-offs:** Two more test files to maintain; some setup overlap with existing tests
**Sources:** `DeploymentLifecycleIntegrationTest.java`, `IoTReconciliationIntegrationTest.java`
**Exploration:** quick
**Status:** captured

## D2: Stub sharing strategy

**Choice:** Extract stubs to the `testing/` module (`casehub-ops-testing`) — shared across deployment, compliance, and future domain module tests
**Alternatives:**
- Duplicate inline in each test — simple but 200+ lines of copy-paste per test class
- Package-level classes in deployment/src/test — only available within the deployment module, compliance can't share
**Rationale:** The `testing/` module already exists for this purpose per CLAUDE.md ("Shared test fixtures. Test scope only."). Both deployment and compliance reconciliation tests need the same stubs. Future modules (infra, iot consumers) will too.
**Trade-offs:** Requires updating the existing `DeploymentLifecycleIntegrationTest` to use shared stubs (minor churn on an existing test)
**Depends on:** D1 (new test class per module)
**Sources:** `testing/pom.xml`, CLAUDE.md module table
**Exploration:** quick
**Status:** captured
