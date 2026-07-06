*Updated: parent#337 closed — removed from What's Left.*

# Handoff — casehub-ops

## Last Session
Designed and implemented Phase 1 (Foundation) of the ops console app — a Quarkus application for deploying and managing K8s microservices through their full lifecycle via the service-as-case model. Concluded #30 (Case works for long-lived services, unchanged). Design review: 5 rounds, 28 issues, all verified ($22.86). Phase 1: 10 commits, 182 tests green, app/ module scaffolded with models, entities, goal compiler, services, REST API, SPI stubs. 3 garden entries captured (CDI embedding anti-pattern, FaultPolicyEngine List injection, surefire hang).

## Immediate Next Step
Continue on branch `issue-29-service-lifecycle-ops-console`. Write the Phase 2 plan (`docs/plans/2026-07-XX-ops-console-phase2-kubernetes.md`) for KubernetesBackend, fabric8 resource provisioners, actual state adapter, drift detection, startup recovery. Then execute via subagent-driven-development. Run `/work` to resume.

## Cross-Module
**Enables:**
- `engine#584` — remains open until at least one consumer migrates

## What's Left
- ops#29 Phase 2: Kubernetes integration — KubernetesBackend (InfraBackend SPI), fabric8 resource provisioners, KubernetesActualStateAdapter, drift detection, startup recovery, integration tests with mock server · L · Med
- ops#29 Phase 3: Case lifecycle — child cases, CVE response, upgrade, approval workflow · L · Med
- ops#29 Phase 4: UI — blocks-ui components, casehub-pages shell, wizard, dashboard · L · High
- ops#29 Phase 5: Demo — Kind setup, online-store.yaml, scenario scripts · M · Med
- ops#39 K8sDeploymentSpec extension + K8sConfigMapSpec — **DONE** (committed on branch, not yet on main) · S · Low
- desiredstate#54 requiresHuman gating for deprovision · S · Low
- desiredstate#55 Update dungeon example · XS · Low
- Pause stack: issue-18-real-evidence-collectors (1 paused branch)
- Project main has design review commits ahead of origin/main — push needed

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #29 | Phase 2: K8s integration | L | Med | Natural next step — fabric8, drift detection, startup recovery |
| #32 | Approval trigger mechanism | M | Med | WorkItem/REST for approve/reject — can interleave with Phase 3 |
| #10 | IoTFaultPolicy | M | Med | Deferred — needs operational feedback |
| #11 | DetectionNodeSpec — RAS | M | Med | Blocked by RAS declarative API |
| #16 | Compliance demo | M | Med | Unblocked |
| #17 | Infra demo | M | Med | Unblocked |
| #18 | Real EvidenceCollectors | M | Med | Paused branch exists |
| #19 | Integration test hardening | M | Low | Unblocked |
| #25 | fsitrading adaptive ops | L | High | Unblocked — ras#20 closed |
| #26 | SOC adaptive ops | L | High | Unblocked — ras#20 closed |

## References
- Design spec: `docs/superpowers/specs/2026-07-05-ops-console-app-design.md`
- Phase 1 plan: `docs/plans/2026-07-05-ops-console-phase1-foundation.md`
- SDD progress ledger: `.hortora/sdd/progress.md`
- Architecture: `ARC42STORIES.MD`
- Diary: `blog/2026-07-06-mdp01-everything-is-a-case.md` (workspace)
- Design journal: `design/JOURNAL.md` (workspace)
