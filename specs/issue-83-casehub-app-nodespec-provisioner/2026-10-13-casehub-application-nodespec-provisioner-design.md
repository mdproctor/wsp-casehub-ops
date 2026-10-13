# CaseHub Application NodeSpec + NodeProvisioner

**Issue:** casehubio/casehub-ops#83
**Parent:** casehubio/engine#1017
**Branch:** issue-83-casehub-app-nodespec-provisioner

## Problem

YAML-first case applications need a deployment mechanism that uses the existing desiredstate infrastructure — reconciliation loop, topology composition, fault policy, human approval gates. No standalone deployer; the deployment module already provides topology composition with load-balancers, HA, service-mesh nodes.

## Solution

Add a `casehub-application` node type to the deployment domain's sealed hierarchy. The provisioner delegates build and deploy to two SPIs:

- **ApplicationBuildPipeline** — source directory → OCI image reference (stubbed until engine#1018/#1019 deliver)
- **ApplicationDeploymentTarget** — image reference → running container (PodmanDeploymentTarget first)

This gives casehub-application nodes full access to the deployment domain's existing capabilities: dependency-ordered provisioning via TransitionPlanner, drift detection via SpecHashStore, approval workflow via ApprovalEvaluator, and fault policy.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  desiredstate runtime                                       │
│  (TransitionPlanner, ReconciliationLoop, FaultPolicy)       │
├─────────────────────────────────────────────────────────────┤
│  ops/deployment                                             │
│  CaseHubAppProvisionHandler                                 │
│    ├── ApplicationBuildPipeline SPI → image ref             │
│    └── ApplicationDeploymentTarget SPI → container lifecycle│
├─────────────────────────────────────────────────────────────┤
│  Topology nodes: load-balancer, HA, service-mesh, etc.      │
│  (compose in same DesiredStateGraph via dependsOn)          │
└─────────────────────────────────────────────────────────────┘
```

## API Types (ops-api)

### CaseHubApplicationSpec

```java
@NodeTypeId("casehub_application")
@JsonIgnoreProperties(ignoreUnknown = true)
public record CaseHubApplicationSpec(
    String applicationId,
    String source,
    String image,
    RuntimeConfig runtime,
    Map<String, Object> config,
    List<String> dependencies,
    int replicas
) implements DeploymentNodeSpec {

    public CaseHubApplicationSpec {
        if (applicationId == null || applicationId.isBlank())
            throw new IllegalArgumentException("applicationId is required");
        if (source == null && image == null)
            throw new IllegalArgumentException("source or image is required");
        config = config != null ? Map.copyOf(config) : Map.of();
        dependencies = dependencies != null ? List.copyOf(dependencies) : List.of();
        if (replicas < 1) replicas = 1;
    }

    @Override
    public String nodeId() { return applicationId; }

    @Override
    public NodeType nodeType() { return NodeType.of("casehub_application"); }
}
```

**Fields:**
- `applicationId` — unique identifier within the deployment graph
- `source` — path to app directory containing casehub.yaml (for build pipeline)
- `image` — pre-built OCI image reference (bypasses build pipeline when set)
- `runtime` — engine version + module list
- `config` — application configuration (LLM provider, database, etc.)
- `dependencies` — optional Maven coordinates for Java extensions
- `replicas` — number of container instances

When `image` is set, the build pipeline is skipped — the provisioner deploys the pre-built image directly. This is the primary path until engine#1018/#1019 deliver the build pipeline.

### RuntimeConfig

```java
public record RuntimeConfig(
    String engine,
    List<String> modules
) {
    public RuntimeConfig {
        if (engine == null || engine.isBlank())
            throw new IllegalArgumentException("engine version is required");
        modules = modules != null ? List.copyOf(modules) : List.of();
    }
}
```

### ApplicationBuildPipeline SPI

```java
public interface ApplicationBuildPipeline {
    BuildResult build(CaseHubApplicationSpec spec);
}
```

```java
public sealed interface BuildResult {
    record Success(String imageRef) implements BuildResult {}
    record Failed(String reason) implements BuildResult {}
}
```

**Stub implementation:** `PrebuiltImagePipeline` — returns `spec.image()` directly. Used until engine#1018/#1019 provide a real build pipeline.

### ApplicationDeploymentTarget SPI

```java
public interface ApplicationDeploymentTarget {
    DeployResult deploy(String imageRef, DeployConfig config);
    DeployResult undeploy(String instanceId);
    TargetStatus status(String instanceId);
    HealthCheckResult healthCheck(String instanceId);
}
```

```java
public record DeployConfig(
    String applicationId,
    int replicas,
    Map<String, Object> config,
    Map<String, String> env
) {}
```

```java
public sealed interface DeployResult {
    record Success(String instanceId) implements DeployResult {}
    record Failed(String reason) implements DeployResult {}
}
```

```java
public sealed interface HealthCheckResult {
    record Healthy(String instanceId) implements HealthCheckResult {}
    record Unhealthy(String instanceId, String reason) implements HealthCheckResult {}
    record Unknown(String instanceId) implements HealthCheckResult {}
}
```

**First implementation:** `PodmanDeploymentTarget` — delegates to `PodmanClient` from the container module. Creates/removes containers, checks container status, verifies engine health via HTTP endpoint.

## Deployment Module Changes

### DeploymentNodeSpec sealed hierarchy

Add `CaseHubApplicationSpec` as the 8th permit:

```java
public sealed interface DeploymentNodeSpec extends NodeSpec permits
    AgentNodeSpec, ChannelNodeSpec, CaseTypeNodeSpec, TrustPolicyNodeSpec,
    EndpointNodeSpec, DetectionNodeSpec, PoolNodeSpec, CaseHubApplicationSpec {
```

### DeploymentGoals

Add `applications` field:

```java
public record DeploymentGoals(
    List<GoalEntry<AgentNodeSpec>> agents,
    List<GoalEntry<ChannelNodeSpec>> channels,
    // ... existing fields ...
    List<GoalEntry<CaseHubApplicationSpec>> applications,
    List<AdaptationRuleSpec> adaptations
) { ... }
```

### DeploymentGoalCompiler

Add `compileEntries(goals.applications(), nodes, dependencies)` alongside existing entries.

### DeploymentNodeProvisioner

Add handler injection and switch cases:

```java
case CaseHubApplicationSpec s -> appHandler.provision(s, context);
```

Register `NodeType.of("casehub_application")` in `handledTypes()`.

### CaseHubAppProvisionHandler

Orchestrates the two SPIs:

1. If `spec.image()` is null → call `buildPipeline.build(spec)`, get image ref
2. If `spec.image()` is set → use it directly
3. Call `deployTarget.deploy(imageRef, deployConfig)`
4. On success → call `deployTarget.healthCheck(instanceId)`
5. Return `ProvisionResult.Success` or `ProvisionResult.Failed`

Deprovision: call `deployTarget.undeploy(instanceId)`.

### CaseHubAppDriftChecker

Implements `NodeDriftChecker<CaseHubApplicationSpec>`:
- Check container status via `deployTarget.status()`
- If container not running → `DRIFTED` (process drift)
- If spec hash changed → `DRIFTED` (config drift, handled by SpecHashStore)
- If health check fails → `DRIFTED` (application drift)

### DeploymentActualStateAdapter

Add casehub_application to the node type dispatch. For running apps, query `deployTarget.status()` → map to `NodeStatus.PRESENT` / `NodeStatus.ABSENT` / `NodeStatus.DRIFTED`.

### Approval Integration

Uses the existing `DeploymentApprovalEvaluator`. Production deployments of casehub-application nodes follow the same risk classification and approval workflow as other deployment nodes.

## YAML Surface

```yaml
nodes:
  - type: casehub-application
    id: incident-app
    spec:
      source: ./apps/incident-response
      runtime:
        engine: "0.2"
        modules: [a2a]
      config:
        llm:
          provider: anthropic
        database:
          type: postgresql
      replicas: 2

  - type: load-balancer
    id: api-lb
    dependsOn: [incident-app]
    spec:
      algorithm: round-robin
```

## Scope Boundaries

**In scope:**
- CaseHubApplicationSpec record + validation
- RuntimeConfig record
- ApplicationBuildPipeline SPI + PrebuiltImagePipeline stub
- ApplicationDeploymentTarget SPI + PodmanDeploymentTarget
- CaseHubAppProvisionHandler (provision/deprovision)
- CaseHubAppDriftChecker
- Wire into DeploymentGoals, GoalCompiler, NodeProvisioner, ActualStateAdapter
- Unit tests for all new types and handlers
- Integration test: compile topology with casehub-application nodes → plan → verify transitions

**Out of scope:**
- Real build pipeline (engine#1018/#1019)
- casehub.yaml descriptor parsing (engine#1019)
- Environment overlay mechanism (separate issue)
- K8s or cloud run deployment targets (future issues)
- Multi-replica orchestration (future — PodmanDeploymentTarget runs single instance)

## Testing

- `CaseHubApplicationSpecTest` — validation, nodeId, nodeType
- `RuntimeConfigTest` — validation
- `PrebuiltImagePipelineTest` — returns image ref from spec
- `CaseHubAppProvisionHandlerTest` — provision/deprovision with mock SPIs
- `CaseHubAppDriftCheckerTest` — container running/stopped/unhealthy scenarios
- `DeploymentGoalCompilerTest` — existing test extended with application entries
- `DeploymentNodeProvisionerTest` — existing test extended with application dispatch
- Integration test: full reconciliation cycle with a casehub-application node

## References

- `api/src/main/java/.../deployment/DeploymentNodeSpec.java` — sealed hierarchy being extended
- `api/src/main/java/.../deployment/AgentNodeSpec.java` — pattern for new NodeSpec records
- `deployment/src/main/java/.../DeploymentGoalCompiler.java` — compiler pattern
- `deployment/src/main/java/.../DeploymentNodeProvisioner.java` — provisioner dispatch pattern
- `deployment/src/main/java/.../handler/AgentProvisionHandler.java` — handler pattern
- `container/src/main/java/.../AppContainerSpec.java` — container module pattern (PodmanDeploymentTarget mirrors this)
- `container/src/main/java/.../podman/PodmanClient.java` — Podman API client (reused by PodmanDeploymentTarget)
- casehubio/engine#1017 — parent epic
- casehubio/engine#1018 — schema-driven record generation (build pipeline dependency)
- casehubio/engine#1019 — casehub.yaml descriptor format (build pipeline dependency)
- casehubio/casehub-ops#117 — deployment profiles (DeploymentTarget aligns with profile concept)
