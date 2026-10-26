# Handoff — casehub-ops

## Active Work

**Branch:** `issue-150-multi-env-deployment`
**Issues:** #150 (epic), #151, #152, #153
**State:** Active — design complete, implementation plan written, ready to execute.

## What to Do

Run `work continue` to pick up. Execute the implementation plan at `/Users/mdproctor/claude/public/casehub-ops/plans/2026-10-26-multi-env-deployment.md`.

Use `executing-plans` to implement task-by-task. 7 tasks across 4 batches:
1. **Foundation (#151):** TargetEnvironment enum, EnvironmentAwareNodeProvisioner, TargetEnvironmentFactory, Maven profiles, regression verification
2. **K8s (#152):** KubernetesDeploymentTarget, K3dClusterExtension
3. **Podman (#153):** PodmanEnvironmentExtension
4. **Verification:** Full project build + regression

## Context

**Design spec:** `/Users/mdproctor/claude/public/casehub-ops/specs/issue-150-multi-env-deployment/2026-10-26-multi-env-deployment-design.md`

**Key design decisions (5 total, all quick picks):**
- D1: TargetEnvironment enum + factory pattern (not CDI, not TopologyRunner)
- D2: Same NodeProvisioner, swap ApplicationDeploymentTarget (not separate provisioners)
- D3: k3d via testcontainers-k3s (not kind)
- D4: Lightweight abstractions in testing/, heavy wiring in topology-tests/
- D5: Default to LOCAL, new profiles additive (backward compatible)

**What exists:**
- `ApplicationDeploymentTarget` SPI in `api/` — `deploy()`, `undeploy()`, `status()`, `healthCheck()`
- `PodmanDeploymentTarget` in `container/` — ready to use
- `Fabric8NodeProvisioner` in `topology-tests/live/` — K8s patterns proven
- `ClaudonyNodeProvisioner` in `topology-tests/integration/` — current provisioner to be generalized
- testcontainers-k3s and fabric8 already in `topology-tests/pom.xml`

## Recently Completed

**Design phase (2026-10-26):**
- Brainstormed #150 design — discovered #117 deployment profiles already designed most of the production infrastructure
- Scoped #150 to test-level multi-env: same YAML, different provisioner via environment abstraction
- 5 decisions captured, all quick picks applying existing patterns
- Design spec and implementation plan written and committed
