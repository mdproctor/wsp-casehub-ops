# Canonical Deployment Topologies — Design Spec

**Date:** 2026-08-29
**Author:** Mark Proctor + Claude
**Status:** Design — pending review
**Research:** `docs/research/2026-08-29-canonical-deployment-topologies.md`
**Decisions:** `specs/canonical-deployment-topologies/decisions.md`

---

## 1. Goal

Prove that the CaseHub desired-state system can express, compile, reconcile, and
deploy the full range of real-world software deployment architectures — from a single
blog on one node to a multi-region active-passive banking system with GOAP-planned
migrations and nine-dimension operational monitoring.

The deliverable is a topology matrix (5 app architectures × 4 infra topologies) with
~14 meaningful intersections, each verified at three levels: YAML compilation,
reconciliation integration, and live K8s deployment.

The topology exemplars also serve as the primary tutorial material for CaseHub's
YAML-first user engagement strategy.

---

## 2. Architecture

### 2.1 Layered Composition

No new compiler. Topologies are expressed as YAML-native compositions using the
existing `casehub-desiredstate-yaml` primitives:

| Primitive | Topology Role | Example |
|---|---|---|
| **Lifecycle phases** | Ordered rollout stages | "Infra first, then data, then app, then web" |
| **Modules** | Reusable infra patterns | `load-balancer.yaml`, `ha-multi-az.yaml` |
| **forEach** | AZ/region replication | "Stamp this service across 3 availability zones" |
| **Invariants** | Topology constraint enforcement | "Multi-tier must have web→app→data chain" |
| **Rules** | Auto-wiring | "Every deployment gets a ClusterIP service" |
| **when:** | Optional features | "Include mesh only when mesh_enabled" |
| **Variables** | Per-environment config | "replicas: 1 in dev, 3 in prod" |
| **topology:** | Metadata for observability | `topology: multi-tier/ha-multi-az` — queryable by dashboards, GOAP, audit |

### 2.2 Generation Pipeline

Java records are the canonical source of truth for all types:

```
Java records + annotations  →  YAML schema (generated)  →  TS types (generated)
                                     │
                                NodeSpecRegistry (auto-populated via Jandex)
                                     │
                            Topology YAML modules use generated schema
```

YAML is the primary user-facing interface. Java is the escape hatch for developers
who want IDE support and type safety.

### 2.3 Full-Stack Integration

Each layer of the CaseHub stack handles a different timescale:

| Layer | Component | Timescale | Role |
|---|---|---|---|
| Declaration | YAML + YamlGraphRecorder | Build time | WHAT should exist |
| Steady-state | TransitionPlanner | Seconds | Diff-based reconciliation |
| Migration | GoapPlanner | Minutes | Multi-step topology transitions |
| Orchestration | Engine Case + ApprovalEvaluator | Hours | Human gates, audit trail |
| Operations | Service Lifecycle (L6) | Days–months | Nine-dimension monitoring |

---

## 3. The Topology Matrix

### 3.1 Application Architectures

| ID | Architecture | Key Structural Pattern | YAML Primitives Used |
|---|---|---|---|
| A1 | Single Service | One deployment, optional ingress | Basic nodes, variables |
| A2 | Multi-Tier | Linear chain: web → app → data | Lifecycle phases, invariants |
| A3 | Microservices | Mesh of independent services | forEach, rules (auto-service), modules |
| A4 | Event-Driven | Broker as architectural centre | Invariants (broker dependency), modules |
| A5 | Sidecar/Mesh | App + proxy pairs | Rules (sidecar injection), modules |

### 3.2 Infrastructure Topologies

| ID | Topology | Key Infrastructure Pattern | New InfraNodeSpec Types |
|---|---|---|---|
| I1 | Single Node | One namespace, no redundancy | None (existing types sufficient) |
| I2 | Load-Balanced Cluster | LB + ingress in front | `LoadBalancerSpec` |
| I3 | HA Multi-AZ | Per-AZ replication, anti-affinity | `LoadBalancerSpec` (reuse) |
| I4 | Multi-Region A/P | Primary + DR, failover | `DnsFailoverSpec`, `DataReplicationSpec` |

### 3.3 Matrix Intersections

Each intersection uses a recognisable real-world domain:

| | I1: Single Node | I2: LB Cluster | I3: HA Multi-AZ | I4: Multi-Region A/P |
|---|---|---|---|---|
| **A1: Single** | Personal blog (Ghost) | Company website | — | — |
| **A2: Multi-Tier** | Local dev stack | E-commerce storefront | Hospital records | Retail banking core |
| **A3: Microservices** | Local dev env | Food delivery platform | Trading platform | Global payments |
| **A4: Event-Driven** | Local dev env | IoT telemetry pipeline | — | — |
| **A5: Sidecar/Mesh** | — | Logistics fleet tracking | Insurance claims | — |

**14 intersections** split into two tiers (per D11):

**Core test intersections (5)** — full 3-layer verification (compilation + reconciliation + live):
1. Single service on single-node (baseline)
2. Multi-tier on single-node (lifecycle phases, linear deps)
3. Microservices on HA multi-AZ (forEach, modules, service discovery)
4. Event-driven on LB cluster (broker-centric, invariants)
5. Sidecar/mesh on LB cluster (rules, mesh module)

**Tutorial exemplars (9)** — compilation tests + rich documentation. These prove YAML
expressiveness across diverse domains and serve as the primary learning material.

---

## 4. New Java Types (Canonical Layer)

### 4.1 InfraNodeSpec Extensions

New sealed variants added to the `InfraNodeSpec` hierarchy in `casehub-ops-api`:

```java
// Load balancer — used by I2, I3, I4
public record LoadBalancerSpec(
    String name,
    String namespace,
    LoadBalancerType type,          // APPLICATION, NETWORK
    String healthCheckPath,
    int healthCheckPort,
    List<String> targetServices,
    Labels labels
) implements InfraNodeSpec {
    public String resourceType() { return "load_balancer"; }
}

// DNS-based failover — used by I4
public record DnsFailoverSpec(
    String domainName,
    String primaryEndpoint,
    String secondaryEndpoint,
    int ttlSeconds,
    FailoverPolicy policy           // LATENCY, FAILOVER, WEIGHTED
) implements InfraNodeSpec {
    public String resourceType() { return "dns_failover"; }
}

// Cross-region data replication — used by I4
public record DataReplicationSpec(
    String sourceCluster,
    String targetCluster,
    String sourceService,
    ReplicationMode mode,           // ASYNC, SEMI_SYNC
    int lagToleranceSeconds
) implements InfraNodeSpec {
    public String resourceType() { return "data_replication"; }
}

// Service mesh control plane — used by A5
public record ServiceMeshControlPlaneSpec(
    String namespace,
    String image,
    int replicas,
    Labels labels
) implements InfraNodeSpec {
    public String resourceType() { return "mesh_control_plane"; }
}

// Sidecar proxy — used by A5
public record SidecarProxySpec(
    String targetService,
    String image,
    ResourceRequirements resources
) implements InfraNodeSpec {
    public String resourceType() { return "sidecar_proxy"; }
}
```

Each type gets a corresponding handler in `InfraNodeProvisioner`.

### 4.2 Supporting Enums

```java
public enum LoadBalancerType { APPLICATION, NETWORK }
public enum FailoverPolicy { LATENCY, FAILOVER, WEIGHTED }
public enum ReplicationMode { ASYNC, SEMI_SYNC }
```

---

## 5. Topology Modules (YAML Layer)

Pre-built YAML modules that users import into their topology declarations.

### 5.1 `modules/load-balancer.yaml`

```yaml
module:
  name: load-balancer
  parameters:
    target_service:
      type: string
      required: true
    health_check_path:
      type: string
      default: "/health"
    namespace:
      type: string
      required: true
    lb_type:
      type: string
      default: "APPLICATION"

nodes:
  lb:
    type: load_balancer
    spec:
      name: "${var.target_service}-lb"
      namespace: ${var.namespace}
      type: ${var.lb_type}
      healthCheckPath: ${var.health_check_path}
      healthCheckPort: 80
      targetServices: ["${var.target_service}"]

  ingress:
    type: k8s-ingress
    dependsOn: [lb]
    spec:
      namespace: ${var.namespace}
      name: "${var.target_service}-ingress"
      rules:
        - host: "${var.target_service}.example.com"
          path: /
          serviceName: ${var.target_service}
          servicePort: 80

invariants:
  lb-has-target:
    match:
      lb: { type: load_balancer }
    message: "Load balancer must have at least one target service"
```

### 5.2 `modules/ha-multi-az.yaml`

```yaml
module:
  name: ha-multi-az
  parameters:
    namespace:
      type: string
      required: true
    region:
      type: string
      required: true
    zones:
      type: list
      required: true

nodes:
  ha-control-plane:
    type: k8s-deployment
    spec:
      namespace: ${var.namespace}
      name: ha-control-plane
      image: k8s.gcr.io/kube-apiserver:v1.29
      replicas: 3
      labels:
        component: control-plane
        region: ${var.region}
```

### 5.3 `modules/multi-region.yaml`

```yaml
module:
  name: multi-region
  parameters:
    primary_cluster:
      type: string
      required: true
    dr_cluster:
      type: string
      required: true
    domain:
      type: string
      required: true
    replication_mode:
      type: string
      default: "ASYNC"

nodes:
  dns-failover:
    type: dns_failover
    spec:
      domainName: ${var.domain}
      primaryEndpoint: ${var.primary_cluster}
      secondaryEndpoint: ${var.dr_cluster}
      ttlSeconds: 60
      policy: FAILOVER

  data-replication:
    type: data_replication
    spec:
      sourceCluster: ${var.primary_cluster}
      targetCluster: ${var.dr_cluster}
      mode: ${var.replication_mode}
      lagToleranceSeconds: 30
```

### 5.4 `modules/service-mesh.yaml`

```yaml
module:
  name: service-mesh
  parameters:
    namespace:
      type: string
      required: true
    control_plane_image:
      type: string
      default: "istio/pilot:1.20"
    control_plane_replicas:
      type: integer
      default: 3

nodes:
  mesh-control-plane:
    type: mesh_control_plane
    spec:
      namespace: ${var.namespace}
      image: ${var.control_plane_image}
      replicas: ${var.control_plane_replicas}

# Auto-inject sidecar for deployments with mesh-inject label
rules:
  sidecar-injection:
    match:
      app: { type: k8s-deployment }
    notExists:
      proxy: { type: sidecar_proxy, of: app, direction: DEPENDENTS }
    actions:
      - addNode:
          id: "proxy-${match.app.id}"
          type: sidecar_proxy
          spec:
            targetService: "${match.app.id}"
            image: envoyproxy/envoy:v1.28
      - addDependency:
          from: "proxy-${match.app.id}"
          to: "${match.app.id}"

invariants:
  mesh-requires-control-plane:
    match:
      proxy: { type: sidecar_proxy }
    directDep:
      cp: { type: mesh_control_plane, of: proxy, direction: DEPENDENCIES }
    message: "Sidecar proxy deployed without mesh control plane"
```

---

## 6. GOAP Migration Actions (Phase 6)

When a topology type changes (e.g., single-node → HA multi-AZ), the `GoapPlanner`
plans the migration. Actions are defined as `GoapAction` records:

| Action | Preconditions | Effects | Cost |
|---|---|---|---|
| `provision-az-nodes` | `namespace-exists` | `az-nodes-running` | 5.0 |
| `setup-data-replication` | `az-nodes-running` × 2 | `data-replicated` | 8.0 |
| `configure-load-balancer` | `az-nodes-running` | `lb-routing` | 3.0 |
| `configure-dns-failover` | `data-replicated` | `failover-configured` | 3.0 |
| `verify-health` | `lb-routing` | `health-verified` | 1.0 |
| `migrate-traffic` | `health-verified`, soft: `failover-configured` | `traffic-migrated` | 2.0 |
| `decommission-old` | `traffic-migrated` | `old-topology-removed` | 1.0 |

Each migration is an engine case with approval gates:
- `provision-*` → auto-approved (low risk)
- `setup-data-replication` → requires approval (HIGH risk)
- `configure-dns-failover` → requires approval (CRITICAL risk)
- `migrate-traffic` → requires approval (CRITICAL risk)
- `decommission-old` → requires approval (HIGH risk)

---

## 7. Module Structure

### 7.1 New Module: `topology-tests/`

```
topology-tests/
├── pom.xml                            (test-scope only, depends on infra + app + desiredstate-yaml)
├── src/test/resources/
│   ├── modules/                       (reusable topology modules)
│   │   ├── load-balancer.yaml
│   │   ├── ha-multi-az.yaml
│   │   ├── multi-region.yaml
│   │   └── service-mesh.yaml
│   └── topologies/                    (exemplar YAML files)
│       ├── single-service/
│       │   ├── single-node-blog.yaml
│       │   └── lb-cluster-website.yaml
│       ├── multi-tier/
│       │   ├── single-node-dev.yaml
│       │   ├── lb-cluster-ecommerce.yaml
│       │   ├── ha-multi-az-healthcare.yaml
│       │   └── multi-region-banking.yaml
│       ├── microservices/
│       │   ├── single-node-dev.yaml
│       │   ├── lb-cluster-delivery.yaml
│       │   ├── ha-multi-az-trading.yaml
│       │   └── multi-region-payments.yaml
│       ├── event-driven/
│       │   ├── single-node-dev.yaml
│       │   └── lb-cluster-telemetry.yaml
│       └── sidecar-mesh/
│           ├── lb-cluster-logistics.yaml
│           └── ha-multi-az-insurance.yaml
├── src/test/java/.../
│   ├── compilation/                   (default profile — YAML → graph assertions)
│   │   ├── SingleServiceCompilationTest.java
│   │   ├── MultiTierCompilationTest.java
│   │   ├── MicroservicesCompilationTest.java
│   │   ├── EventDrivenCompilationTest.java
│   │   └── SidecarMeshCompilationTest.java
│   ├── reconciliation/               (-Preconciliation profile)
│   │   └── ...IntegrationTest.java
│   └── live/                          (-Pinfra-live profile)
│       └── ...LiveDeploymentTest.java
```

### 7.2 Changes to Existing Modules

| Module | Change |
|---|---|
| `api/` | New sealed variants in `InfraNodeSpec`: `LoadBalancerSpec`, `DnsFailoverSpec`, `DataReplicationSpec`, `ServiceMeshControlPlaneSpec`, `SidecarProxySpec`. New enums. |
| `infra/` | New handlers in `InfraNodeProvisioner` for each new spec type |
| `app/` | No changes (ApplicationGoalCompiler already handles ServiceDefinition → K8s resources) |

---

## 8. Verification Strategy

### 8.1 Compilation Tests (Default)

For each of the 14 YAML exemplars:

```java
@Test
void multiTierEcommerceOnLbCluster_compilesCorrectGraph() {
    var graph = compileTopology("topologies/multi-tier/lb-cluster-ecommerce.yaml");

    // Correct node count: namespace + LB + ingress + 3 deployments + 2 services
    assertThat(graph.nodes()).hasSize(8);

    // Multi-tier invariant holds: web → app → data
    assertDependencyChain(graph, "storefront-nginx", "catalog-api", "product-db");

    // LB node exists and targets web tier
    assertNodeOfType(graph, "load_balancer", lb ->
        assertThat(((LoadBalancerSpec) unwrap(lb.spec())).targetServices())
            .contains("storefront-nginx"));

    // Lifecycle phases: infrastructure → data → application → web
    // (if CompilationResult.lifecycle)
}
```

### 8.2 Reconciliation Tests (`-Preconciliation`)

```java
@Test
void multiTierEcommerce_producesCorrectTransitionPlan() {
    var graph = compileTopology("topologies/multi-tier/lb-cluster-ecommerce.yaml");
    var actual = ActualState.empty();

    var plan = transitionPlanner.plan(graph, actual);

    // All nodes need provisioning (starting from empty)
    assertThat(plan.additions()).hasSize(8);

    // Namespace provisioned before deployments
    assertOrderedBefore(plan, "storefront-namespace", "product-db");
    assertOrderedBefore(plan, "product-db", "catalog-api");
    assertOrderedBefore(plan, "catalog-api", "storefront-nginx");
}
```

### 8.3 Live Deployment Tests (`-Pinfra-live`)

```java
@Test
void multiTierEcommerce_deploysAndServesTraffic() {
    var graph = compileTopology("topologies/multi-tier/lb-cluster-ecommerce.yaml");

    reconciliationLoop.reconcile(graph);
    awaitAllPresent(graph, Duration.ofMinutes(5));

    // nginx responds
    assertHealthy("http://storefront-nginx/health");
    // catalog-api responds
    assertHealthy("http://catalog-api:8080/health");
    // postgres accepts connections
    assertConnectable("product-db", 5432);
}
```

---

## 9. Implementation Phases

| Phase | Deliverable | Depends On | Effort |
|---|---|---|---|
| 1 | InfraNodeSpec extensions + provisioner handlers | Platform generator ready | M |
| 2 | Topology modules (pure YAML) | Phase 1 (types must exist) | S |
| 3 | YAML exemplars + compilation tests | Phase 2 (modules must exist) | M |
| 4 | Reconciliation integration tests | Phase 3 | S |
| 5 | Live K8s deployment tests | Phase 4 | M |
| 6 | GOAP migration actions + engine cases | Phase 3 | L |
| 7 | Service lifecycle integration | Phase 6, Chapter 5 completion | M |

**Phase 1–3** deliver the core value: proof that the YAML frontend can express all
14 topology intersections. **Phase 4–5** prove it works in practice. **Phase 6–7**
prove the full CaseHub stack integration.

---

## 10. Out of Scope

- **Multi-region active-active** — requires CRDT/consensus data architecture, not just infra
- **Multi-cluster coordination** — multi-region active-passive requires cross-cluster reconciliation that doesn't exist yet. Multi-region YAML exemplars and compilation tests are in scope. Live multi-region deployment is Phase 7+ (per R1-05)
- **TransitionPlanner/GOAP coordination mechanism** — migration mode flag suspending TransitionPlanner during GOAP migration. Designed in Phase 6, not Phase 1–5 (per R1-04)
- **Deployment strategies** (blue-green, canary, rolling) — orthogonal to topology; future work
- **Real cloud provider integrations** — live tests use minikube/kind, not AWS/GCP/Azure
- **UI for topology management** — TS types are generated, but no UI screens in this phase
- **Production-grade provisioner implementations** — handlers are functional but not hardened
- **Batch/Job and Serverless architectures** — different deployment model (run-to-completion vs continuous). Future topology expansion (per R1-07)

---

## 11. Success Criteria

1. All 14 YAML exemplars compile to correct DesiredStateGraphs
2. Invariants catch structural violations (remove a tier from multi-tier → build fails)
3. Rules auto-wire correctly (sidecar injection adds proxy nodes)
4. forEach stamps correctly (3 AZ copies per service for HA)
5. Lifecycle phases execute in order (infra before services)
6. Reconciliation produces correct transition plans for all exemplars
7. At least 3 exemplars deploy successfully to real K8s
8. GOAP plans correct migration sequence for single-node → HA transition
9. Topology exemplars serve as readable, self-documenting tutorials

---

## References

- `docs/research/2026-08-29-canonical-deployment-topologies.md` — full research with web sources
- `casehub-desiredstate-yaml/runtime/` — YAML frontend: YamlGraphRecorder, ForEachExpander, ModuleExpander
- `casehub-desiredstate/examples/webapp-yaml/` — tutorials demonstrating YAML primitives
- `casehub-desiredstate/runtime/TransitionPlanner.java` — steady-state reconciliation
- `casehub-engine/api/GoapPlanner.java` — A* goal-oriented action planner
- `casehub-ops/ARC42STORIES.MD` — existing domain module patterns (SPI quad)
- `casehub-ops/api/.../infra/InfraNodeSpec.java` — sealed hierarchy to extend
- `casehub-ops/app/.../goal/ApplicationGoalCompiler.java` — existing service→K8s compilation
