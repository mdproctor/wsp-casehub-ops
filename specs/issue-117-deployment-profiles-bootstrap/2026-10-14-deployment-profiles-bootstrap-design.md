# Deployment Profiles and Bootstrap Model

**Issue:** casehubio/casehub-ops#117
**Branch:** issue-117-deployment-profiles-bootstrap
**Scale:** L — architecture design, touches service module, app module, container module, topology YAML

## Problem

Ops + desiredstate is designed to deploy and manage a full CaseHub installation from a single YAML topology declaration. But there is no definition of how the system bootstraps from day zero, how different deployment scales are handled, or how the ops app maintains itself after bootstrap.

## Solution Overview

Three deployment profiles representing the same topology at different scales, a two-phase bootstrap model where the ops app is the seed that provisions everything else, and a self-maintenance loop where the ops app manages its own upgrades.

The core principle is **"use existing or deploy if missing"** — the topology YAML declares what should exist. The actual-state adapter probes for existing services. The reconciliation loop only provisions what's absent. Per-node YAML overrides allow explicit control when needed.

### Profiles

| Profile | Quarkus profile | Default backend | Persistence | Use case |
|---------|----------------|-----------------|-------------|----------|
| `local-dev` | `%local-dev` | `in-process` | In-memory / H2 | Developer inner loop. Single `java -jar`, everything in one CDI container. |
| `local-distributed` | `%local-distributed` | `container` | PostgreSQL container | Integration testing. Real network boundaries via Podman containers. |
| `enterprise` | `%enterprise` | `kubernetes` | Production PostgreSQL, TLS | Real deployment. Same YAML, production-grade backends. |

The topology YAML stays structurally identical across profiles. The profile selects:
1. Which provisioner backend is the default (in-process, container, kubernetes)
2. Which persistence implementations are active (CDI alternatives)
3. Connection parameters for external services

Individual nodes can override the default backend: `backend: container` in a node's YAML forces container provisioning even when the profile default is `in-process`.

### Profile as a Spectrum

Profiles are not rigid buckets. `local-dev` can range from "everything in-memory in-process" to "some services persistent" to "some services in separate containers." The profile sets the default; per-node overrides provide the spectrum. A developer might run `local-dev` with qhorus embedded but postgres in a container — the topology YAML supports this naturally.

## Architecture

### Topology YAML Layers

The full system topology is declared as layered desiredstate modules. Each layer is a YAML module with `provides`/`requires` on `NodeType` for cross-layer dependency ordering:

```
META-INF/casehub/topology/
  infra.yaml          # postgres, networking, volumes
  platform.yaml       # qhorus, eidos, engine
  apps.yaml           # domain services, ops app itself
```

The desiredstate module system handles directory-based merging, classpath+filesystem resolution, and `CrossDomainCompositionEngine` provides dependency ordering: apps depend on platform, platform depends on infra.

**Location:** shipped in the ops app JAR at `META-INF/casehub/topology/`. A config property `casehub.ops.topology.path` points to a filesystem directory for installation-specific overrides. Classpath is the default; filesystem takes precedence when specified.

### Platform Service Node Types

New sealed hierarchy in `service/` module:

```java
public sealed interface PlatformServiceSpec extends NodeSpec
    permits PostgresSpec, QhorusSpec, EidosSpec, EngineSpec, OpsAppSpec {
}
```

Each spec carries service-specific configuration:

```java
@NodeTypeId("platform_postgres")
public record PostgresSpec(
    String instanceId,
    String version,
    int port,
    String databaseName,
    Map<String, String> config
) implements PlatformServiceSpec { ... }

@NodeTypeId("platform_qhorus")
public record QhorusSpec(
    String instanceId,
    String version,
    int port,
    Map<String, String> config
) implements PlatformServiceSpec { ... }

@NodeTypeId("platform_eidos")
public record EidosSpec(
    String instanceId,
    String version
) implements PlatformServiceSpec { ... }

@NodeTypeId("platform_engine")
public record EngineSpec(
    String instanceId,
    String version
) implements PlatformServiceSpec { ... }

@NodeTypeId("platform_ops")
public record OpsAppSpec(
    String instanceId,
    String version,
    String image,
    String source,
    Map<String, String> config
) implements PlatformServiceSpec { ... }
```

### Use Existing or Deploy if Missing

The `PlatformActualStateAdapter` probes for existing services before provisioning:

```
For each platform service node in the topology:
  1. Check YAML override: if `existing: <url>`, probe that URL → PRESENT or ABSENT
  2. Check YAML override: if `backend: <type>`, use that backend regardless of probe
  3. Default: probe known endpoints (TCP connect, HTTP /health, JDBC connect)
     - PRESENT → skip provisioning, continue drift monitoring
     - ABSENT → provisioner creates using profile default backend
```

Probe implementations per service type:

| Service | Probe | Success criteria |
|---------|-------|-----------------|
| PostgreSQL | JDBC connection test | Connection established, target database exists |
| Qhorus | HTTP GET `/q/health/ready` | 200 OK |
| Eidos | CDI bean availability (in-process) or HTTP health (container/K8s) | Bean resolvable or 200 OK |
| Engine | CDI bean availability (in-process) or HTTP health | Bean resolvable or 200 OK |

**Drift detection on existing services:** once a service is PRESENT, the adapter continues monitoring it. If a pre-deployed qhorus goes down, the adapter reports ABSENT and the provisioner can respond — either re-provisioning or raising a fault depending on `FaultPolicy`.

### Provisioner Backend Architecture

`PlatformServiceProvisioner` dispatches to backend implementations selected by CDI profile alternatives:

```java
public interface PlatformBackend {
    String backendId();  // "in-process", "container", "kubernetes"
    ProvisionResult provision(PlatformServiceSpec spec, ProvisionContext context);
    DeprovisionResult deprovision(PlatformServiceSpec spec, DeprovisionContext context);
    NodeStatus status(PlatformServiceSpec spec);
}
```

Three implementations:

**InProcessBackend** (`%local-dev` default):
- PostgresSpec → H2 in-memory datasource via CDI alternative
- QhorusSpec → CDI beans already on classpath, verify resolvable
- EidosSpec → CDI beans already on classpath
- EngineSpec → CDI beans already on classpath
- OpsAppSpec → no-op (already running)

**ContainerBackend** (`%local-distributed` default):
- Delegates to existing `PodmanClient` for container lifecycle
- PostgresSpec → postgres container with Flyway migration
- QhorusSpec → qhorus container image, network config
- EidosSpec → eidos container image
- EngineSpec → engine container image
- OpsAppSpec → build image if source specified, pull if image specified, run container

**KubernetesBackend** (`%enterprise` default):
- K8s Deployment + Service per platform service
- Integrates with existing `K8sWatchManager` for drift detection
- TLS, resource limits, health probes from spec config

### Two-Phase Bootstrap

```
Phase 1 — SEED (ops app starts, provisions dependencies)
├── Load topology YAML (classpath default)
├── Start reconciliation loop
├── Provision infra layer:
│   ├── PostgreSQL (H2 in local-dev, container in local-distributed, managed DB in enterprise)
│   ├── Networking (no-op in local-dev, Podman network in local-distributed, K8s namespace in enterprise)
│   └── Volumes (no-op, Podman volume, PVC)
├── Provision platform layer (depends on infra):
│   ├── Qhorus (CDI beans in local-dev, container in local-distributed, K8s pod in enterprise)
│   ├── Eidos (CDI beans, container, K8s pod)
│   └── Engine (CDI beans, container, K8s pod)
└── Health check all dependencies → Phase 2

Phase 2 — SELF-AWARE (ops app registers itself as a managed node)
├── Add OpsAppSpec node to the topology graph
├── SelfMaintenanceProvisioner activates:
│   ├── Monitor for new versions (config, registry, or manual trigger)
│   ├── On upgrade: provision replacement instance
│   ├── Health-check replacement
│   ├── Swap traffic (in-process: restart, container: stop/start, K8s: rolling update)
│   └── Shutdown old instance
└── Key invariant: the running instance never kills itself — it orchestrates its replacement
```

**Phase transition:** The ops app transitions from Phase 1 to Phase 2 when all dependencies report PRESENT from the actual-state adapter. This transition is idempotent — if the ops app restarts during Phase 1, it re-probes and continues.

**Self-maintenance invariant:** The running ops instance provisions a new instance, health-checks it, swaps traffic, then the OLD instance shuts down. The new instance becomes the running instance and inherits the self-maintenance role. In local-dev this is a JVM restart with the new JAR. In containers, the old container is stopped after the new one is healthy. In K8s, it's a standard rolling update.

### Topology YAML Examples

**infra.yaml:**
```yaml
desiredstate:
  module: casehub-infra
  provides: [platform_postgres, network, volume]
  nodes:
    - id: ops-postgres
      type: platform_postgres
      spec:
        instanceId: ops-postgres
        version: "16"
        port: 5432
        databaseName: casehub_ops
    - id: ops-network
      type: network
      spec:
        name: casehub-net
```

**platform.yaml:**
```yaml
desiredstate:
  module: casehub-platform
  requires: [platform_postgres, network]
  provides: [platform_qhorus, platform_eidos, platform_engine]
  nodes:
    - id: qhorus
      type: platform_qhorus
      spec:
        instanceId: qhorus
        version: "0.5-SNAPSHOT"
        port: 8081
      # Override example: force container even in local-dev
      # backend: container
    - id: eidos
      type: platform_eidos
      spec:
        instanceId: eidos
        version: "0.3-SNAPSHOT"
    - id: engine
      type: platform_engine
      spec:
        instanceId: engine
        version: "0.4-SNAPSHOT"
```

**apps.yaml:**
```yaml
desiredstate:
  module: casehub-apps
  requires: [platform_qhorus, platform_eidos, platform_engine]
  nodes:
    - id: ops-app
      type: platform_ops
      spec:
        instanceId: ops-app
        version: "0.1-SNAPSHOT"
        image: casehubio/casehub-ops-app:latest
        config:
          casehub.ops.topology.path: /etc/casehub/topology
```

**Per-node override example (use existing qhorus in enterprise):**
```yaml
    - id: qhorus
      type: platform_qhorus
      existing: https://qhorus.internal:8443
      spec:
        instanceId: qhorus
        version: "0.5"
```

### Module Structure Changes

| Module | Changes |
|--------|---------|
| `api/` | New `io.casehub.ops.api.platform` package: `PlatformServiceSpec` sealed hierarchy |
| `service/` | `PlatformGoalCompiler`, `PlatformActualStateAdapter`, `PlatformServiceProvisioner`, `PlatformBackend` SPI, `InProcessBackend`, health probes |
| `container/` | `ContainerPlatformBackend` (extends `PodmanNodeProvisioner` for platform services) |
| `app/` | `KubernetesPlatformBackend`, profile-specific `application.properties`, bootstrap phase management |
| `testing/` | Test fixtures for platform service stubs across all three profiles |

### Testing Strategy

Three-level verification:

1. **Unit tests** — each provisioner backend in isolation. `InProcessBackend` with mock CDI, `ContainerBackend` with mock `PodmanClient`, `KubernetesBackend` with mock fabric8.
2. **Integration tests** — full bootstrap sequence per profile. `@QuarkusTest` with `@TestProfile(LocalDevProfile.class)` etc. Verify: topology compiles, reconciliation provisions all nodes, actual-state adapter reports PRESENT, self-maintenance registers ops app.
3. **Profile transition tests** — verify the same topology YAML produces correct graphs for all three profiles. Same nodes, different provisioner backends. Assert: node IDs match, dependency edges match, only backend implementations differ.

**"Use existing" testing:** Mock an external service (e.g. WireMock for qhorus health endpoint). Verify adapter reports PRESENT, provisioner skips. Stop WireMock. Verify adapter reports ABSENT, provisioner provisions.

### Configuration

```properties
# Profile selection
quarkus.profile=local-dev  # or local-distributed, enterprise

# Default backend (derived from profile, overridable)
casehub.ops.profile.default-backend=in-process

# Topology location (classpath default, filesystem override)
casehub.ops.topology.path=  # empty = classpath default

# Service connection overrides
casehub.ops.platform.postgres.url=jdbc:postgresql://localhost:5432/casehub_ops
casehub.ops.platform.qhorus.url=http://localhost:8081
casehub.ops.platform.eidos.url=http://localhost:8082

# Self-maintenance
casehub.ops.self-maintenance.enabled=true
casehub.ops.self-maintenance.health-check-timeout=30s
casehub.ops.self-maintenance.swap-strategy=rolling  # or restart, blue-green
```

## Scope Boundaries

**In scope:**
- Profile abstraction with Quarkus config profiles
- PlatformServiceSpec sealed hierarchy and provisioner
- Three backend implementations (in-process, container, kubernetes)
- Two-phase bootstrap with phase transition
- Self-maintenance provisioner (provision replacement, health-check, swap)
- Topology YAML layers with module composition
- "Use existing or deploy if missing" with per-node override
- Health probes per service type
- Test strategy across all three profiles

**Out of scope:**
- Git-backed topology versioning (future enhancement)
- Real EvidenceCollector implementations
- Multi-region/HA for the ops app itself
- CI/CD pipeline integration for the enterprise profile
- UI for topology visualization

## References

- `infra/src/main/resources/META-INF/desiredstate/modules/` — existing module YAML pattern
- ARC42STORIES.MD §4 — CrossDomainCompositionEngine, module composition
- ARC42STORIES.MD §8 — CaseHub Application Node Type, ApplicationBuildPipeline and ApplicationDeploymentTarget SPIs
- `container/src/main/java/io/casehub/ops/container/` — existing Podman provisioning
- `api/src/main/java/io/casehub/ops/api/deployment/CaseHubApplicationSpec.java` — existing casehub_application node type
- `app/src/main/resources/application.properties` — existing Quarkus profile config with %dev/%test
- Kubernetes kubelet self-hosting model — architectural precedent for two-phase bootstrap
- casehubio/casehub-ops#117 — parent issue
