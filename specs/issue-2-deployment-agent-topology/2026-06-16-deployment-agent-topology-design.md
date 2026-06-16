# Deployment Module Design — CaseHub Agent Topology

**Issue:** casehubio/casehub-ops#2
**Date:** 2026-06-16
**Status:** Approved (revision 4 — post-review)

## Problem

A CaseHub application's topology — agents, channels, case types, trust policies — is configured through scattered mechanisms: `application.properties` for agents, initializer beans for channels, classpath YAML for case definitions, CDI beans for trust policies. There is no single declaration, no drift detection, no self-healing, and no audit trail.

## Solution

A deployment module in casehub-ops that implements the casehub-desiredstate SPIs for CaseHub's own topology. A single `casehub-deployment.yaml` declares what should exist. The desiredstate reconciliation loop ensures it does.

## Scope

This spec covers **registration-level provisioning** — ensuring the right descriptors, channels, definitions, and policies are registered in the right foundation modules. Provider-specific configuration (claudony session settings, openclaw gateway config, case definition file loading, connector integration) is tracked in casehubio/casehub-ops#7.

## Key Design Decisions

**No inherent dependency edges.** Agents, channels, case types, and trust policies are independently creatable. `AgentDescriptor` does not reference channels. `CaseDefinition` does not reference agents by ID. Channels do not reference agents. Binding happens at runtime. The graph is flat by default — edges only exist for explicit user-declared `dependsOn` relationships.

**No backend abstraction.** Each node type has exactly one provisioning target: agents → eidos `AgentRegistry`, channels → qhorus `ChannelService`, case types → engine `CaseDefinitionRegistry`, trust → internal policy store. The provisioner dispatches by node type, not by backend. The infra module's backend layer is justified by multi-tool routing (terraform/ansible/standalone for the same resource type) — the deployment domain has no such routing.

**Single type hierarchy — no declaration/spec mirroring.** The infra module separates goal declarations (YAML parse targets) from node specs (graph nodes) because the compiler does genuine transformation: parsing raw JsonNode → typed specs, resolving backends, validating compatibility. The deployment module's compilation is trivial — wrap and extract dependsOn edges. A parallel declaration hierarchy would duplicate every field with no architectural benefit. `DeploymentNodeSpec` records are both the YAML parse targets and the graph node specs. A generic `GoalEntry<S>` wrapper carries `dependsOn` as compilation metadata.

**No wrapper record.** The infra module wraps `InfraNodeSpec` in `InfraDesiredNodeSpec` to carry `backendId` — routing information consumed by the provisioner. The deployment module has no routing. `DeploymentNodeSpec extends NodeSpec` directly — each concrete record is used as `DesiredNode.spec()` without an intermediate wrapper. One fewer class, zero indirection.

**Topology is layout, not interactions.** The deployment module declares what exists — not how agents communicate, how cases route work, or how sessions start. Process lifecycle is managed by existing worker provisioners at case task time.

**YAML schema derived from target types.** Every field maps 1:1 to a field on the provisioning target (`AgentDescriptor`, `ChannelCreateRequest`, `CaseDefinition`, `TrustRoutingPolicy`).

## Desiredstate SPI Signatures (verified)

The deployment module implements these exact interfaces from `casehub-desiredstate-api`:

```java
// GoalCompiler<G>
DesiredStateGraph compile(G goals, DesiredStateGraphFactory factory);

// NodeProvisioner
ProvisionResult provision(DesiredNode node, ProvisionContext context);
DeprovisionResult deprovision(DesiredNode node, DeprovisionContext context);

// ActualStateAdapter (current — tenancyId SPI change pending, see §Prerequisite)
ActualState readActual(DesiredStateGraph desired);
// After casehubio/casehub-desiredstate#36:
// ActualState readActual(DesiredStateGraph desired, String tenancyId);

// FaultPolicy
List<GraphMutation> onFault(FaultEvent event, DesiredStateGraph current);

// EventSource
Multi<StateEvent> stream();
```

Context records:
- `ProvisionContext(String tenancyId, DesiredStateGraph graph)`
- `DeprovisionContext(String tenancyId, DesiredStateGraph graph)`

Result types (sealed, no other variants):
- `ProvisionResult` → `Success()` | `Failed(String reason)`
- `DeprovisionResult` → `Success()` | `Failed(String reason)`

FaultType enum: `NODE_DESTROYED`, `NODE_DEGRADED`, `PROVISION_FAILED`, `DEPROVISION_FAILED`, `HUMAN_NODE_TIMEOUT`, `DEPENDENCY_UNAVAILABLE`

A `ReactiveNodeProvisioner` (returns `Uni<T>`) also exists in the API but the deployment module implements the blocking `NodeProvisioner`, consistent with the infra module. `CaseDefinitionRegistry.registerCaseDefinition()` returns `Uni<CaseMetaModel>` — the handler bridges via `await().indefinitely()`.

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
    axisVocabularies:                  # AgentDescriptor.axisVocabularies (Map<DispositionAxis, String>)
      SOCIAL_ORIENTATION: "urn:casehub:vocab:social"
      AUTONOMY: "urn:casehub:vocab:autonomy"
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

NodeType strings: `"agent"`, `"channel"`, `"case_type"`, `"trust_policy"`.

**allowedTypes/deniedTypes YAML mapping:** `ChannelCreateRequest` uses `Set<MessageType>` where `null` means "open" (unrestricted). The YAML mapping:
- Field absent or `[]` → `null` → open (all types permitted)
- `[COMMAND, RESPONSE]` → `Set.of(COMMAND, RESPONSE)` → only these types

`ChannelNodeSpec` uses `Set<MessageType>` (nullable). Jackson absent → null, `[]` → empty Set → mapped to null by the compiler.

**Platform limitation:** `ChannelService.setTypeConstraints()` normalises `null` → `Set.of()` before processing, and `MessageType.serializeTypes(Set.of())` returns `null`. So the "nothing permitted" semantic (`Set.of()`) cannot be distinguished from "open" (`null`) after storage. Both YAML absent and `[]` result in an open channel. If a "nothing permitted" semantic is genuinely needed, that requires a qhorus change — not a deployment module concern.

**CaseDefinition scope note:** Only identity + metadata fields (namespace, name, version, title, summary) are populated in this pass. `CaseDefinition` is a mutable class with many additional fields (capabilities, workers, bindings, milestones, goals, completion, semanticData, episodicMemoryConfig, panelNames) — full population via `definitionFile` loading is a #7 concern. The handler maps CaseTypeNodeSpec to `CaseDefinition.builder().namespace(...).name(...).version(...).title(...).summary(...).build()`.

## Architecture

### Node type hierarchy (ops-api)

```java
public sealed interface DeploymentNodeSpec extends NodeSpec permits
    AgentNodeSpec, ChannelNodeSpec, CaseTypeNodeSpec, TrustPolicyNodeSpec {
    String nodeId();
    String nodeType();
}
```

`DeploymentNodeSpec` directly extends `NodeSpec` (the desiredstate marker interface). No intermediate wrapper. Each concrete record implements `nodeId()` returning its natural identifier and `nodeType()` returning its type string.

| Record | nodeId() | nodeType() | Target type |
|--------|----------|-----------|-------------|
| `AgentNodeSpec` | `agentId` | `"agent"` | `AgentDescriptor` |
| `ChannelNodeSpec` | `name` | `"channel"` | `ChannelCreateRequest` |
| `CaseTypeNodeSpec` | `namespace + ":" + name + ":" + version` | `"case_type"` | `CaseDefinition` |
| `TrustPolicyNodeSpec` | `capability` | `"trust_policy"` | `TrustRoutingPolicy` |

`AgentNodeSpec` directly holds `List<AgentCapability>` and `AgentDisposition` from eidos-api. No mirroring types.

### Goal declaration (ops-api)

```java
public record GoalEntry<S extends DeploymentNodeSpec>(S spec, List<String> dependsOn) {
    public GoalEntry {
        dependsOn = dependsOn != null ? List.copyOf(dependsOn) : List.of();
    }
}

public record DeploymentGoals(
    List<GoalEntry<AgentNodeSpec>> agents,
    List<GoalEntry<ChannelNodeSpec>> channels,
    List<GoalEntry<CaseTypeNodeSpec>> caseTypes,
    List<GoalEntry<TrustPolicyNodeSpec>> trust
) {}
```

`GoalEntry` carries `dependsOn` as compilation metadata — the only field consumed by the compiler but irrelevant to provisioning. The spec itself is used directly as `DesiredNode.spec()`.

### GoalCompiler (deployment module)

`DeploymentGoalCompiler implements GoalCompiler<DeploymentGoals>`:

```java
@ApplicationScoped
public class DeploymentGoalCompiler implements GoalCompiler<DeploymentGoals> {

    @Override
    public DesiredStateGraph compile(DeploymentGoals goals, DesiredStateGraphFactory factory) {
        List<DesiredNode> nodes = new ArrayList<>();
        List<Dependency> dependencies = new ArrayList<>();

        compileEntries(goals.agents(), nodes, dependencies);
        compileEntries(goals.channels(), nodes, dependencies);
        compileEntries(goals.caseTypes(), nodes, dependencies);
        compileEntries(goals.trust(), nodes, dependencies);

        return factory.of(nodes, dependencies);
    }

    private <S extends DeploymentNodeSpec> void compileEntries(
            List<GoalEntry<S>> entries, List<DesiredNode> nodes, List<Dependency> deps) {
        for (var entry : entries) {
            var spec = entry.spec();
            nodes.add(new DesiredNode(
                NodeId.of(spec.nodeId()), NodeType.of(spec.nodeType()), spec, false));
            for (String dep : entry.dependsOn()) {
                deps.add(new Dependency(NodeId.of(spec.nodeId()), NodeId.of(dep)));
            }
        }
    }
}
```

`requiresHuman` is `false` for all deployment node types.

### Prerequisite: tenancyId SPI change (casehubio/casehub-desiredstate#36)

The `ActualStateAdapter.readActual(DesiredStateGraph)` SPI lacks tenancyId. The deployment module needs it to query tenant-scoped foundation APIs. The same gap exists in `TransitionExecutor.execute(TransitionPlan)` — `SimpleTransitionExecutor` hardcodes `DEFAULT_TENANCY = "default"`.

The `ReconciliationLoop.TenantLoop` holds the correct tenancyId but cannot pass it through either SPI. Verified in the runtime code: `TenantLoop.reconcile()` calls `actualStateAdapter.readActual(desired)` with no tenant context, and `executor.execute(plan)` with no tenant context.

**Fix (tracked in casehubio/casehub-desiredstate#36):** Add `String tenancyId` to both:
- `ActualState readActual(DesiredStateGraph desired, String tenancyId)`
- `Uni<TransitionResult> execute(TransitionPlan plan, String tenancyId)`

This SPI change must land before or alongside the deployment module implementation. All call-site updates are mechanical. Per platform design principles: this is the right design, not a workaround.

### ActualStateAdapter (deployment module)

`DeploymentActualStateAdapter implements ActualStateAdapter`:
- Injects `AgentRegistry` (from eidos-api — blocking, takes explicit tenancyId)
- Injects `ChannelService` (from qhorus-runtime — blocking, tenant-scoped via `CurrentPrincipal` internally)
- Trust and case type state: tracked in deployment module's own internal maps

After the SPI change (casehubio/casehub-desiredstate#36), tenancyId flows from the reconciliation loop through `readActual(desired, tenancyId)`. For agents, this passes directly to `AgentRegistry.findById(agentId, tenancyId)`. For channels, `ChannelService.findByName()` resolves tenancy via `CurrentPrincipal` — the reconciliation loop will need to set up a tenant context before calling readActual (same mechanism it uses for other tenant-scoped CDI calls).

Per node type:
- **Agent**: `agentRegistry.findById(agentId, tenancyId)` → PRESENT / ABSENT. If present, compares capabilities list for DRIFTED.
- **Channel**: `channelService.findByName(name)` → returns `Optional<Channel>` (the JPA entity, not `ChannelDetail`). PRESENT / ABSENT. **Drift detection compares mutable fields only:** `allowedTypes`, `deniedTypes`, `rateLimitPerChannel`, `rateLimitPerInstance`, `allowedWriters`, `adminInstances`. For `allowedTypes`/`deniedTypes`: `Channel` stores these as comma-separated strings — parse via `MessageType.parseTypes(channel.allowedTypes)` before comparing against the `Set<MessageType>` in `ChannelNodeSpec`. **Immutable fields are excluded from drift detection:** `semantic`, `description`, `barrierContributors`, connector bindings have no setter methods on `ChannelService`. Detecting drift on these would create an unresolvable reconciliation loop (DRIFTED → re-provision → handler can't update → still DRIFTED). Changing an immutable field requires manual deprovision + provision (delete and recreate the channel). Automated immutable-field drift resolution is a #7 concern requiring reconciliation model changes.
- **Case type**: **Always PRESENT (first-pass simplification).** `CaseDefinitionRegistry.getCaseMetaModel(CaseDefinition)` throws `RuntimeException` on not-found — there is no clean existence query. The deployment module tracks registered case types in its own internal set (same pattern as trust). Real drift detection requires a `findByIdentity()` method on the registry (SPI change to engine-common, tracked for #7).
- **Trust policy**: **Always PRESENT (first-pass simplification).** The deployment module owns the policy data via `DeploymentTrustRoutingPolicyProvider`. External overrides at higher CDI priority would be invisible. Proper trust drift detection is a #7 concern.

Drift detection for agents and channels compares identity + key structural fields. Full field-by-field comparison is a #7 concern.

### NodeProvisioner and handlers (deployment module)

`DeploymentNodeProvisioner implements NodeProvisioner`:

```java
@ApplicationScoped
public class DeploymentNodeProvisioner implements NodeProvisioner {

    private final AgentProvisionHandler agentHandler;
    private final ChannelProvisionHandler channelHandler;
    private final CaseTypeProvisionHandler caseTypeHandler;
    private final TrustPolicyProvisionHandler trustHandler;

    @Inject
    public DeploymentNodeProvisioner(...) { ... }

    @Override
    public ProvisionResult provision(DesiredNode node, ProvisionContext context) {
        if (!(node.spec() instanceof DeploymentNodeSpec spec))
            return new ProvisionResult.Failed("spec is not DeploymentNodeSpec");

        return switch (spec) {
            case AgentNodeSpec s -> agentHandler.provision(s, context);
            case ChannelNodeSpec s -> channelHandler.provision(s, context);
            case CaseTypeNodeSpec s -> caseTypeHandler.provision(s, context);
            case TrustPolicyNodeSpec s -> trustHandler.provision(s, context);
        };
    }

    @Override
    public DeprovisionResult deprovision(DesiredNode node, DeprovisionContext context) {
        if (!(node.spec() instanceof DeploymentNodeSpec spec))
            return new DeprovisionResult.Failed("spec is not DeploymentNodeSpec");

        return switch (spec) {
            case AgentNodeSpec s -> agentHandler.deprovision(s, context);
            case ChannelNodeSpec s -> channelHandler.deprovision(s, context);
            case CaseTypeNodeSpec s -> caseTypeHandler.deprovision(s, context);
            case TrustPolicyNodeSpec s -> trustHandler.deprovision(s, context);
        };
    }
}
```

Handlers:

| Handler | Foundation API | Module | Provision | Deprovision | Idempotency |
|---------|---------------|--------|-----------|-------------|-------------|
| `AgentProvisionHandler` | `AgentRegistry` | eidos-api | Build `AgentDescriptor` from spec + `context.tenancyId()`, call `register()` | Deregister from eidos | **Idempotent.** `register()` is upsert — `ConcurrentHashMap.put()` overwrites existing. Calling with updated capabilities replaces the descriptor. **Known constraint:** `InMemoryAgentRegistry` keys by `agentId` alone (not `(agentId, tenancyId)`) — multi-tenant deployments with overlapping agentIds would overwrite across tenants. `JpaAgentRegistry` presumably uses a compound key. Not a deployment module issue — pre-existing eidos design choice. Single-tenant PoC is unaffected. |
| `ChannelProvisionHandler` | `ChannelService` | qhorus-runtime | `findByName()` first. If absent → `create(ChannelCreateRequest)`. If present → update mutable fields via setters (`setTypeConstraints`, `setRateLimits`, `setAllowedWriters`, `setAdminInstances`). Immutable fields (semantic, description, barrierContributors, connector bindings) are set at creation only — changes require deprovision + provision. | Remove channel | **Not idempotent on create.** `@UniqueConstraint(columnNames = {"tenancy_id", "name"})` throws on duplicate name. Handler must check-then-create. Race between find and create caught by constraint — handle `PersistenceException` as success (channel now exists). |
| `CaseTypeProvisionHandler` | `CaseDefinitionRegistry` | engine-common | Build `CaseDefinition` via builder (namespace, name, version, title, summary only), call `registerCaseDefinition(def).await().indefinitely()` | Unregister definition | **Idempotent.** `registerCaseDefinition()` returns existing metadata if already registered. `Uni<CaseMetaModel>` return — must await. Full case definition population is #7. |
| `TrustPolicyProvisionHandler` | Internal policy map | deployment module | Store `TrustRoutingPolicy` keyed by capability name | Remove from map (reverts to `TrustRoutingPolicy.DEFAULT`) | **Idempotent.** Map put overwrites. |

`DeploymentTrustRoutingPolicyProvider` implements `TrustRoutingPolicyProvider` (from engine-api). Serves policies from the handler's stored map. Falls back to `TrustRoutingPolicy.DEFAULT` for undeclared capabilities.

### FaultPolicy (deployment module)

```java
@ApplicationScoped
public class DeploymentFaultPolicy implements FaultPolicy {
    @Override
    public List<GraphMutation> onFault(FaultEvent event, DesiredStateGraph current) {
        return List.of();
    }
}
```

No graph mutations for this pass. The runtime handles retry and re-provisioning. The `FaultEvent` carries a `FaultType` (one of: `NODE_DESTROYED`, `NODE_DEGRADED`, `PROVISION_FAILED`, `DEPROVISION_FAILED`, `HUMAN_NODE_TIMEOUT`, `DEPENDENCY_UNAVAILABLE`). Graph mutations become relevant in #7 when provider-specific failures may warrant node replacement.

### EventSource (deployment module)

```java
@ApplicationScoped
public class DeploymentEventSource implements EventSource {
    // hot Multi<StateEvent> with broadcast
    // emit(StateEvent) for external callers to push drift signals
    // emitDrift(NodeId) convenience method
}
```

Not wired to external event sources for this pass. Reconciliation loop's periodic polling via `ActualStateAdapter` is the primary drift detection mechanism. Wiring foundation module CDI events (`@Observes` on eidos/qhorus/engine state changes) to `emit()` is a follow-up concern.

## Package Layout

**`api/` (casehub-ops-api):**

```
io.casehub.ops.api.deployment/
├── DeploymentNodeSpec.java          # sealed interface extends NodeSpec
├── AgentNodeSpec.java               # record — all AgentDescriptor fields except tenancyId
├── ChannelNodeSpec.java             # record — all ChannelCreateRequest fields
├── CaseTypeNodeSpec.java            # record — CaseDefinition identity fields
├── TrustPolicyNodeSpec.java         # record — TrustRoutingPolicy fields + capability key
├── GoalEntry.java                   # record<S extends DeploymentNodeSpec>(S spec, List<String> dependsOn)
└── DeploymentGoals.java             # record — top-level goal declaration
```

7 classes total. No Declaration hierarchy, no wrapper record.

**`deployment/` (casehub-ops-deployment):**

```
io.casehub.ops.deployment/
├── DeploymentGoalCompiler.java          # GoalCompiler<DeploymentGoals>
├── DeploymentActualStateAdapter.java    # ActualStateAdapter
├── DeploymentNodeProvisioner.java       # NodeProvisioner (dispatch switch)
├── DeploymentFaultPolicy.java           # FaultPolicy (empty)
├── DeploymentEventSource.java           # EventSource (hot Multi<StateEvent>)
├── DeploymentTrustRoutingPolicyProvider.java  # TrustRoutingPolicyProvider impl
└── handler/
    ├── AgentProvisionHandler.java       # eidos AgentRegistry
    ├── ChannelProvisionHandler.java     # qhorus ChannelService
    ├── CaseTypeProvisionHandler.java    # engine CaseDefinitionRegistry
    └── TrustPolicyProvisionHandler.java # stores policies, feeds provider
```

## Dependencies

**Already in `deployment/pom.xml`:**
- `casehub-desiredstate-api` — SPIs being implemented
- `casehub-eidos-api` — `AgentDescriptor`, `AgentCapability`, `AgentDisposition`, `AgentRegistry`, `AgentQuery`
- `casehub-qhorus-api` — `ChannelSemantic`, `MessageType` (enum types only — store and service are in runtime)
- `casehub-engine-api` — `CaseDefinition`, `TrustRoutingPolicy`, `TrustRoutingPolicyProvider`
- `casehub-work-api` — not used in this pass
- `casehub-platform-api` — `CurrentPrincipal` for tenancy context
- `quarkus-arc` — CDI
- `casehub-desiredstate-testing` (test scope)

**Must add:**

```xml
<!-- ChannelService, ChannelCreateRequest, Channel (JPA entity) -->
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-qhorus</artifactId>
    <scope>provided</scope>
</dependency>

<!-- CaseDefinitionRegistry (io.casehub.engine.common.spi) -->
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-engine-common</artifactId>
    <scope>provided</scope>
</dependency>
```

`provided` scope is correct: the deployment module is a Jandex library activated by classpath presence. Any application consuming it will already have qhorus-runtime and engine-common on its classpath. `casehub-engine-common` does NOT contain the full CDI bean graph that causes the 31+ failure issue (GE-20260529-b5723e — that's `casehub-engine` runtime). engine-common depends on engine-api, mutiny, jackson-databind, quarkus-vertx, and serverlessworkflow.

**Implementation-time risk:** engine-common transitively depends on `serverlessworkflow-impl-core`, which may bring CDI beans from the Serverless Workflow runtime. Verify during implementation. If this causes CDI issues, the fallback is to not depend on engine-common and instead maintain case type state internally (same in-memory tracking pattern as trust policies), making `CaseDefinitionRegistry` a #7 concern when real drift detection is needed.

## Testing

All tests are plain JUnit + AssertJ. No `@QuarkusTest`. Foundation APIs stubbed via constructor injection.

| Test class | Covers |
|-----------|--------|
| `DeploymentGoalCompilerTest` | Each node type's compilation, explicit dependsOn edges, empty sections, GoalEntry wrapping |
| `AgentProvisionHandlerTest` | AgentNodeSpec → AgentDescriptor field mapping (every field including axisVocabularies), provision + deprovision, tenancyId injection from ProvisionContext |
| `ChannelProvisionHandlerTest` | ChannelNodeSpec → ChannelCreateRequest mapping, connector binding all-or-nothing, MessageType set conversion, check-then-create idempotency, findByName→update path for mutable fields, absent/[] both map to null for allowedTypes/deniedTypes, @Transactional interceptor works from non-request thread (scheduler context) |
| `CaseTypeProvisionHandlerTest` | CaseTypeNodeSpec → CaseDefinition builder, Uni bridging via await().indefinitely() |
| `TrustPolicyProvisionHandlerTest` | Policy storage, DeploymentTrustRoutingPolicyProvider serving, fallback to DEFAULT, deprovision reverts |
| `DeploymentActualStateAdapterTest` | PRESENT / ABSENT / DRIFTED for agent (capabilities mismatch), channel (mutable field drift only — allowedTypes string→Set round-trip via MessageType.parseTypes(), rate limits, allowedWriters; immutable fields NOT compared). Case type and trust always PRESENT. |
| `DeploymentNodeProvisionerTest` | Dispatch by sealed type (exhaustive switch), error path for non-DeploymentNodeSpec |
| `DeploymentLifecycleIntegrationTest` | End-to-end: compile goals → provision all → read actual → verify all PRESENT |

## Prerequisites and Follow-ups

**Prerequisite (must land before or alongside implementation):**
- casehubio/casehub-desiredstate#36 — add tenancyId to `ActualStateAdapter.readActual()` and `TransitionExecutor.execute()`. Without this, the deployment module's actual state adapter cannot query tenant-scoped foundation APIs from the reconciliation loop.

**Follow-up:**
- casehubio/casehub-ops#7 — application-level topology provisioning: provider-specific agent config, connector integration, case definition file loading, cross-repo claudony/openclaw integration, full-field drift detection, trust policy drift detection via effective CDI resolution, `CaseDefinitionRegistry.findByIdentity()` for real case type drift detection.

## Garden entries noted

Relevant GEs to keep in context during implementation:
- `GE-20260529-b5723e` — casehub-engine full runtime as compile dep causes 31+ CDI failures (use -api and -common only)
- `GE-20260607-a4d78a` — ChannelSlugValidator: slugs must start with letter, hyphens only
- `GE-20260613-7b7ae1` — ChannelService.create() now requires ChannelCreateRequest (no 9-arg overload)
- `GE-20260517-5b8e78` — qhorus core services (ChannelService, ReactiveChannelStore) are CDI-injectable
- `GE-20260516-2805b7` — abstract superclasses indexed by Jandex fail deployment
