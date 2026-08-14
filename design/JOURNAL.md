# Design Journal — issue-38-stubbed-ui-screens

## 2026-08-14 — Scope pivot and design

Issue #38 ("stubbed UI screens") pivoted significantly during brainstorming:

1. **No UI exists** — the app module has REST endpoints only, no HTML/TS/Lit components
2. **Scaffold UI epic** — generic operational screens (case browsers, state browsers) belong in scaffold, not ops. Filed as a separate future concern.
3. **blocks-ui components** — ops-specific building blocks go in blocks-ui as generic components (not ops-branded). Five new: service-card, cluster-panel, reconciliation-status, dimension-dashboard, topology-viewer.
4. **Backend scope expanded** — "wire everything" includes CVE registry (CveStore with JPA), two new case descriptors (cve-response, service-upgrade), SSE infrastructure (ApplicationEventBroadcaster), and ScalingService extraction.

Key design decisions: CveStore in app/ not api/ (no cross-domain consumers), topology viewer extends blocks-dag-viewer (no new stencil), gap event for SSE reconnection, case-level approval via ActionRiskClassifier (not ApprovalEvaluator).

Implementation: 3 of 9 tasks complete (CveStore, updateServiceImage/rollbackToDeployment, CveResponseCaseDescriptor).
