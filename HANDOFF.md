# Handover — deployment topology integration tests

**Branch:** `issue-129-hello-llm-deployment-topology` (closed, landed on main)
**Issue:** casehubio/casehub-ops#129 (closed)
**Status:** Level 01 complete. Ready for level 02.

## What happened

Designed and implemented a data-driven deployment topology integration test suite in `topology-tests`. The suite validates deploying CaseHub apps via claudony through the full YAML → compile → reconcile → provision pipeline.

**Key design decisions:**
- The minimum honest CaseHub hello world needs five services: manifest + eidos + engine + blocks + LLM — all deployed via claudony. Anything less and something can't function.
- Claudony is the deployment target from day one. Without it, desiredstate adds nothing over `docker run`.
- Tests are tutorial-style: each YAML has header comments explaining what it teaches. `INDEX.md` provides the reading order.
- Nine levels planned, each adding one platform capability. Iterative — reassess after each level.

**What was built (level 01):**
- `01-hello-llm.yaml` — three `casehub_application` nodes (eidos → engine → my-llm) with dependency ordering
- `DeploymentCompilationTest` — 5 tests (graph structure, types, deps)
- `DeploymentReconciliationTest` — 5 tests (ordering, execution, idempotency)
- `DeploymentTargetIntegrationTest` — 2 tests (WireMock claudony fleet API)
- `ClaudonyWireMockSupport` — reusable WireMock stubs for claudony API
- `TopologyTestBase` extended with `buildTypeRegistryFrom()` and `compileSingleGraph(path, registry)` overloads
- Blog post: "The Minimum Honest LLM Deployment" with 3 SVG diagrams

**Pre-existing issues fixed along the way:**
- `TransitionPlan` API: `additions()` → `flatAdditions()` across 8 test files
- `SimpleTransitionExecutor`: now wraps `NodeStepExecutor`
- `YamlGraphRecorder`: dropped factory provider parameters
- `YamlModule`/`YamlModuleFile`: package migration to `yaml.core.module`

**Pre-existing issues NOT fixed (infra tests still broken):**
- `K8sNamespaceSpec` cannot be cast to `NodeSpec` — version mismatch between infra module and desiredstate API
- Module YAML `ParameterType` enum case mismatch (`string` vs `STRING`)

## Next work

**Level 02: + ledger + work** — adds accountability to the hello-llm topology. Each deployment is audited, each task generates work items. This is a separate issue/branch.

The full 9-level progression is in the spec: `/Users/mdproctor/claude/public/casehub-ops/specs/deployment-topology-tests/2026-10-14-deployment-topology-integration-tests-design.md`

**Infra test fixes:** The pre-existing infra topology test failures should be addressed in a separate issue — they're unrelated to deployment topology work but block `mvn test` on the full topology-tests module.

## Key files

- `topology-tests/src/test/resources/topologies/deployment/01-hello-llm.yaml`
- `topology-tests/src/test/java/io/casehub/ops/topology/compilation/DeploymentCompilationTest.java`
- `topology-tests/src/test/java/io/casehub/ops/topology/reconciliation/DeploymentReconciliationTest.java`
- `topology-tests/src/test/java/io/casehub/ops/topology/deployment/DeploymentTargetIntegrationTest.java`
- `topology-tests/src/test/java/io/casehub/ops/topology/deployment/ClaudonyWireMockSupport.java`
