## D1: Environment abstraction architecture

**Choice:** TargetEnvironment enum + factory pattern in testing module
**Alternatives:**
- CDI with test profiles — highest fidelity to production but requires migrating all topology tests to @QuarkusTest, massive scope expansion
- TopologyRunner harness — hides provisioner wiring but requires refactoring tests that currently call provisioners directly
**Rationale:** Aligns with the existing test architecture (vanilla JUnit 5, manual wiring), reuses the ApplicationDeploymentTarget SPI from #117, and is the smallest change to the current test model. Each environment gets its own JUnit lifecycle extension.
**Trade-offs:** Factory is test-only — production uses CDI for target selection. This is acceptable since topology tests are deliberately outside the CDI container for isolation and speed.
**Sources:** #117 deployment profiles spec, ClaudonyNodeProvisioner (topology-tests), ServiceClusterExtension, ApplicationDeploymentTarget SPI
**Exploration:** quick
**Status:** captured

## D2: Provisioner dispatch for casehub_application across environments

**Choice:** Same NodeProvisioner, swap ApplicationDeploymentTarget per environment
**Alternatives:**
- Separate NodeProvisioner implementations per environment — more explicit but duplicates the delegation logic that ApplicationDeploymentTarget was designed to abstract
**Rationale:** The ApplicationDeploymentTarget SPI from #117 was designed for exactly this: same node type, different deployment mechanism. The test provisioner extracts CaseHubApplicationSpec and delegates to whichever target the environment provides. Production uses the same pattern via CDI.
**Trade-offs:** The test provisioner is a thin adapter that mirrors production's CaseHubAppProvisionHandler. If the handler's logic changes, the test adapter needs updating — but this is a feature, not a bug: tests should track production behavior.
**Sources:** #117 deployment profiles spec, CaseHubAppProvisionHandler, ApplicationDeploymentTarget SPI
**Exploration:** quick
**Depends on:** D1 (environment abstraction provides the target)
**Status:** captured

## D3: K8s cluster tool

**Choice:** k3d (lightweight k3s wrapper)
**Alternatives:**
- kind (full Kubernetes in Docker) — heavier (~30-45s startup, ~1GB RAM), better conformance coverage but topology tests verify deployment correctness not K8s conformance
**Rationale:** k3d starts in ~15s with ~512MB RAM and includes built-in image import. Topology tests need a cluster that can run K8s Deployments and Services — not full conformance testing. k3s is CNCF-certified Kubernetes.
**Trade-offs:** k3s has minor API differences from full Kubernetes (e.g., default storage class, some alpha features). For topology deployment testing these don't matter.
**Sources:** k3d documentation, kind documentation, existing @KubernetesAvailable condition in topology-tests/live/
**Exploration:** quick
**Depends on:** D1 (K8s lifecycle extension manages k3d cluster)
**Status:** captured

## D4: Module placement for new code

**Choice:** testing/ module (shared test fixtures)
**Alternatives:**
- topology-tests/ module — co-located with consumers but prevents reuse by other test modules
**Rationale:** testing/ is the established shared test fixtures module. TargetEnvironment, TargetEnvironmentFactory, and lifecycle extensions belong alongside other test infrastructure. topology-tests already depends on testing/.
**Trade-offs:** Adds dependencies (fabric8 client, PodmanClient) to testing/ module's test scope. Acceptable since testing/ is never compile/runtime scope.
**Sources:** testing/ module structure, existing shared fixtures (AuditCapture, AuditAssertions)
**Exploration:** quick
**Status:** captured

## D5: Backward compatibility for existing integration tests

**Choice:** Default to LOCAL — existing tests unchanged, new profiles are additive
**Alternatives:**
- None considered — backward compatibility is a hard constraint, not a design choice
**Rationale:** TargetEnvironmentFactory defaults to LOCAL when casehub.topology.target-environment is unset. The -Pintegration-live profile continues to run exactly as today. New -Ptarget-k8s and -Ptarget-podman profiles are additive. Existing tests only gain multi-env support when a test author opts in by reading the target from the factory.
**Trade-offs:** Existing tests that hardcode ClaudonyDeploymentTarget won't automatically run on K8s/Podman. Migration is opt-in per test. This is deliberate — forced migration would break the acceptance test suite.
**Sources:** Existing @Tag("integration-live") tests, ServiceClusterExtension, -Pintegration-live Maven profile
**Exploration:** quick
**Status:** captured
