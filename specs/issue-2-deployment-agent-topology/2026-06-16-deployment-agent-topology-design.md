# Deployment Module Design — CaseHub Agent Topology

**Issue:** casehubio/casehub-ops#2
**Date:** 2026-06-16
**Status:** Approved

## Problem

A CaseHub application's topology — agents, channels, case types, trust policies — is configured through scattered mechanisms: `application.properties` for agents, initializer beans for channels, classpath YAML for case definitions, CDI beans for trust policies. There is no single declaration, no drift detection, no self-healing, and no audit trail.

## Solution

A deployment module in casehub-ops that implements the casehub-desiredstate SPIs (`GoalCompiler`, `ActualStateAdapter`, `NodeProvisioner`, `FaultPolicy`, `EventSource`) for CaseHub's own topology. A single `casehub-deployment.yaml` declares what should exist. The desiredstate reconciliation loop ensures it does.

## Scope

This spec covers **registration-level provisioning** — ensuring the right descriptors, channels, definitions, and policies are registered in the right foundation modules. Provider-specific configuration (claudony session settings, openclaw gateway config, case definition file loading, connector integration) is tracked in casehubio/casehub-ops#7.

## Key Design Decisions

**No inherent dependency edges.** Agents, channels, case types, and trust policies are independently creatable in the CaseHub platform. `AgentDescriptor` does not reference channels. `CaseDefinition` does not reference agents by ID. Channels do not reference agents. Binding happens at runtime via routing policies, capability matching, and channel dispatch. The deployment graph is flat by default — edges only exist for explicit user-declared `dependsOn` relationships.

**No backend abstraction.** The infra module routes provisioning through backends (terraform, ansible, standalone) because the same resource type can be provisioned by different tools. In the deployment domain, each node type has exactly one provisioning target: agents → eidos, channels → qhorus, case types → engine, trust → internal policy store. The provisioner dispatches by node type, not by backend.

**Topology is layout, not interactions.** The deployment module declares what exists — not how agents communicate, how cases route work, or how sessions start. Process lifecycle (claudony tmux sessions, openclaw gateways) is managed by the existing worker provisioners at case task time. The deployment module manages the declarations that make the platform topology correct.

**YAML schema derived from target types.** Every field in the deployment YAML maps 1:1 to a field on the provisioning target (`AgentDescriptor`, `ChannelCreateRequest`, `CaseDefinition`, `TrustRoutingPolicy`). No fictional properties.

## YAML Schema

```yaml
agents:
  - agentId: code-reviewer           # AgentDescriptor.agentId (required)
    name: Code Reviewer               # AgentDescriptor.name (required)
    slot: worker                       # AgentDescriptor.slot (required)
    provider: anthropic                # AgentDescriptor.provider
    modelFamily: claude                # AgentDescriptor.modelFamily
    modelVersion: opus-4               # AgentDescriptor.modelVersion
    version: "1.0"                     # AgentDescriptor.version
    weightsFingerprint: null           # AgentDescriptor.weightsFingerprint
    jurisdiction: EU                   # AgentDescriptor.jurisdiction
    dataHandlingPolicy: gdpr-compliant # AgentDescriptor.dataHandlingPolicy
    domainVocabulary: null             # AgentDescriptor.domainVocabulary
    slotVocabulary: null               # AgentDescriptor.slotVocabulary
    dispositionVocabulary: null        # AgentDescriptor.dispositionVocabulary
    capabilities:                      # List<AgentCapability>
      - name: code-review             #   AgentCapability.name (required)
        qualityHint: 0.9              #   AgentCapability.qualityHint
        latencyHintP50Ms: 5000        #   AgentCapability.latencyHintP50Ms
        costHint: per-token            #   AgentCapability.costHint
        inputTypes: [text, diff]       #   AgentCapability.inputTypes
        outputTypes: [text, json]      #   AgentCapability.outputTypes
        tags: [java, quarkus]          #   AgentCapability.tags
        epistemicDomains:              #   AgentCapability.epistemicDomains
          software-engineering: 0.9
      - name: security-audit
    disposition:                       # AgentDisposition
      socialOrient: collaborative      #   open vocabulary string
      ruleFollowing: strict
      riskAppetite: conservative
      autonomy: guided
      conflictMode: accommodate
      delegation: false                #   boolean
    dependsOn: []                      # explicit dependency edges (optional)

channels:
  - name: dev/work                     # ChannelCreateRequest.name (required, slug-validated)
    description: Development work      # ChannelCreateRequest.description
    semantic: APPEND                   # ChannelCreateRequest.semantic (required)
    allowedTypes: [COMMAND, RESPONSE, DONE]  # Set<MessageType>
    deniedTypes: []                    # Set<MessageType>
    allowedWriters: "capability:code-review" # comma-separated patterns
    adminInstances: null               # comma-separated instance IDs
    barrierContributors: null          # comma-separated agent IDs (BARRIER semantic)
    rateLimitPerChannel: 100           # max messages/min across all senders
    rateLimitPerInstance: 10           # max messages/min per sender
    inboundConnectorId: null           # connector binding (all-or-nothing)
    externalKey: null
    outboundConnectorId: null
    outboundDestination: null
    dependsOn: []

caseTypes:
  - namespace: io.casehub.ops          # CaseDefinition.namespace (required)
    name: pr-review                    # CaseDefinition.name (required)
    version: "1.0"                     # CaseDefinition.version (required)
    title: Pull Request Review         # CaseDefinition.title
    summary: Automated PR review       # CaseDefinition.summary
    dependsOn: []

trust:
  - capability: code-review            # key for TrustRoutingPolicyProvider
    threshold: 0.7                     # TrustRoutingPolicy.threshold
    minimumObservations: 10            # TrustRoutingPolicy.minimumObservations
    borderlineMargin: 0.1             # TrustRoutingPolicy.borderlineMargin
    blendFactor: 0.6                   # TrustRoutingPolicy.blendFactor
    bootstrapEscalationRequired: false # TrustRoutingPolicy.bootstrapEscalationRequired
    qualityFloors:                     # Map<String, Double>
      accuracy: 0.8
```

ChannelSemantic values: `APPEND`, `COLLECT`, `BARRIER`, `EPHEMERAL`, `LAST_WRITE`.

MessageType values: `QUERY`, `COMMAND`, `RESPONSE`, `STATUS`, `DECLINE`, `HANDOFF`, `DONE`, `FAILURE`, `EVENT`.

## Architecture

### Node type hierarchy (ops-api)

```
DeploymentNodeSpec (sealed interface)
├── AgentNodeSpec (record)         — all AgentDescriptor fields except tenancyId
├── ChannelNodeSpec (record)       — all ChannelCreateRequest fields
├── CaseTypeNodeSpec (record)      — CaseDefinition identity fields
└── TrustPolicyNodeSpec (record)   — TrustRoutingPolicy fields + capability key

DeploymentDesiredNodeSpec (record, implements NodeSpec)
└── wraps DeploymentNodeSpec
```

`AgentNodeSpec` directly holds `List<AgentCapability>` and `AgentDisposition` from eidos-api. No mirroring layer.

### Goal declaration (ops-api)

```
DeploymentGoals (record)
├── agents: List<AgentDeclaration>
├── channels: List<ChannelDeclaration>
├── caseTypes: List<CaseTypeDeclaration>
└── trust: List<TrustPolicyDeclaration>
```

Each declaration record mirrors the YAML structure. The compiler maps declarations to typed node specs.

### GoalCompiler (deployment module)

`DeploymentGoalCompiler implements GoalCompiler<DeploymentGoals>`:
- Iterates each section, maps declarations to `DeploymentNodeSpec`, wraps in `DeploymentDesiredNodeSpec`
- Creates `DesiredNode(NodeId.of(id), NodeType.of(type), wrapper, false)` for each
- Collects explicit `dependsOn` edges as `Dependency` instances
- Returns `factory.of(nodes, dependencies)`

`requiresHuman` is `false` for all deployment node types.

### ActualStateAdapter (deployment module)

`DeploymentActualStateAdapter implements ActualStateAdapter`:
- Injects `AgentRegistry` (eidos), `ReactiveChannelStore` (qhorus), `CaseDefinitionRegistry` (engine)
- Per node type:
  - **Agent**: `agentRegistry.findById(agentId, tenancyId)` → PRESENT / ABSENT / DRIFTED (capabilities mismatch)
  - **Channel**: `channelStore.findByName(name)` → PRESENT / ABSENT / DRIFTED (semantic or allowedTypes mismatch)
  - **Case type**: query by namespace/name/version → PRESENT / ABSENT / DRIFTED (version mismatch)
  - **Trust policy**: always PRESENT (we own the policy data via `DeploymentTrustRoutingPolicyProvider`)

Drift detection compares identity + key structural fields for this pass. Full field-by-field comparison in #7.

### NodeProvisioner and handlers (deployment module)

`DeploymentNodeProvisioner implements NodeProvisioner`:
- Dispatches via exhaustive `switch` on `DeploymentNodeSpec` sealed permits
- Delegates to four `@ApplicationScoped` handler classes

Handlers:

| Handler | Foundation API | Provision action | Deprovision action |
|---------|---------------|-----------------|-------------------|
| `AgentProvisionHandler` | `AgentRegistry` (eidos) | Build `AgentDescriptor` from spec + `tenancyId` from context, call `register()` | Deregister from eidos |
| `ChannelProvisionHandler` | `ChannelService` (qhorus) | Build `ChannelCreateRequest` from spec, call `create()` | Remove channel |
| `CaseTypeProvisionHandler` | `CaseDefinitionRegistry` (engine) | Build `CaseDefinition` via builder, call `registerCaseDefinition()` | Unregister definition |
| `TrustPolicyProvisionHandler` | Internal policy map | Store `TrustRoutingPolicy` keyed by capability | Remove from map (reverts to DEFAULT) |

`TrustPolicyProvisionHandler` feeds `DeploymentTrustRoutingPolicyProvider`, which implements `TrustRoutingPolicyProvider` and serves policies from the stored map with `TrustRoutingPolicy.DEFAULT` as fallback.

### FaultPolicy (deployment module)

`DeploymentFaultPolicy implements FaultPolicy` — returns `List.of()`. No graph mutations for this pass. The runtime handles retry and re-provisioning. Graph mutations become relevant in #7 when provider-specific failures may warrant node replacement.

### EventSource (deployment module)

`DeploymentEventSource implements EventSource` — hot `Multi<StateEvent>` with broadcast. Provides `emit(StateEvent)` and `emitDrift(NodeId)`. Not wired to external event sources for this pass — reconciliation loop's periodic polling via `ActualStateAdapter` is the primary drift detection mechanism. Wiring foundation module CDI events to `emit()` is a follow-up concern.

## Package layout

**`api/` (casehub-ops-api):**

```
io.casehub.ops.api.deployment/
├── DeploymentNodeSpec.java
├── AgentNodeSpec.java
├── ChannelNodeSpec.java
├── CaseTypeNodeSpec.java
├── TrustPolicyNodeSpec.java
├── DeploymentDesiredNodeSpec.java
└── goal/
    ├── DeploymentGoals.java
    ├── AgentDeclaration.java
    ├── ChannelDeclaration.java
    ├── CaseTypeDeclaration.java
    └── TrustPolicyDeclaration.java
```

**`deployment/` (casehub-ops-deployment):**

```
io.casehub.ops.deployment/
├── DeploymentGoalCompiler.java
├── DeploymentActualStateAdapter.java
├── DeploymentNodeProvisioner.java
├── DeploymentFaultPolicy.java
├── DeploymentEventSource.java
├── DeploymentTrustRoutingPolicyProvider.java
└── handler/
    ├── AgentProvisionHandler.java
    ├── ChannelProvisionHandler.java
    ├── CaseTypeProvisionHandler.java
    └── TrustPolicyProvisionHandler.java
```

## Testing

All tests are plain JUnit + AssertJ. No `@QuarkusTest`. Foundation APIs stubbed via constructor injection.

| Test class | Covers |
|-----------|--------|
| `DeploymentGoalCompilerTest` | Each node type's compilation, explicit dependsOn edges, empty sections |
| `AgentProvisionHandlerTest` | AgentNodeSpec → AgentDescriptor field mapping, provision + deprovision |
| `ChannelProvisionHandlerTest` | ChannelNodeSpec → ChannelCreateRequest mapping, connector binding all-or-nothing |
| `CaseTypeProvisionHandlerTest` | CaseTypeNodeSpec → CaseDefinition builder mapping |
| `TrustPolicyProvisionHandlerTest` | Policy storage, provider serving, fallback to DEFAULT |
| `DeploymentActualStateAdapterTest` | PRESENT / ABSENT / DRIFTED for each node type |
| `DeploymentNodeProvisionerTest` | Dispatch by node type, error path for wrong spec type |
| `DeploymentLifecycleIntegrationTest` | End-to-end: compile → provision all → read actual → verify all PRESENT |

## Dependencies

Already declared in `deployment/pom.xml`:
- `casehub-desiredstate-api` — SPIs being implemented
- `casehub-eidos-api` — `AgentDescriptor`, `AgentCapability`, `AgentDisposition`, `AgentRegistry`
- `casehub-qhorus-api` — `ChannelSemantic`, `MessageType`, `ChannelDetail`, `ReactiveChannelStore`
- `casehub-engine-api` — `CaseDefinition`, `TrustRoutingPolicy`, `TrustRoutingPolicyProvider`
- `casehub-work-api` — not used in this pass (no work-routing node type)
- `casehub-platform-api` — tenancy context
- `quarkus-arc` — CDI
- `casehub-desiredstate-testing` (test scope)

`ChannelService` and `ChannelCreateRequest` are in `casehub-qhorus` runtime, not api. The deployment module needs a `<scope>provided</scope>` dependency on `casehub-qhorus` runtime to access channel creation. This is safe — the deployment module is a Jandex library activated by classpath presence, and any application using it will already have qhorus runtime on the classpath. Add to `deployment/pom.xml`:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-qhorus</artifactId>
    <scope>provided</scope>
</dependency>
```

Similarly, check whether `AgentRegistry.register()` is in eidos-api or eidos runtime. If runtime, same pattern applies.

## Follow-up

casehubio/casehub-ops#7 — application-level topology provisioning: provider-specific agent config, connector integration, case definition file loading, cross-repo claudony/openclaw integration, full-field drift detection.

## Garden entries noted

Relevant GEs from casehub/garden to keep in context during implementation:
- `GE-20260529-b5723e` — casehub-engine full runtime as compile dep causes 31+ CDI failures (use -api only)
- `GE-20260607-a4d78a` — ChannelSlugValidator: slugs must start with letter, hyphens only
- `GE-20260613-7b7ae1` — ChannelService.create() now requires ChannelCreateRequest (API changed)
- `GE-20260517-5b8e78` — qhorus core services are CDI-injectable
- `GE-20260516-2805b7` — abstract superclasses indexed by Jandex fail deployment
