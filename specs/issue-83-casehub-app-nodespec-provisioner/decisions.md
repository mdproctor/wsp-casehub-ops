## D1: Module placement

**Choice:** Deployment module — add to DeploymentNodeSpec sealed hierarchy
**Alternatives:**
- New module — clean separation but introduces cross-domain composition complexity and a new SPI quad
- Container module — closest to provisioning mechanism but loses topology composition with deployment nodes
**Rationale:** casehub-application is a topology concept that composes with load-balancers, HA, service-mesh in the same deployment graph. The provisioner delegates container lifecycle to a deploy-target SPI.
**Trade-offs:** Deployment module grows larger; the provisioning mechanism (build + deploy container) differs from existing handlers that call foundation APIs.
**Sources:** DeploymentNodeSpec.java, DeploymentNodeProvisioner.java, ContainerNodeSpec.java, engine#1017
**Exploration:** quick
**Status:** captured

## D2: Build pipeline architecture

**Choice:** BuildPipeline SPI delegation — define SPI in ops-api, stub implementation now
**Alternatives:**
- Inline stub pipeline — simpler initially but harder to swap when engine#1018/#1019 arrive
- Deploy-only scope — limits #83 to pre-built images, deferring build entirely
**Rationale:** engine#1018 (schema-driven record generation) and #1019 (casehub.yaml descriptor) don't exist yet. SPI lets the provisioner focus on deploy + health check + reconciliation. Real build pipeline plugs in when engine delivers.
**Trade-offs:** Extra abstraction layer for something that has only one implementation initially.
**Sources:** engine#1017 epic, engine#1018, engine#1019
**Exploration:** quick
**Depends on:** D1 (module placement determines where the SPI lives)
**Status:** captured

## D3: Deployment target abstraction

**Choice:** DeploymentTarget SPI — deploy, undeploy, status, healthCheck
**Alternatives:**
- Direct PodmanClient use — simpler but couples to Podman, requires refactoring for K8s
- Reuse container module via CrossDomainCompositionEngine — elegant but adds graph complexity
**Rationale:** The issue envisions Docker, K8s, and cloud run targets. A DeploymentTarget SPI allows PodmanDeploymentTarget now with K8s and cloud run later, matching the multi-target vision from #117 (deployment profiles).
**Trade-offs:** Another SPI when only one backend exists initially. PodmanDeploymentTarget will duplicate some logic from the container module's PodmanNodeProvisioner.
**Sources:** ContainerNodeSpec.java, PodmanNodeProvisioner.java, issue #117 (deployment profiles)
**Exploration:** quick
**Depends on:** D1 (module placement), D2 (build pipeline produces an image ref that the target deploys)
**Status:** captured
