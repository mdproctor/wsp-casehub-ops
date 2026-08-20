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
