# Handoff — casehub-ops

## Last Session
Completed Tasks 4–8 of the #38 implementation plan. ServiceUpgradeCaseDescriptor (four-phase upgrade pattern), CveStatusObserver (CDI lifecycle event → CveStore), ApplicationEventBroadcaster (per-app ring buffer, filtered subscriptions, gap detection), ScalingService extraction from ScalingResource, and full REST endpoint wiring across 6 resources (ApplicationResource PUT, DeploymentResource list/current/rollback, CaseResource list/get/events, ReconciliationResource status/trigger/events, SecurityResource getCves/scanCves/getPosture, ServiceOperationResource status/scale/upgrade). Added blocks-ui repo to slot 86. 462 tests passing, 0 regressions (3 pre-existing KubernetesFaultPolicyTest failures unchanged).

Epic #29 progress: 4/6 issues done (batches 1-3 complete). Batch 4 (#38) in progress — 8/9 tasks done.

## Immediate Next Step
Run `/work` in slot 86 to continue #38. Task 9 is the only remaining task: five blocks-ui Lit 3.x web components. The blocks-ui repo is already cloned into `slots/86/blocks-ui` on branch `issue-38-stubbed-ui-screens`. Implementation follows the existing component convention in blocks-ui — `DataSourceMixin(LitElement)`, Shadow DOM, vitest. Components to build:

1. **blocks-service-card** — per-service health card (health badge, replicas, image, per-cluster status). Props: `data?: ServiceCardData`, `endpoint?: string`. Event: `service-card.selected`.
2. **blocks-cluster-panel** — cluster list with registration form and connectivity test. Props: `data?: ClusterInfo[]`, `endpoint?: string`, `readonly?: boolean`. Events: `cluster.registered`, `cluster.deleted`, `cluster.tested`.
3. **blocks-reconciliation-status** — desired vs actual diff with per-node status indicators, SSE live updates. Props: `data?: ReconciliationSnapshot`, `endpoint?: string`, `sseEndpoint?: string`. Events: `reconciliation.node-selected`, `reconciliation.trigger-requested`.
4. **blocks-dimension-dashboard** — N dimensions with status indicators and severity badges, compact layout option. Props: `data?: DimensionDashboardData`, `endpoint?: string`, `compact?: boolean`. Event: `dimension.selected`.
5. **blocks-topology-viewer** — service dependency DAG using `graph-renderer` + `computeElkLayout`, extending `blocks-dag-viewer` with custom node types. Props: `data?: TopologySnapshot`, `endpoint?: string`, `sseEndpoint?: string`. Event: `topology.node-selected`. Dependencies: `@casehubio/graph-core`, `@casehubio/graph-renderer`.

Each component: create package dir, copy package.json/tsconfig from `compliance-summary` template, define types, write failing test, implement, run `npx vitest run`. Commit all five together.

## What's Left
- Pre-existing: @QuarkusTest + H2 + Hibernate 6.6 JOINED inheritance DDL failure (GE-20260718-d18dc0) — handled via @Disabled, needs TestContainers migration
- Pre-existing: @QuarkusTest offline config failure — upstream config properties missing in offline mode (GE-20260814-8bc7ef). Non-QuarkusTest unit tests (462) pass clean.
- GitHub Packages token expired — refresh before next session for online builds
- `epic_manager.py advance` bug — doesn't skip already-done issues. Needs fix in soredium.

## References
- Spec: `specs/issue-38-stubbed-ui-screens/2026-08-14-stubbed-ui-screens-full-implementation-design.md`
- Decisions: `specs/issue-38-stubbed-ui-screens/decisions.md`
- Plan: `plans/2026-08-14-stubbed-ui-screens.md`
- Architecture: `ARC42STORIES.MD`
- Blog: `blog/2026-08-14-mdp01-stubs-to-services.md`
- Garden: GE-20260814-0fcc1a (CaseLifecycleEvent observer), GE-20260814-58bc55 (ReconciliationLoop no-status), GE-20260814-cba922 (JAX-RS path conflict)
