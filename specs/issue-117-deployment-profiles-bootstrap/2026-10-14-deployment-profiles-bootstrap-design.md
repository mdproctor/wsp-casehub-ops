# Deployment Profiles and Bootstrap Model

**Issue:** casehubio/casehub-ops#117
**Branch:** issue-117-deployment-profiles-bootstrap
**Scale:** L — architecture design, touches deployment module, container module, app module, topology YAML
**Prerequisites:** #118 (service extraction — ops embeddable as library), #83 (CaseHubApplicationSpec + ApplicationDeploymentTarget SPIs)

## Problem

Ops + desiredstate is designed to deploy and manage a full CaseHub installation from a single YAML topology declaration. But there is no definition of how the system bootstraps from day zero, how different deployment scales are handled, or how the ops app maintains itself after bootstrap.

## Solution Overview

Three deployment profiles representing the same topology at different scales, a two-phase bootstrap model where the ops app is the seed that provisions everything else, and a self-maintenance loop where the ops app manages its own upgrades.

Platform services (Qhorus, Eidos, Engine) are modeled as `CaseHubApplicationSpec` instances — they are CaseHub applications deployed through the `ApplicationDeploymentTarget` SPI established in #83. Infrastructure (PostgreSQL, networking) uses existing infra and container domain types (`DatabaseContainerSpec`, `DatabaseClusterSpec`). The deployment profile selects which `ApplicationDeploymentTarget` implementation is active.

The core principle is **"use existing or deploy if missing"** — this is standard reconciliation loop behavior: `ActualStateAdapter` reports per-node `NodeStatus`, `TransitionPlanner` plans transitions for ABSENT/DRIFTED nodes, `NodeProvisioner` provisions. Domain-specific health probes are implementation details of each domain's `ActualStateAdapter`.

### Profiles

| Profile | CaseHub property | Default deployment target | Persistence | Use case |
|---------|-----------------|--------------------------|-------------|----------|
| `local-dev` | `casehub.ops.deployment-profile=local-dev` | `InProcessDeploymentTarget` | In-memory / H2 | Developer inner loop. Single `java -jar`, everything in one CDI container. |
| `local-distributed` | `casehub.ops.deployment-profile=local-distributed` | `PodmanDeploymentTarget` | PostgreSQL container | Integration testing. Real network boundaries via Podman containers. |
| `enterprise` | `casehub.ops.deployment-profile=enterprise` | `KubernetesDeploymentTarget` | Production PostgreSQL, TLS | Real deployment. Same YAML, production-grade targets. |

The deployment profile is a CaseHub-level configuration property, separate from Quarkus profiles (`%dev`, `%test`, `%prod`). Quarkus profiles continue to control framework behavior (H2 vs PostgreSQL datasource, Hibernate schema management). The CaseHub deployment profile controls which `ApplicationDeploymentTarget` is active for platform service provisioning.

Individual nodes can override the default deployment target: `deploymentTarget: podman` inside a node's `spec:` block forces Podman deployment even when the profile default is `in-process`. This follows the same pattern as infra's `backendId` — routing metadata lives inside the spec map and is extracted by the `NodeSpecFactory` before converting to the typed record (see `InfraWrappingFactory`).

### Profile as a Spectrum

Profiles are not rigid buckets. `local-dev` can range from "everything in-memory in-process" to "some services persistent" to "some services in separate containers." The profile sets the default; per-node overrides provide the spectrum. A developer might run `local-dev` with Qhorus embedded but PostgreSQL in a container — the topology YAML supports this via per-node `deploymentTarget` overrides inside the `spec:` block.

## Architecture

### Topology YAML Layers

The full system topology is declared as standard desiredstate YAML graphs at the established classpath location `META-INF/desiredstate/`, using the `desiredState:` root format processed by `YamlDesiredStateProcessor`. Cross-layer dependency ordering is managed by `CrossDomainCompositionEngine` via Java-level `DomainRegistration` (provides/requires on `NodeType`):

```
META-INF/desiredstate/
  casehub-topology.yaml        # all platform services + ops app (same across all profiles)
```

All `casehub_application` nodes (Qhorus, Eidos, Engine, ops-app) live in a single topology YAML file. This ensures `dependsOn` references resolve within one `DesiredStateGraph` — cross-file `dependsOn` between separate YAML graphs is not supported without `CrossDomainCompositionEngine` domain registration, and two domains providing the same `NodeType` triggers `validateDuplicateProvides()`.

Infrastructure is profile-specific — different profiles use different infrastructure types (`DatabaseContainerSpec` vs `DatabaseClusterSpec`, `NetworkSpec` vs `K8sNamespaceSpec`). Profile-specific infra is loaded from `META-INF/desiredstate/modules/` and imported by the main topology:

```
META-INF/desiredstate/modules/
  casehub-infra-local-dev.yaml         # H2 placeholder, no-op network
  casehub-infra-local-distributed.yaml # container:database, podman network
  casehub-infra-enterprise.yaml        # database_cluster, k8s namespace
```

Reusable infrastructure modules (load-balancer, HA, service-mesh) remain at this established `META-INF/desiredstate/modules/` path alongside the profile-specific infra modules.

**Location:** shipped in the ops app JAR at `META-INF/desiredstate/`. A config property `casehub.ops.topology.path` points to a filesystem directory for installation-specific overrides. Classpath is the default; filesystem takes precedence when specified.

**YAML format:** standard `desiredState:` graph format with `imports:` for reusable module composition. No `provides`/`requires` in YAML — domain dependency ordering is registered at the Java level via `DomainRegistration` with `CrossDomainCompositionEngine`.

**YAML processing prerequisite — DeploymentNodeSpecFactoryProvider:** The `desiredState:` YAML format resolves node types via `NodeSpecFactory` implementations registered through `NodeSpecFactoryProvider` SPI. The infra domain has `InfraNodeSpecFactoryProvider`, the container domain has `ContainerNodeSpecFactoryProvider`. The deployment domain currently lacks one — it uses `DeploymentGoalLoader` with its own YAML format.

This spec adds `DeploymentNodeSpecFactoryProvider` in the `deployment/` module, following the established pattern:

```java
public class DeploymentNodeSpecFactoryProvider implements NodeSpecFactoryProvider {
    @Override
    public Map<String, NodeSpecFactory> provide() {
        ObjectMapper mapper = new ObjectMapper();
        return Map.of("casehub_application", specMap -> {
            Map<String, Object> fields = new LinkedHashMap<>(specMap);
            String deploymentTarget = (String) fields.remove("deploymentTarget");
            CaseHubApplicationSpec appSpec = mapper.convertValue(fields, CaseHubApplicationSpec.class);
            return deploymentTarget != null
                ? new DeploymentApplicationNodeSpec(appSpec, deploymentTarget)
                : appSpec;
        });
    }
}
```

When `deploymentTarget` is present in the spec map, the factory wraps the spec in `DeploymentApplicationNodeSpec(CaseHubApplicationSpec spec, String deploymentTargetId)` implementing `NodeSpec` — mirroring the `InfraDesiredNodeSpec(InfraNodeSpec, backendId)` pattern. When absent, the bare `CaseHubApplicationSpec` is returned and the profile default target is used.

**Cross-domain ordering — production DomainRegistration wiring:** The `CrossDomainCompositionEngine` has been tested but not wired in production. This spec adds `BootstrapDomainRegistrar` CDI beans that register domains at startup:

```java
@ApplicationScoped
public class BootstrapDomainRegistrar {
    @Inject CrossDomainCompositionEngine engine;

    void onStartup(@Observes StartupEvent event) {
        // Container domain: provides container:database, network, volume
        engine.registerDomain(DomainRegistration.builder(
            DomainId.of("container"), containerCompilationResult)
            .provides(Set.of(NodeType.of("container:database"), NodeType.of("network"), NodeType.of("volume")))
            .build());

        // Deployment domain: provides casehub_application, requires container:database
        engine.registerDomain(DomainRegistration.builder(
            DomainId.of("deployment"), deploymentCompilationResult)
            .provides(Set.of(NodeType.of("casehub_application")))
            .requires(Set.of(NodeType.of("container:database")))
            .build());
    }
}
```

This ensures `TransitionPlanner` provisions infra nodes before deployment nodes. The registrar must fire before the first reconciliation cycle — `@Observes StartupEvent` guarantees this.

### Platform Services as CaseHub Applications

Platform services (Qhorus, Eidos, Engine, and the ops app itself) are modeled as `CaseHubApplicationSpec` instances from the deployment domain (#83). They are CaseHub applications — they have images, configuration, and deploy to the same targets as any other CaseHub app:

| Service | applicationId | image | Notes |
|---------|--------------|-------|-------|
| Qhorus | `qhorus` | `casehubio/qhorus:0.5-SNAPSHOT` | Mesh communication |
| Eidos | `eidos` | `casehubio/eidos:0.3-SNAPSHOT` | Agent identity |
| Engine | `engine` | `casehubio/engine:0.4-SNAPSHOT` | Case orchestration |
| Ops App | `ops-app` | `casehubio/casehub-ops-app:latest` | Self-referential — bootstrap seed |

This reuses the existing `CaseHubAppProvisionHandler` orchestration: build pipeline → deployment target → health check. The `PrebuiltImagePipeline` returns `spec.image()` directly. No new sealed hierarchy needed — platform services ARE CaseHub applications.

`CaseHubApplicationSpec.runtime` is nullable — platform services that are not CaseHub engine applications set `runtime: null`. The constructor does not validate `runtime` for null.

**PostgreSQL is infrastructure, not a CaseHub application.** PostgreSQL provisioning uses existing domain types per profile:
- **local-dev:** H2 via Quarkus `%dev` datasource config — no provisioning node needed
- **local-distributed:** `DatabaseContainerSpec` (`container:database`) → `PodmanNodeProvisioner` — existing container domain
- **enterprise:** `DatabaseClusterSpec` (`database_cluster`) → `InfraNodeProvisioner` → `InfraBackend` — existing infra domain

### Reconciliation-Based Provisioning

The bootstrap topology uses the existing reconciliation loop — no new adapter types or mechanism names are introduced:

1. `ActualStateAdapter` implementations query actual state per node type (`DeploymentActualStateAdapter` for platform services, `PodmanActualStateAdapter` for containers, `InfraActualStateAdapter` for infra)
2. `TransitionPlanner` plans transitions for ABSENT/DRIFTED nodes (DRIFTED handling via casehub-desiredstate#38)
3. `NodeProvisioner` provisions using the active `ApplicationDeploymentTarget`
4. `FaultPolicy` handles provisioning failures per domain convention

**Health probes** are implementation details of the relevant `ActualStateAdapter`:

| Service | Probe | Adapter |
|---------|-------|---------|
| PostgreSQL (container) | `PodmanClient` container status | `PodmanActualStateAdapter` |
| PostgreSQL (H2/local-dev) | JDBC connection test on Quarkus datasource | Not a topology node — Quarkus config |
| Qhorus, Eidos, Engine (container/K8s) | HTTP GET `/q/health/ready` | `DeploymentActualStateAdapter` via `CaseHubAppDriftChecker` |
| Qhorus, Eidos, Engine (in-process) | CDI bean availability check | `DeploymentActualStateAdapter` via `CaseHubAppDriftChecker` |

**Drift detection:** standard reconciliation behavior. The `ActualStateAdapter` continues monitoring provisioned services. If a service goes down, the adapter reports ABSENT/DRIFTED and `TransitionPlanner` plans corrective action per the existing reconciliation protocol.

### ApplicationDeploymentTarget Implementations

Three `ApplicationDeploymentTarget` implementations extend the SPI from #83. `ApplicationDeploymentTarget` gains a `targetId()` default method:

```java
public interface ApplicationDeploymentTarget {
    default String targetId() { return ""; }
    DeployResult deploy(String imageRef, DeployConfig config);
    DeployResult undeploy(String instanceId);
    NodeStatus status(String instanceId);
    HealthCheckResult healthCheck(String instanceId);
}
```

The default method avoids breaking existing implementations:
- **`NoOpDeploymentTarget`** (`@DefaultBean` in `api/`) — inherits `targetId() = ""`, remains as CDI fallback. Not discoverable by target routing (empty ID never matches).
- **`StubDeploymentTarget`** (`testing/`) — inherits `targetId() = ""`, continues working as-is in tests.

All three new implementations are `@ApplicationScoped` CDI beans coexisting on the classpath, each overriding `targetId()`. `CaseHubAppProvisionHandler` is updated to inject `Instance<ApplicationDeploymentTarget>` (replacing the current single injection) and resolve by `targetId()` when a `DeploymentApplicationNodeSpec` carries a `deploymentTargetId`. For bare `CaseHubApplicationSpec` nodes (no per-node override), the handler uses the profile-default target selected via CDI alternative. `NoOpDeploymentTarget` with `@DefaultBean` is only active when no real target is configured — it's superseded by any CDI alternative.

**InProcessDeploymentTarget** (`local-dev` default, `targetId() = "in-process"`):
- In-process mode assumes all platform services are on the classpath at startup — a direct consequence of #118 (service extraction into embeddable libraries). `InProcessDeploymentTarget.deploy()` is a verification-only operation: it confirms CDI bean presence but cannot add beans at runtime. Missing classpath dependencies fail the bootstrap immediately with a clear error, not retry.
- `deploy()`: Verify CDI beans are resolvable for the given applicationId → return `Success(applicationId)`. Fail immediately if beans are absent.
- `undeploy()`: No-op (CDI beans can't be removed from a running container)
- `status()`: Check CDI bean injection → `PRESENT` if resolvable, `ABSENT` if not
- `healthCheck()`: Invoke bean's health endpoint if available → `Healthy` or `Unhealthy`

**PodmanDeploymentTarget** (`local-distributed` default, `targetId() = "podman"`):
- Delegates to existing `PodmanClient` for container lifecycle
- `deploy()`: Pull/build image, create container with port/env/volume mapping → `Success(containerId)`
- `undeploy()`: Stop and remove container
- `status()`: Query container status via `PodmanClient`
- `healthCheck()`: HTTP health endpoint after container starts

**KubernetesDeploymentTarget** (`enterprise` default, `targetId() = "kubernetes"`):
- K8s Deployment + Service per application
- `deploy()`: `kubectl apply` Deployment spec via fabric8 client → `Success(deploymentName)`
- `undeploy()`: Delete Deployment and Service
- `status()`: Query Deployment readiness conditions
- `healthCheck()`: K8s readiness probe result via `K8sWatchManager`

This follows the same pattern as `InfraBackend` discovery (ARC42STORIES §Layer L2): all implementations coexist as `@ApplicationScoped` beans, resolved by identifier at dispatch time.

### Two-Phase Bootstrap

```
Phase 1 — SEED (ops app starts, provisions dependencies)
├── Load topology YAML (classpath META-INF/desiredstate/casehub-topology.yaml, standard YamlDesiredStateProcessor discovery)
├── Start reconciliation loop with tenancyId from casehub.ops.app.default-tenancy-id
├── Provision infra layer (via CrossDomainCompositionEngine ordering):
│   ├── PostgreSQL (H2 in local-dev [Quarkus config], container in local-distributed, managed DB in enterprise)
│   ├── Networking (no-op in local-dev, Podman network in local-distributed, K8s namespace in enterprise)
│   └── Volumes (no-op, Podman volume, PVC)
├── Provision platform layer (depends on infra):
│   ├── Qhorus (CDI beans in local-dev, container in local-distributed, K8s pod in enterprise)
│   ├── Eidos (CDI beans, container, K8s pod)
│   └── Engine (CDI beans, container, K8s pod)
└── Health check all dependencies → Phase 2

Phase 2 — SELF-AWARE (ops app activates self-maintenance)
├── OpsAppSpec node in casehub-topology.yaml — adapter reports PRESENT (running seed)
├── SelfMaintenanceMonitor activates:
│   ├── Monitor for new versions (config, registry, or manual trigger)
│   ├── On upgrade: provision replacement via active ApplicationDeploymentTarget
│   ├── Health-check replacement
│   ├── Swap traffic
│   └── Old instance deactivates gracefully
└── Key invariant: the running instance provisions its replacement — it does not kill itself
```

**Tenancy:** The bootstrap reconciliation loop uses `casehub.ops.app.default-tenancy-id` (existing config property, default: `"default"`). This is the bootstrap tenant — all topology nodes are provisioned under this tenancy. Post-bootstrap tenant onboarding is out of scope: each additional tenant would declare its own topology graph registered via the API.

**Phase transition:** The ops app transitions from Phase 1 to Phase 2 when all dependencies report PRESENT from the `ActualStateAdapter`. During Phase 1, the ops-app node exists in the topology but the provisioner treats it as a no-op — the seed instance is already running, so the adapter reports PRESENT. Phase 2 activates the self-maintenance monitor. This transition is idempotent — if the ops app restarts during Phase 1, it re-probes and continues.

**Self-maintenance mechanisms per mode:**

| Mode | Mechanism | Handoff |
|------|-----------|---------|
| `local-dev` | Not applicable — developer restarts manually | N/A |
| `local-distributed` | Old instance provisions new container via `PodmanDeploymentTarget.deploy()` with NO host port mapping (avoids port conflict with running instance). Health-check via Podman internal network DNS (container name `casehub-ops-next`). After health-check passes, old instance exits gracefully — container stops as a natural consequence of process exit. New instance detects old stopped container via `PodmanClient`, removes it, then remaps its own host port. | Container name convention: old=`casehub-ops`, new=`casehub-ops-next`. After swap, new container is renamed to `casehub-ops`. |
| `enterprise` | Ops app patches its own K8s Deployment spec via fabric8 client to use the new image. K8s Deployment controller performs a standard rolling update — starts new pod, readiness-gates it, drains old pod. | K8s manages pod replacement, traffic switching, and old pod cleanup. The ops app initiates but does not execute the swap. |

**Self-maintenance failure modes** (detailed protocol deferred to #123):
- **In-flight reconciliation:** The reconciliation loop is idempotent by design — re-running after restart converges to the same state. No state handoff needed.
- **Version compatibility:** Forward compatibility required — new version must read topology YAML and database state from old version. Guaranteed by additive-only Flyway migrations.
- **Rollback:** If health checks fail, old instance remains active (new container/pod is removed). If health checks pass but behavior is incorrect, manual rollback is required (tracked in #123).

**Relationship to Quarkus profiles:** The CaseHub deployment profile (`casehub.ops.deployment-profile`) is orthogonal to Quarkus profiles. Existing `%dev` and `%test` Quarkus profiles continue to control framework behavior (H2 datasource, Hibernate schema management). No new Quarkus profiles (`%local-dev`, `%local-distributed`, `%enterprise`) are introduced.

| Deployment profile | Quarkus profile | Datasource |
|-------------------|----------------|------------|
| `local-dev` | `dev` | H2 (from existing `%dev` config) |
| `local-distributed` | default (prod) | PostgreSQL container (provisioned by topology) |
| `enterprise` | default (prod) | Managed PostgreSQL (production) |

### Topology YAML Examples

**casehub-topology.yaml** (same across all profiles — all `casehub_application` nodes in one graph):
```yaml
desiredState:
  namespace: casehub
  name: casehub-topology

nodes:
  qhorus:
    type: casehub_application
    spec:
      applicationId: qhorus
      image: casehubio/qhorus:0.5-SNAPSHOT
      replicas: 1
      config:
        quarkus.http.port: "8081"
      # Override example: force container even in local-dev
      # deploymentTarget: podman

  eidos:
    type: casehub_application
    spec:
      applicationId: eidos
      image: casehubio/eidos:0.3-SNAPSHOT
      replicas: 1

  engine:
    type: casehub_application
    spec:
      applicationId: engine
      image: casehubio/engine:0.4-SNAPSHOT
      replicas: 1

  ops-app:
    type: casehub_application
    dependsOn: [qhorus, eidos, engine]
    spec:
      applicationId: ops-app
      image: casehubio/casehub-ops-app:latest
      replicas: 1
      config:
        casehub.ops.topology.path: /etc/casehub/topology
```

**casehub-infra-local-distributed.yaml** (profile-specific infra module):
```yaml
module:
  name: casehub-infra-local-distributed
  parameters: {}

nodes:
  ops-postgres:
    type: container:database
    spec:
      name: ops-postgres
      image: "postgres:16"
      databaseName: casehub_ops
      username: casehub
      password: casehub
      dataVolume: ops-postgres-data

  ops-network:
    type: network
    spec:
      name: casehub-net
```

**Per-node override example (use existing Qhorus in enterprise):**
```yaml
  qhorus:
    type: casehub_application
    spec:
      applicationId: qhorus
      image: casehubio/qhorus:0.5
      existing: https://qhorus.internal:8443
```

Note: `existing` and `deploymentTarget` live inside the `spec:` block, following the infra domain's `backendId` pattern — `InfraWrappingFactory` extracts `backendId` from the spec map before converting to the typed record. The `DeploymentNodeSpecFactoryProvider` does the same for deployment routing fields. This avoids cross-repo `YamlNode` schema changes in `casehub-desiredstate-yaml`.

### Module Structure Changes

| Module | Changes |
|--------|---------|
| `api/` | `ApplicationDeploymentTarget` extended with `default String targetId()` method. `DeploymentApplicationNodeSpec` wrapper record (mirrors `InfraDesiredNodeSpec`). No changes to existing `NoOpDeploymentTarget` (inherits default). |
| `deployment/` | `DeploymentNodeSpecFactoryProvider` — registers `casehub_application` with `NodeSpecFactory` SPI, extracts `deploymentTarget`/`existing` from spec map (follows `InfraWrappingFactory` pattern). `InProcessDeploymentTarget`, `PodmanDeploymentTarget` implementations. `CaseHubAppProvisionHandler` updated to inject `Instance<ApplicationDeploymentTarget>` and resolve by `targetId()`. |
| `container/` | No changes — existing `PodmanNodeProvisioner` handles `DatabaseContainerSpec` |
| `infra/` | No changes — existing `InfraNodeProvisioner` handles `DatabaseClusterSpec` |
| `service/` | `BootstrapDomainRegistrar` — wires `CrossDomainCompositionEngine` with domain registrations (container→deployment ordering) at startup. Bootstrap phase coordinator, self-maintenance monitor. |
| `app/` | `KubernetesDeploymentTarget`, deployment profile configuration, topology YAML classpath resources |
| `testing/` | `StubDeploymentTarget` unchanged (inherits default `targetId()`). Test fixtures for deployment target stubs across all three profiles. |

### Testing Strategy

Three-level verification:

1. **Unit tests** — each `ApplicationDeploymentTarget` in isolation. `InProcessDeploymentTarget` with mock CDI, `PodmanDeploymentTarget` with mock `PodmanClient`, `KubernetesDeploymentTarget` with mock fabric8.
2. **Integration tests** — full bootstrap sequence per profile. `@QuarkusTest` with profile-specific config. Verify: topology compiles, reconciliation provisions all nodes, actual-state adapter reports PRESENT, self-maintenance registers ops app.
3. **Profile transition tests** — verify the same platform service nodes produce correct behavior for all three profiles. Same `CaseHubApplicationSpec` instances, different `ApplicationDeploymentTarget` implementations. Assert: node IDs match, dependency edges match, only deployment target behavior differs.

**"Use existing" testing:** Mock an external service (e.g. WireMock for Qhorus health endpoint). Verify adapter reports PRESENT, provisioner skips. Stop WireMock. Verify adapter reports ABSENT, provisioner provisions.

### Configuration

```properties
# Deployment profile (CaseHub-level, selects ApplicationDeploymentTarget)
casehub.ops.deployment-profile=local-dev  # or local-distributed, enterprise

# Topology location (classpath default, filesystem override)
casehub.ops.topology.path=  # empty = classpath default

# Service connection overrides (for "use existing" scenarios)
casehub.ops.platform.postgres.url=jdbc:postgresql://localhost:5432/casehub_ops
casehub.ops.platform.qhorus.url=http://localhost:8081
casehub.ops.platform.eidos.url=http://localhost:8082

# Self-maintenance
casehub.ops.self-maintenance.enabled=true
casehub.ops.self-maintenance.health-check-timeout=30s
```

**Relationship to Quarkus profiles:** No new Quarkus profiles are introduced. The CaseHub deployment profile (`casehub.ops.deployment-profile`) is orthogonal to existing `%dev`/`%test`/`%prod` Quarkus profiles. For `local-dev`, the application runs with `quarkus.profile=dev` (existing H2 config). For `local-distributed` and `enterprise`, the default Quarkus profile provides the PostgreSQL datasource configuration.

## Scope Boundaries

**In scope:**
- Deployment profile abstraction via `casehub.ops.deployment-profile` config property
- Three `ApplicationDeploymentTarget` implementations (in-process, podman, kubernetes)
- Two-phase bootstrap with phase transition and tenancy model
- Self-maintenance monitor (mechanism-level: provision replacement, health-check, swap)
- Topology YAML using standard desiredstate `desiredState:` format at `META-INF/desiredstate/`
- Profile-specific infrastructure modules at `META-INF/desiredstate/modules/`
- Health probes as `ActualStateAdapter` implementation details
- Test strategy across all three profiles

**Out of scope (tracked as GitHub issues):**
- Self-maintenance detailed protocol — failure modes, state handoff, rollback (#123)
- Git-backed topology versioning (#124)
- Real EvidenceCollector implementations (compliance domain concern)
- Multi-region/HA for the ops app itself (#125)
- CI/CD pipeline integration for the enterprise profile (#126)
- UI for topology visualization

## References

- **#118 (Extract ops/service module)** — architectural prerequisite. The in-process deployment target requires platform services as embeddable libraries, which is only possible after #118 extracts service code from the Quarkus shell into a plain JAR library.
- **#83 (CaseHub Application NodeSpec + Provisioner)** — establishes `CaseHubApplicationSpec`, `ApplicationBuildPipeline`, and `ApplicationDeploymentTarget` SPIs. Decision D3: *"A DeploymentTarget SPI allows PodmanDeploymentTarget now with K8s and cloud run later, matching the multi-target vision from #117."* This spec implements that vision with three deployment target implementations.
- `infra/src/main/resources/META-INF/desiredstate/modules/` — existing module YAML pattern (e.g. `load-balancer.yaml`)
- ARC42STORIES.MD §4 — CrossDomainCompositionEngine, module composition, DomainRegistration provides/requires
- ARC42STORIES.MD §8 — CaseHub Application Node Type, ApplicationBuildPipeline and ApplicationDeploymentTarget SPIs
- ARC42STORIES.MD §Layer L2 — InfraBackend discovery pattern: all backends `@ApplicationScoped`, coexist, resolved by `backendId()` via `@Any Instance<InfraBackend>`. ApplicationDeploymentTarget follows the same pattern with `targetId()`.
- `container/src/main/java/io/casehub/ops/container/` — existing Podman provisioning (DatabaseContainerSpec, PodmanNodeProvisioner)
- `api/src/main/java/io/casehub/ops/api/deployment/CaseHubApplicationSpec.java` — platform service node type
- `api/src/main/java/io/casehub/ops/api/deployment/ApplicationDeploymentTarget.java` — deployment target SPI
- `deployment/src/main/java/io/casehub/ops/deployment/handler/CaseHubAppProvisionHandler.java` — orchestrates build pipeline → deploy target → health check
- `app/src/main/resources/application.properties` — existing Quarkus profile config with `%dev`/`%test`
- Kubernetes kubelet self-hosting model — architectural precedent for two-phase bootstrap
- casehubio/casehub-ops#117 — parent issue
