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
