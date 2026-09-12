# YAML-First Plugin Architecture — Design Spec

## Status

**All infrastructure described in this spec is proposed new work** unless explicitly
marked as "EXISTS". The three-layer architecture, all Layer 1 Java SPIs, all Layer 2
YAML primitives, and the Layer 3 plugin schema are new proposals. The spec builds on
top of existing infrastructure in `casehub-desiredstate` and `casehub-ops` — see §SPI
Quad Integration for the mapping.

## Overview

A three-layer architecture enabling infrastructure plugins to be authored primarily in YAML, with Java as the escape hatch for transport and protocol concerns. Each plugin is a complete self-healing unit: it provisions a resource, detects its own problems, resolves them using CBR-guided strategies, and learns from outcomes.

A **plugin YAML file** defines a new node type's SPI implementations — how to provision it, how to read its actual state, what faults look like, what CBR features are relevant, and what RAS situations to detect. Plugin YAML files extend the node type ecosystem; **they do not replace `YamlGraph` topology declarations**. Users compose plugins into topologies using the existing `YamlGraph` format.

**Repos:** casehub-desiredstate (domain-agnostic primitives), casehub-ops (domain-specific plugins)

**Key constraint:** Type safety throughout. Java records are canonical; JSON Schema is generated for IDE validation and refactoring. Every reference between YAML files is typed and resolvable.

## Architecture

### SPI Quad Integration

EXISTS: Every domain module in casehub-ops implements five SPIs from `casehub-desiredstate-api`:
`GoalCompiler`, `ActualStateAdapter`, `NodeProvisioner`, `FaultPolicy`, and `EventSource`.

A YAML plugin compiles into implementations of these SPIs at Quarkus build time. The
plugin YAML is the authoring format; the SPI Quad is the runtime contract.

| Plugin YAML Section | Compiles To | How |
|---|---|---|
| `plugin.nodeType` + `actual-state` | `ActualStateAdapter` contribution | Generated adapter uses rest-call/graphql-call primitives to read actual state. Registered via `handledTypes()` for the declared `nodeType`. `compare-state` fields feed into drift detection by comparing desired `NodeSpec` fields against actual API responses. |
| `plugin.nodeType` + `provisioner` | `NodeProvisioner` contribution | Generated provisioner uses rest-call/graphql-call primitives for create/update/delete. Registered via `handledTypes()` for the declared `nodeType`. Receives `ProvisionContext` with `tenancyId` and approval state. `plugin.resyncInterval` passes through to `NodeProvisioner.resyncInterval()` (default PT5M, validated ≥ 1s by `DefaultNodeProvisionerRouter`). Runtime overrides via `PreferenceProvider` take precedence. |
| `approval` | `ApprovalEvaluator` contribution | Generated evaluator classifies risk by action type (create/update/delete) with optional namespace-pattern overrides. Compiles to an `ApprovalEvaluator` (EXISTS: `casehub-ops-api`) that returns `ApprovalDecision.RequiresApproval` when risk ≥ configured threshold, `AutoApproved` otherwise. Without an `approval:` section, the plugin auto-approves all operations. |
| `fault-policy` | `ThresholdFaultPolicy` configuration | The plugin's fault tiers compile into `ThresholdFaultPolicy.Tier` entries with `TypedFaultPolicy` actions. Fault types use `FaultType` enums (PROVISION_FAILED, NODE_DEGRADED), not HTTP status codes. Transport-level errors (HTTP retries) are handled by Layer 2 `retry` primitive, not by the graph-level fault policy. **Retry budget:** total HTTP requests before escalation is bounded by `retry.max-attempts × fault-policy highest-tier threshold`. Build-time validation warns when this product exceeds 50. |
| `cbr` | `CbrFaultPolicy` learning surface metadata | Plugin CBR declarations define what contextual features to extract when building `RetrievalContext` for the `ConfigurationRetriever`. This enriches the existing graph-level CBR with per-node-type domain knowledge — it does not replace it. |
| `ras.situations` | `SituationDefinition` registrations | Each situation compiles into a `SituationDefinition` record registered with the RAS `SituationDefinitionProvider`. Supports the full `SituationDefinition` field set — simple fields map directly; expression fields (`correlationKeyExpression`, `eventFilter`) use JQ expression syntax compiled to `JQExpressionEvaluator` (EXISTS: `casehub-platform-api`). See §RAS for field mapping. |
| (implicit) | `EventSource` | YAML plugins do not declare an event-source section. Plugin-generated adapters and provisioners use periodic re-sync only (the reconciliation loop's default interval-grouped timers via `plugin.resyncInterval`). For sub-second drift detection via external change feeds (K8s watch API, webhooks, SSE), a Java `EventSource` implementation is required — the Layer 1 `StreamingStateSource` SPI provides the contract. This is an intentional boundary: streaming protocols are transport-specific and vary too widely for a YAML-level abstraction. |

The existing `YamlGraph` format (EXISTS: `desiredState`, `variables`, `nodes`, `faultPolicy`,
`invariants`, `rules`, `lifecycle`, `iterations`, `imports`) is **not changed**. Plugin YAML
files register new node types; `YamlGraph` topology files reference those node types via
`type:` fields in their `nodes:` block.

**Topology authoring flow:**

```
1. Plugin YAML (this spec)     → registers nodeType + SPI implementations
2. YamlGraph topology (EXISTS) → declares nodes using registered nodeTypes
3. YamlGraphRecorder (EXISTS)  → compiles topology into DesiredStateGraph
4. Reconciliation loop (EXISTS)→ drives the SPI Quad implementations
```

### Relationship to YamlGraph

The existing `YamlGraph` format (delivered by #116, #117) declares WHAT should exist — a
graph of nodes with types, specs, and dependencies. The YAML plugin architecture defines
HOW each node type is implemented — how to provision it, how to read its state, how to
handle faults.

| Concern | Where | Format |
|---|---|---|
| "I want 3 K8s deployments behind a load balancer" | `YamlGraph` topology file | `nodes:`, `dependsOn:`, `variables:`, `lifecycle:` |
| "A K8s deployment is provisioned via REST API to /apis/apps/v1/..." | Plugin YAML file | `actual-state:`, `provisioner:`, `fault-policy:`, `cbr:`, `ras:` |

A `NodeSpecFactory` (EXISTS: `NodeSpecFactory.create(Map<String, Object>)`) bridges the
two: the topology file's `spec:` block is parsed by the plugin's factory into a typed
`NodeSpec` record.

### Three Layers

All three layers below are **proposed new work**.

```
┌─────────────────────────────────────────────────────────┐
│  Layer 3: YAML Plugins (casehub-ops/plugins/)           │
│  k8s-deployment.yaml, cloudflare-dns.yaml, ...          │
│  Each: actual-state + provisioner + fault-policy         │
│        + cbr + ras                                       │
├─────────────────────────────────────────────────────────┤
│  Layer 2: YAML Primitives (casehub-desiredstate/        │
│           yaml/primitives/)                              │
│  rest-call, graphql-call, json-extract,                  │
│  compare-state, paginate, poll-until, auth-ref,          │
│  retry                                                   │
│  Compose over each other AND over Layer 1                │
├─────────────────────────────────────────────────────────┤
│  Layer 1: Java SPIs (casehub-desiredstate/               │
│           yaml/spi/)                                     │
│  RestClient, GraphQlClient, AuthProvider,                │
│  JsonPathExtractor, StreamingStateSource,                │
│  RateLimiter, NodeSpecSchemaGenerator                    │
└─────────────────────────────────────────────────────────┘
```

### Composability Proof

Three layers compose cleanly. Example: reading paginated state from a K8s API.

A plugin's `actual-state.list` uses `paginate`, which uses `rest-call`, which uses `RestClient` (Java). The plugin author writes:

```yaml
actual-state:
  list:
    paginate:
      strategy: link-header
      rest-call:
        method: GET
        url: "${plugin.baseUrl}/apis/apps/v1/namespaces/${plugin.namespace}/deployments"
        auth: "${plugin.auth}"
      extract:
        forEach: "$.items[*]"
        mappings:
          nodeId: "$.metadata.name"
          replicas: "$.spec.replicas"
          readyReplicas: "$.status.readyReplicas"
          image: "$.spec.template.spec.containers[0].image"
```

No Java. Three YAML layers (plugin → paginate → rest-call), one Java layer underneath (RestClient).

## Layer 1: Java SPIs

**All proposed new work.** These are transport and computation primitives that YAML cannot express.

### api-client (shared)

| SPI | Purpose | Key Methods |
|-----|---------|------------|
| `AuthProvider` | Authentication with token lifecycle | `authenticate(SecretRef) → AuthSession` (headers + TTL + refresh) |
| `RetryPolicy` | Backoff and retry logic | `shouldRetry(int statusCode, int attempt) → RetryDecision` |
| `RateLimiter` | Cross-request throttling | `acquire(String endpoint) → Permit` |

Implementations: `BearerTokenAuth`, `OAuth2Auth`, `ApiKeyAuth`, `BasicAuth`. Configured by YAML `auth-ref` primitive.

### rest-client

| SPI | Purpose | Key Methods |
|-----|---------|------------|
| `RestClient` | HTTP request execution | `execute(RestRequest) → RestResponse` |
| `JsonPathExtractor` | JSONPath evaluation | `extract(JsonNode, Map<String, String> paths) → Map<String, Object>` |
| `PaginationStrategy` | Cursor/offset/link-header | `nextPage(RestResponse) → Optional<RestRequest>` |

Implementations: `CursorPagination`, `OffsetPagination`, `LinkHeaderPagination`. Quarkus REST client underneath.

### graphql-client

| SPI | Purpose | Key Methods |
|-----|---------|------------|
| `GraphQlClient` | GraphQL query/mutation execution | `execute(GraphQlRequest) → GraphQlResponse` |
| `RelayPagination` | Relay cursor convention | `nextPage(GraphQlResponse, String connectionField) → Optional<GraphQlRequest>` |

Response extraction is native to GraphQL — the query itself selects fields. No JSONPath equivalent needed.

### schema-gen

| SPI | Purpose | Key Methods |
|-----|---------|------------|
| `NodeSpecSchemaGenerator` | Java record → JSON Schema | `generate(Class<? extends NodeSpec>) → JsonSchema` |

**Proposed new capability.** Would run at Quarkus build time via `YamlDesiredStateProcessor` (EXISTS). Implementation requires Jandex reflective access to record component names — feasible but non-trivial. Feeds IDE plugins for validation and autocomplete.

### streaming

| SPI | Purpose | Key Methods |
|-----|---------|------------|
| `StreamingStateSource` | WebSocket/SSE for live state | `subscribe(StreamConfig) → Flow<StateEvent>` |

For resources that push state changes (K8s watch API, CloudEvents). Falls back to polling when streaming unavailable.

## Layer 2: YAML Primitives

**All proposed new work.** Composable YAML operations backed by Layer 1 Java SPIs.

**Scope: HTTP API primitives.** Layer 2 primitives express CRUD operations against REST
and GraphQL endpoints. This is the right abstraction for the long tail of cloud vendor APIs
(Cloudflare, Supabase, Fly.io, OCI, ACME/Let's Encrypt) where writing a Java provisioner
per vendor is disproportionate to the complexity. For resources managed via specialised
client libraries or CLI tools (K8s fabric8 watch/informer, Terraform CLI, Ansible),
production integrations may prefer Java SPI implementations that use those libraries
directly. The K8s plugin examples in this spec use the K8s REST API — which is valid, as
fabric8 is a convenience layer over the same HTTP API — but the existing
`KubernetesNodeProvisioner` (EXISTS) with fabric8 Watch is more appropriate for
production K8s management where watch streams and resource versioning matter.

### Interpolation Model

Plugin YAML extends the existing `VariableResolver` (EXISTS: `casehub-platform-yaml-core`)
with plugin-specific `VariableSource` registrations. `VariableResolver` provides prefix-based
dispatch via `${prefix.name}` pattern, deferred-prefix support for runtime-only values,
error handling via `UnresolvedVariableException` with context, and `withScope()`/
`withChainedScope()` for composable prefix registration. Plugin-specific prefixes:

| Prefix | Scope | Resolution Time | Examples |
|--------|-------|----------------|---------|
| `${plugin.name}` | Plugin `defaults:` block and `auth:` block | Build-time | `${plugin.baseUrl}`, `${plugin.namespace}`, `${plugin.auth}` |
| `${spec.name}` | NodeSpec record fields (resolved from the node being provisioned) | Runtime | `${spec.name}`, `${spec.replicas}`, `${spec.image}` |
| `${response.path}` | Most recent rest-call/graphql-call response (JSONPath) | Runtime | `${response.$.id}`, `${response.$.status}` |
| `${step.N.path}` | Named or indexed step result in a provisioner sequence | Runtime | `${step.0.$.operation.id}` |
| `${fault.name}` | FaultEvent fields (in fault-policy tier spec templates only) | Runtime (fault time) | `${fault.nodeId}`, `${fault.type}`, `${fault.detail}` |

The `plugin` prefix resolves at build time from the plugin's `defaults:` block. The `spec`,
`response`, `step`, and `fault` prefixes are registered as deferred prefixes during build-time
validation (recognised but not resolved) and resolve at runtime via `VariableResolver.withScope()`
with the appropriate `VariableSource`.

**Migration note:** The existing `YamlFaultPolicyBuilder.resolveFaultString()` (EXISTS) uses
hardcoded `String.replace()` for `${fault.*}` variables. The plugin compiler will use
`VariableResolver` with a `fault` `VariableSource` backed by `FaultEvent` fields, giving
uniform error handling across all plugin sections. Migration of `YamlFaultPolicyBuilder` to
`VariableResolver` is tracked separately — the two approaches produce identical output for
the three supported variables.

Build-time validation: every template expression must have a recognised prefix. Unrecognised
prefixes fail the build. `${spec.*}` field references are validated against the NodeSpec
record's component names via Jandex (proposed — see §Build-Time Validation).

**Runtime resolution errors:** Template expressions that pass build-time validation can still
fail at runtime. The runtime error model:

| Condition | Behaviour |
|---|---|
| `${spec.field}` resolves to `null` or `Optional.empty()` | Fail with `TemplateResolutionException` naming the expression, scope, and step |
| JSONPath extraction (`$.path`) returns no matches | Variable set to `null`; if referenced in subsequent step, that step fails with source trace |
| `${step.N.*}` references step that has not executed (out of range) | Fail with `TemplateResolutionException` — step index validated before execution |
| Type mismatch (e.g., string value in numeric field) | Fail at deserialization with coercion error — existing `ObjectMapper.convertValue()` path |

All runtime resolution errors produce structured `TemplateResolutionException` with: expression
text, resolved scope, step index (if in provisioner sequence), and the original cause. These
propagate as non-retryable `PROVISION_FAILED` faults to the graph-level fault policy.

### rest-call

```yaml
rest-call:
  method: POST | GET | PUT | DELETE | PATCH
  url: "${plugin.baseUrl}/path/${spec.field}"
  auth: "${plugin.auth}"
  headers:
    Content-Type: application/json
  body:
    field: "${spec.field}"
  expect: 200
  extract:
    nodeId: "$.result.id"
```

### graphql-call

```yaml
graphql-call:
  endpoint: "${plugin.baseUrl}/graphql"
  auth: "${plugin.auth}"
  query: |
    mutation CreateRecord($input: RecordInput!) {
      createRecord(input: $input) { id status }
    }
  variables:
    input:
      name: "${spec.name}"
      type: "${spec.recordType}"
  extract:
    nodeId: "$.data.createRecord.id"
```

### json-extract

```yaml
json-extract:
  source: "${response}"
  mappings:
    nodeId: "$.id"
    status: "$.status"
    ip: "$.addresses[0].ip"
```

### compare-state

Defines per-plugin field-level drift comparison. At runtime, the generated
`ActualStateAdapter` reads actual state via the plugin's `actual-state` section,
then uses `compare-state` to determine which fields have drifted. This feeds into
the existing `NodeStatus.DRIFTED` mechanism — the adapter returns DRIFTED when any
compared field differs, triggering re-provisioning through the normal reconciliation
loop.

```yaml
compare-state:
  fields: [name, replicas, image, resources]
  ignore: [metadata.resourceVersion, status.observedGeneration]
  tolerance:
    cpu: 10%                  # percentage only: zero baseline → always drift
    memory: { percent: 10%, absolute: 50Mi }  # composite: absolute fallback when baseline is zero
```

### paginate

Composes over `rest-call`:

```yaml
paginate:
  strategy: cursor | offset | link-header | relay
  rest-call: { ... }
  cursor-path: "$.metadata.continue"
  results-path: "$.items"
  page-size: 100
  max-pages: 500           # bound resource consumption; exceeding emits structured error
  on-cursor-invalid: fail  # 410 Gone → fail operation, retry on next reconciliation cycle
```

**Consistency model:** Pagination produces an eventually-consistent snapshot — early pages
reflect state at T₁, late pages at Tₙ. Drift detection via `compare-state` may produce
false positives/negatives during large list operations. This is inherent to paginated APIs
and acceptable — the reconciliation loop self-corrects on subsequent cycles.

### poll-until

Composes over `rest-call`:

```yaml
poll-until:
  rest-call:
    method: GET
    url: "${plugin.baseUrl}/operations/${step.0.$.operation.id}"
  condition: "$.status == 'READY'"
  failure-condition: "$.status in ['FAILED', 'ERROR', 'CANCELLED']"
  interval: 5s
  timeout: 5m
  backoff: linear | exponential
```

### auth-ref

Defines authentication configuration for the plugin. The plugin's top-level `auth:`
block IS the auth-ref — there is one auth configuration per plugin, referenced via
`${plugin.auth}` in rest-call/graphql-call bodies. Multi-auth plugins (e.g., different
auth for control plane vs data plane) declare named auth blocks:

```yaml
auth:
  default:
    type: bearer-token | oauth2 | api-key | basic
    secret-ref: my-secret-name
  data-plane:
    type: oauth2
    token-url: "https://auth.example.com/oauth/token"
    client-id-ref: oauth-client-id
    scopes: [read, write]
```

Referenced as `${plugin.auth}` (resolves to `default`) or `${plugin.auth.data-plane}`.

### retry

Transport-level retry logic. This handles HTTP-level errors (429, 500, 502, 503) —
it is distinct from graph-level fault policy which handles `FaultType` enums after
provisioning fails.

```yaml
retry:
  max-attempts: 3
  backoff: exponential
  initial-delay: 1s
  max-delay: 30s
  retryable: [429, 500, 502, 503]
  fatal: [401, 403, 404]
  transport-errors: retryable    # DNS, TCP, TLS failures — retryable | fatal
  on-401: refresh-and-retry      # single token refresh before classifying as fatal
```

### Composability: YAML over YAML

`paginate` composes over `rest-call`. `poll-until` composes over `rest-call`. A plugin's `actual-state` composes over `paginate` which composes over `rest-call`. Three layers of YAML, Java only at the bottom.

A plugin can also compose primitives in sequence:

```yaml
provisioner:
  create:
    - rest-call:
        method: POST
        url: "${plugin.baseUrl}/resources"
        body: { name: "${spec.name}" }
        extract: { operationId: "$.operation.id" }
    - poll-until:
        rest-call:
          method: GET
          url: "${plugin.baseUrl}/operations/${step.0.$.operation.id}"
        condition: "$.status == 'READY'"
        failure-condition: "$.status in ['FAILED', 'ERROR']"
        interval: 5s
        timeout: 5m
```

## Layer 3: Plugin Schema

Each plugin YAML has two required sections (`actual-state`, `provisioner`) and four
optional sections (`approval`, `fault-policy`, `cbr`, `ras`). Omitting optional sections emits a
build-time warning — the plugin works but without self-healing capabilities. A plugin
with only `actual-state` + `provisioner` contributes a functioning `ActualStateAdapter`
and `NodeProvisioner` to the SPI Quad; approval adds risk-gated provisioning, fault-policy,
CBR, and RAS add progressively richer self-healing behaviour.

**Approval model:** Without an `approval:` section, the generated provisioner always returns
`ProvisionResult.Success` (auto-approved). With an `approval:` section, the generated
provisioner calls `ApprovalEvaluator.evaluate()` (EXISTS: `casehub-ops-api`) before
provisioning, returning `ProvisionResult.PendingApproval` when risk exceeds the threshold.
`SimpleTransitionExecutor` (EXISTS) wraps every provisioner call with
`PendingApprovalHandler` — the approval lifecycle (check → record → acknowledge) is
handled by existing infrastructure.

### Complete Plugin Example — KubernetesDeploymentSpec

**Syntax reference only.** This example demonstrates the full YAML plugin format against
the K8s REST API. In production, K8s deployments are provisioned by
`KubernetesNodeProvisioner` (EXISTS) via fabric8 — deploying this plugin alongside the
existing provisioner would cause a startup crash due to duplicate `NodeType` registration.
See §Plugin Catalogue for production plugins.

```yaml
plugin:
  name: k8s-deployment
  version: 1.0
  nodeType: k8s_deployment
  resyncInterval: PT5M              # ISO 8601 duration; default PT5M; validated >= 1s
  # EXISTS: nodeType must resolve to a registered @NodeTypeId

auth:
  default:
    type: bearer-token
    secret-ref: k8s-service-account-token

defaults:
  baseUrl: "https://kubernetes.default.svc"
  namespace: default

# ── Section 1: Actual State ──────────────────────────
# Compiles to: ActualStateAdapter contribution for nodeType k8s_deployment

actual-state:
  list:
    paginate:
      strategy: link-header
      rest-call:
        method: GET
        url: "${plugin.baseUrl}/apis/apps/v1/namespaces/${plugin.namespace}/deployments"
        auth: "${plugin.auth}"
      results-path: "$.items"
      extract:
        forEach: "$.items[*]"
        mappings:
          nodeId: "$.metadata.name"
          replicas: "$.spec.replicas"
          readyReplicas: "$.status.readyReplicas"
          image: "$.spec.template.spec.containers[0].image"
          conditions: "$.status.conditions"

  drift:
    compare-state:
      fields: [replicas, image, resources, env]
      ignore: [metadata.resourceVersion, status.observedGeneration]

# ── Section 2: Provisioner ───────────────────────────
# Compiles to: NodeProvisioner contribution for nodeType k8s_deployment
# Receives ProvisionContext with tenancyId and approval state at runtime
#
# Idempotency: The reconciliation loop provides create-idempotency through
# actual-state reconciliation. If a create POST succeeds but the response is
# lost, the next cycle's actual-state.list discovers the resource and skips
# re-creation. Plugins for vendors whose list API has eventual consistency
# should guard creates by checking actual-state before POST.

provisioner:
  create:
    rest-call:
      method: POST
      url: "${plugin.baseUrl}/apis/apps/v1/namespaces/${plugin.namespace}/deployments"
      auth: "${plugin.auth}"
      body:
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: "${spec.name}"
          labels: "${spec.labels}"
        spec:
          replicas: "${spec.replicas}"
          selector:
            matchLabels: "${spec.labels}"
          template:
            metadata:
              labels: "${spec.labels}"
            spec:
              containers:
                - name: "${spec.name}"
                  image: "${spec.image}"
                  resources: "${spec.resources}"
      extract:
        nodeId: "$.metadata.name"

  update:
    rest-call:
      method: PUT
      url: "${plugin.baseUrl}/apis/apps/v1/namespaces/${plugin.namespace}/deployments/${spec.nodeId}"
      auth: "${plugin.auth}"
      body:
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: "${spec.name}"
          labels: "${spec.labels}"
        spec:
          replicas: "${spec.replicas}"
          selector:
            matchLabels: "${spec.labels}"
          template:
            metadata:
              labels: "${spec.labels}"
            spec:
              containers:
                - name: "${spec.name}"
                  image: "${spec.image}"
                  resources: "${spec.resources}"

  delete:
    rest-call:
      method: DELETE
      url: "${plugin.baseUrl}/apis/apps/v1/namespaces/${plugin.namespace}/deployments/${spec.nodeId}"
      auth: "${plugin.auth}"

# ── Section 3: Approval ─────────────────────────────
# Compiles to: ApprovalEvaluator contribution for nodeType k8s_deployment
# Called by SimpleTransitionExecutor before provisioning — returns
# PendingApproval when risk exceeds threshold, auto-approved otherwise.

approval:
  rules:
    - action: create
      risk: medium
    - action: update
      risk: low
    - action: delete
      risk: high
  overrides:
    - namespace-pattern: "prod-*"
      risk: high                    # all operations in prod namespaces escalate to high

# ── Section 4: Fault Policy ──────────────────────────
# Compiles to: ThresholdFaultPolicy configuration
# Transport-level errors (HTTP 429/500) handled by Layer 2 retry primitive
# This section handles graph-level faults (PROVISION_FAILED, NODE_DEGRADED)

fault-policy:
  faultTypes: [PROVISION_FAILED, NODE_DEGRADED]
  # PROPOSED: counter-reset and counter-window require extending FaultCountStore (EXISTS)
  # and ThresholdFaultPolicy (EXISTS) — neither supports time-windowed counting or
  # signal-triggered reset today. FaultCountStore.reset() is explicit (caller decides);
  # ThresholdFaultPolicy counts indefinitely until reset. These are new capabilities.
  counter-reset: on-successful-provision   # PROPOSED: reset when faulting node provisions successfully
  # on-successful-provision is the only counter-reset mode. The reconciliation loop calls
  # FaultCountStore.reset() when a previously-faulting node provisions successfully.
  # on-outcome-signal (CBR outcome triggers reset) was considered but requires cross-policy
  # communication between CbrFaultPolicy and ThresholdFaultPolicy — the runtime explicitly
  # forbids shared state between fault policies (FaultPolicyEngine composes independently).
  # If cross-policy signalling is needed in future, it requires new runtime infrastructure.
  counter-window: PT10M               # PROPOSED: only count faults within this window
  # Review node types: Each tier's reviewNode.type must be a registered @NodeTypeId.
  # The existing YamlFaultPolicyBuilder (EXISTS) calls NodeSpecRegistry.resolve(type)
  # and deserialises spec template via Jackson convertValue into the resolved class.
  #
  # For YAML plugins, review specs are Java records in casehub-ops-api — same pattern
  # as IoTReviewSpec (EXISTS: faultedNode + reason). Each plugin family needs a review
  # spec record registered via @NodeTypeId:
  #   @NodeTypeId("k8s_deployment_review")
  #   public record K8sDeploymentReviewSpec(String action, NodeId faultedNode) implements NodeSpec {}
  #
  # The NodeProvisioner for the review type handles it as a no-op Success (review
  # completes when the human approves the WorkItem) — same as IoTNodeProvisioner
  # handling iot-review.
  tiers:
    - threshold: 3
      reviewNode:
        type: k8s_deployment_review
        spec:
          action: restart-pod
          faultedNode: "${fault.nodeId}"
    - threshold: 5
      reviewNode:
        type: k8s_deployment_review
        spec:
          action: escalate-human
          faultedNode: "${fault.nodeId}"
        humanGating: ALL

# ── Section 5: CBR — Learning Surface ────────────────
# Declares what contextual features to extract when building RetrievalContext
# for the existing CbrFaultPolicy (EXISTS: ConfigurationRetriever/Adapter).
# Does NOT replace graph-level CBR — enriches it with per-node-type domain knowledge.
#
# Cold-start path: when CBR has no cases (new deployment), CbrFaultPolicy.onFault()
# returns List.of() — no mutations. The fault-policy tiers (Section 4) provide the
# baseline remediation path. As CBR accumulates cases from successful tier-driven
# remediations (via outcome-signals), it begins proposing learned strategies that
# may pre-empt or supplement tier escalation.

cbr:
  context-features:
    - error-type          # OOMKilled, CrashLoopBackOff, ImagePullBackOff
    - resource-utilisation # CPU/memory percentage at failure time
    - time-of-day         # peak vs off-peak
    - replica-count       # how many replicas were running
    - previous-image      # the image before the failing one
  resolution-strategies:
    - restart-pod
    - rollback-image
    - scale-horizontal
    - scale-vertical
    - cordon-node
  outcome-signals:
    - pod-healthy-after: 5m
    - no-restart-within: 30m
    - ready-replicas-match: true

# ── Section 6: RAS — Detection Situations ────────────
# Each situation compiles to a SituationDefinition (EXISTS: casehub-ras-api)
# with full field support. Registered via SituationDefinitionProvider.
# Ganglion IDs are auto-prefixed with plugin.name at build time to prevent
# namespace collisions across plugins (e.g., restart-counter → k8s-deployment:restart-counter).
# SituationDefinitionRegistry (EXISTS) throws on duplicate ganglionId — auto-prefixing
# ensures plugins cannot collide.
#
# SituationDefinition field mapping (12 fields):
#   Required in YAML:  situationId, eventTypes, chainMode, triggerAction
#   Optional in YAML:  correlationWindow, eventBufferDelay, triggerMode (default: fire-once),
#                      correlationKeyExpression (JQ), eventFilter (JQ),
#                      dynamicCaseData (map of JQ expressions), deadline
#   Not in YAML:       feedbackConfig (requires Java — too many interdependent parameters
#                      for declarative expression; plugins needing feedback tuning use a
#                      Java SituationDefinitionProvider)
#
# Expression fields use JQ syntax compiled to JQExpressionEvaluator (EXISTS:
# casehub-platform-api). JQ is the natural fit: events are JSON documents, JQ
# operates on JSON, and the platform already has JQExpressionEvaluator.
#
# triggerAction build-time transformation: The YAML uses a flat format where
# caseNamespace/caseName/caseVersion are siblings of type:. The build-time
# compiler extracts these fields and creates:
#   new TriggerAction.CreateCase(new CaseTriggerConfig(caseNamespace, caseName, caseVersion))
# This is NOT Jackson deserialization — the compiler creates Java objects directly.
# The flat YAML format is more ergonomic than the nested Java record structure.

ras:
  situations:
    - situationId: pod-crash-loop
      eventTypes: [k8s.pod.restart]
      chainMode:
        type: count
        ganglionId: restart-counter
        requiredCount: 3
      correlationWindow: PT10M
      eventBufferDelay: PT2S
      correlationKeyExpression: ".data.podName"         # JQ: correlate by pod name
      eventFilter: '.data.namespace == "default"'       # JQ: only default namespace
      triggerAction:
        type: create-case
        caseNamespace: k8s
        caseName: deployment-incident
        caseVersion: "1.0"
      triggerMode:
        type: fire-once
      deadline: PT1H

    - situationId: memory-pressure
      eventTypes: [k8s.metrics.memory]
      chainMode:
        type: threshold
        ganglia: [mem-usage]
        minConfidence: 0.85
      correlationWindow: PT5M
      correlationKeyExpression: ".data.deploymentName"  # JQ: correlate by deployment
      triggerAction:
        type: notify-only
      dynamicCaseData:
        memoryPct: ".data.utilizationPct"               # JQ: extract for case context
        nodeName: ".data.nodeName"

    - situationId: image-pull-failure
      eventTypes: [k8s.pod.image-pull-failed]
      chainMode:
        type: count
        ganglionId: pull-failure
        requiredCount: 1
      correlationWindow: PT2M
      triggerAction:
        type: create-case
        caseNamespace: k8s
        caseName: deployment-incident
        caseVersion: "1.0"

    - situationId: replica-unavailable
      eventTypes: [k8s.deployment.replica-unavailable]
      chainMode:
        type: streak
        ganglionId: unavailable-streak
        requiredCount: 3
      correlationWindow: PT15M
      triggerAction:
        type: create-case
        caseNamespace: k8s
        caseName: deployment-incident
        caseVersion: "1.0"
```

## Plugin Catalogue — 7 Production Plugins

K8s deployment, service, and ingress are managed in production by `KubernetesNodeProvisioner`
(EXISTS) via fabric8 — these types already have registered `NodeProvisioner` and
`ActualStateAdapter` beans. The §Complete Plugin Example uses `k8s_deployment` to
demonstrate YAML syntax against a well-known API; it is a **syntax reference**, not a
production plugin. Deploying a YAML plugin for a nodeType that already has a Java
provisioner causes a startup crash (`DefaultNodeProvisionerRouter` throws
`IllegalArgumentException` on duplicate `NodeType`).

| # | Plugin | nodeType | API Protocol | CaseHub Integration |
|---|--------|----------|-------------|---------------------|
| 1 | KubernetesSecret | k8s_secret | REST (K8s API) | Ledger: tamper-evident rotation audit trail |
| 2 | CloudflareDns | cloudflare_dns_record | REST (Cloudflare API) | Trust on propagation speed vs other DNS providers |
| 3 | CloudflareWorker | cloudflare_worker | REST (Cloudflare API) | CBR: cold start avoidance strategies |
| 4 | SupabaseDatabase | supabase_database | REST (Supabase API) | Pre-emptive: storage trending toward free-tier limit |
| 5 | FlyIoMachine | flyio_machine | REST (Fly.io API) | Trust: region reliability scores from CBR history |
| 6 | OracleCloudVm | oci_vm_instance | REST (OCI API) | Pre-emptive: free-tier usage approaching limits |
| 7 | LetsEncryptCert | letsencrypt_certificate | REST (ACME) | Ledger: certificate lifecycle audit trail |

### InfraNodeSpec Sealed Hierarchy

EXISTS: `InfraNodeSpec` is a `sealed interface` permitting 15 types. Each of the 7
production plugins requires a new sealed variant:

| Plugin | New Sealed Variant | Notes |
|---|---|---|
| KubernetesSecret | `K8sSecretSpec` | Structurally similar to K8sConfigMapSpec but with base64 encoding and different RBAC |
| CloudflareDns | `CloudflareDnsRecordSpec` | DNS record type, name, target, TTL, proxy status |
| CloudflareWorker | `CloudflareWorkerSpec` | Script name, routes, environment variables, KV bindings |
| SupabaseDatabase | `SupabaseDatabaseSpec` | Project ref, region, plan, connection pooling config |
| FlyIoMachine | `FlyIoMachineSpec` | App name, region, image, size, services config |
| OracleCloudVm | `OracleCloudVmSpec` | Shape, image, VNIC, availability domain |
| LetsEncryptCert | `LetsEncryptCertSpec` | Domains, challenge type, key algorithm |

Each variant is a Java record in `casehub-ops-api` with typed fields. The plugin YAML's
`provisioner` section references these fields via `${spec.fieldName}` — build-time
validation ensures the referenced fields exist on the record (see §Build-Time Validation).

Adding sealed variants is mechanical: add the record, add it to the `permits` clause,
add a `@NodeTypeId` annotation, register an `InfraNodeSpecFactoryProvider` (EXISTS pattern:
`InfraWrappingFactory`). No architectural change — the pattern is established by C6
Canonical Deployment Topologies.

**Future: fully YAML-defined types.** For community plugins that should not require Java
records, a future `YamlDynamicNodeSpec` implementing `NodeSpec` directly (bypassing the
`InfraNodeSpec` sealed hierarchy) could carry `type + Map<String, Object>` properties.
This is out of scope for the initial 7 production plugins but is a natural extension
point. Tracked as a future enhancement, not a prerequisite.

## Type Safety and IDE Support

### Build-Time Validation Pipeline

EXISTS: `@NodeTypeId` annotation scan → `NodeSpecRegistry` (type string → class) → `NodeSpecFactory.create(Map<String, Object>)` at runtime.

**Proposed additions** to `YamlDesiredStateProcessor` (EXISTS):

```
EXISTS: @NodeTypeId Jandex scan → NodeSpecRegistry (type → class)
NEW:    NodeSpecSchemaGenerator (class → JSON Schema via Jandex record component introspection)
NEW:    Plugin YAML validation against schema
NEW:    Build fails on type mismatch
```

### Plugin YAML Validation

The Quarkus build-time extension (`YamlDesiredStateProcessor`, EXISTS) validates plugins.
Existing vs proposed validations:

| Validation | Status | Notes |
|---|---|---|
| `nodeType` resolves to a registered `@NodeTypeId` | **EXISTS** | `YamlDesiredStateProcessor.scanNodeTypes()` does this today |
| `${spec.field}` references resolve to NodeSpec record fields | **PROPOSED** | Requires Jandex introspection of record component names — feasible but non-trivial |
| `${plugin.auth}` resolves to a declared auth block | **PROPOSED** | New plugin-level validation |
| `cbr.context-features` and `cbr.resolution-strategies` are non-empty (or explicitly empty with warning) | **PROPOSED** | See §OQ1 |
| `ras.situations` has at least one situation (or explicitly empty with warning) | **PROPOSED** | See §OQ1 |
| `compare-state.fields` reference valid NodeSpec fields | **PROPOSED** | Same Jandex introspection as `${spec.*}` validation |
| Fault policy `faultTypes` resolve to `FaultType` enum values | **PROPOSED** | Compile-time enum validation |
| `counter-reset` value is `on-successful-provision` | **PROPOSED** | Build-time error if counter-reset has any value other than `on-successful-provision` (the only supported mode). |
| RAS situation `triggerAction` has `caseNamespace`, `caseName`, `caseVersion` as direct fields for `create-case` | **PROPOSED** | Mirrors `CaseTriggerConfig` record: 3 non-null fields + optional `baseCaseData`. Build-time compiler transforms flat YAML to nested `TriggerAction.CreateCase(new CaseTriggerConfig(...))` — NOT Jackson deserialization. |
| RAS `correlationKeyExpression` and `eventFilter` are valid JQ expressions | **PROPOSED** | Compiled to `JQExpressionEvaluator` at build time. Syntax validation via JQ parser — invalid expressions fail the build. |
| RAS `dynamicCaseData` values are valid JQ expressions | **PROPOSED** | Each map value compiled to `JQExpressionEvaluator`. Same JQ syntax validation. |
| `plugin.resyncInterval` ≥ 1s (ISO 8601 duration) | **PROPOSED** | Matches `DefaultNodeProvisionerRouter.MIN_RESYNC`. Default PT5M if omitted. |
| `approval.rules` action values are valid `StepAction` values | **PROPOSED** | Compile-time enum validation against `StepAction` (create → PROVISION, update → PROVISION, delete → DEPROVISION). |
| `retry.max-attempts × fault-policy highest-tier threshold ≤ 50` | **PROPOSED** | Build-time WARNING (not error) when total retry budget exceeds 50. Alerts plugin authors to amplification risk. |
| Plugin `nodeType` does not collide with existing `NodeProvisioner.handledTypes()` | **PROPOSED** | Build fails if a YAML plugin declares a nodeType already claimed by a Java `NodeProvisioner` bean. Prevents `DefaultNodeProvisionerRouter` `IllegalArgumentException` at startup. |
| Duplicate YAML filenames across JARs emit build-time WARNING | **PROPOSED** | `discoverYamlFiles()` uses `seen.add(fileName)` for dedup — currently silent. Warning alerts when a second JAR ships a file with the same name. |

### IDE Plugin Contract

**Proposed.** JSON Schema per plugin enables:
- **Autocomplete:** `${spec.}` triggers field list for the declared nodeType
- **Validation:** red squiggle on `${spec.nonexistent}`
- **Rename:** rename a NodeSpec field in Java → schema regenerates → IDE flags all YAML files using the old name
- **Navigate to definition:** `nodeType: k8s_deployment` → jump to `KubernetesDeploymentSpec.java`

IntelliJ and VS Code plugins consume the generated JSON Schema. Plugin development is a future phase.

## Testing Strategy

| Plugin type | Local test | Integration test |
|-------------|-----------|-----------------|
| K8s Secret (1) | Kind on Podman | Real K8s cluster (CI) |
| Cloudflare (2-3) | WireMock container | Real Cloudflare free account |
| Supabase (4) | WireMock container | Real Supabase free project |
| Fly.io (5) | WireMock container | Real Fly.io free account |
| Oracle Cloud (6) | WireMock container | Real OCI always-free |
| Let's Encrypt (7) | Pebble (ACME test server) | Let's Encrypt staging |
| K8s syntax reference | Kind on Podman | Framework validation only — not deployed alongside existing provisioner |

All local tests run on Podman. No Docker dependency.

## Cross-Vendor Topology Example

A `YamlGraph` topology (EXISTS: format from #116/#117) composing nodes from
multiple YAML plugins:

```yaml
desiredState:
  namespace: myapp
  name: cross-vendor-topology

variables:
  db_region: eu-west-1

nodes:
  myapp-db:
    type: supabase_database
    spec:
      name: myapp-db
      region: "${var.db_region}"
      plan: free
    dependsOn: []

  api-server:
    type: k8s_deployment
    spec:
      name: api-server
      image: myapp/api:latest
      replicas: 3
      env:
        DATABASE_URL: "${var.supabase_connection_string}"
    dependsOn: [myapp-db]

  api-service:
    type: k8s_service
    spec:
      name: api-service
      port: 8080
      targetPort: 8080
    dependsOn: [api-server]

  api-ingress:
    type: k8s_ingress
    spec:
      name: api-ingress
      host: api.myapp.com
      service: api-service
      tls: true
    dependsOn: [api-service]

  api-dns:
    type: cloudflare_dns_record
    spec:
      name: api.myapp.com
      recordType: CNAME
      target: "${var.lb_hostname}"
    dependsOn: [api-ingress]
```

This is a single-phase graph — no `lifecycle:` section. The reconciliation loop manages
all five resources across three vendors continuously, using the dependency edges for
ordering (data-tier before compute-tier before edge-tier).

For multi-phase topologies, nodes move inside `lifecycle.phases` — the existing
`YamlDesiredStateProcessor.validateLifecycle()` rejects graphs with both top-level
`nodes:` and `lifecycle:`. Each `YamlPhase` contains `Map<String, YamlNode> nodes`
(full node definitions, not ID references). Example:

```yaml
desiredState:
  namespace: myapp
  name: phased-topology

lifecycle:
  phases:
    - id: data-tier
      completionCondition: allPresent
      nodes:
        myapp-db:
          type: supabase_database
          spec:
            name: myapp-db
            region: "${var.db_region}"
            plan: free
    - id: compute-tier
      completionCondition: allPresent
      nodes:
        api-server:
          type: k8s_deployment
          spec:
            name: api-server
            image: myapp/api:latest
            replicas: 3
          dependsOn: [myapp-db]
```

Note: cross-node field references (e.g., reading a connection string from one node's
actual state to inject into another node's spec) require the `YamlGraph` `variables:`
block and external variable injection. Direct inter-node field access is not supported
in the interpolation model.

## Open Questions (from adversarial review)

### OQ1: Plugin adoption barrier — resolved

SETTLED: Sections 3-6 (`approval`, `fault-policy`, `cbr`, `ras`) are now optional. Only
`actual-state` and `provisioner` are required. Omitting optional sections emits a build-time warning
("`Plugin 'cloudflare_dns_record' has no CBR learning surface — self-healing will be
limited to threshold-based fault policy only`"). This removes the adoption barrier without
sacrificing self-healing capabilities for plugins that choose to declare them.

**Validation semantics for declared sections:**
- `cbr.context-features`: string identifiers. Validated at runtime when `RetrievalContext` is built — the CBR adapter looks up named feature extractors. Unknown feature names log warnings but don't fail (extensibility).
- `cbr.resolution-strategies`: string identifiers. **Not validated at build time** — there is no named registry of `TypedFaultPolicy` implementations today (`TypedFaultPolicy` is an interface with anonymous implementations created by `YamlFaultPolicyBuilder.createTemplateTierAction()`). Validated at runtime when CBR proposes a strategy — unknown strategy names log a warning and fall through to tier-based escalation.
- `ras.situations`: each situation is validated against `SituationDefinition` record constraints (non-null situationId, non-empty eventTypes, non-null chainMode, non-null triggerAction). For `create-case` trigger actions, `CaseTriggerConfig` requires three non-null fields: `caseNamespace`, `caseName`, `caseVersion`.

### OQ2: Plugin versioning and vendor API drift

The spec doesn't address what happens when a vendor API changes. Plugin YAML references specific endpoints and response shapes — a Cloudflare API v4 → v5 migration would break plugins silently.

**Needs:**
- `plugin.version` field (semver) — already in the schema
- `plugin.api-version` field — the vendor API version this plugin targets
- Compatibility check at build time: warn if the deployed vendor API version doesn't match
- Migration path: the existing `NodeSpecRegistry` (EXISTS) enforces type string uniqueness
  (`resolve()` throws on unknown types). Supporting multiple versions of the same nodeType
  requires extending the registry to accept a `(typeString, apiVersion)` compound key. This
  is a new capability — the registry currently uses `Map<String, Class<? extends NodeSpec>>`.

### OQ3: Vendor-specific error classification

A Cloudflare 429 (rate limit) and a K8s 429 (admission webhook rejection) have different semantics. The Layer 2 `retry` primitive uses HTTP status codes, which are ambiguous across vendors.

**Proposed:** Optional `error-classifier` section per plugin that maps vendor-specific error responses to semantic categories. This lives at the transport layer (Layer 2), keeping the graph-level fault policy (Layer 3) clean:

```yaml
error-classifier:
  rules:
    - match:
        status: 429
        body-contains: "rate limit"
      fault: rate-limited
      retryable: true
      backoff: exponential
    - match:
        status: 429
        body-contains: "admission webhook"
      fault: rejected
      retryable: false
```

This replaces the flat `retryable: [429]` with semantic classification. The CBR learning surface can then distinguish between "rate-limited and retried successfully" vs "rejected and escalated."

## Issue #87 Requirements

From [casehubio/casehub-ops#87](https://github.com/casehubio/casehub-ops/issues/87):

| Requirement | Spec Section | Status |
|---|---|---|
| Java primitives (RestClient, GraphQlClient, AuthProvider, etc.) | §Layer 1 | Designed |
| YAML primitives (rest-call, graphql-call, json-extract, etc.) | §Layer 2 | Designed |
| 7 production plugins + K8s syntax reference (Cloudflare, Supabase, Fly.io, OCI, LE, K8s Secret) | §Plugin Catalogue | Designed |
| Plugin YAML schema (2 required + 4 optional sections) | §Layer 3 | Designed |
| CBR integration per plugin | §Layer 3 CBR section | Designed |
| RAS integration per plugin | §Layer 3 RAS section | Designed |
| Testing (Kind-on-Podman, WireMock, real free-tier accounts) | §Testing Strategy | Designed |
| Cross-repo architecture (desiredstate + ops) | §Architecture | Designed |

## References

- `casehub-desiredstate` — YamlDesiredStateProcessor (EXISTS), NodeSpecFactory (EXISTS), NodeSpecRegistry (EXISTS), CbrFaultPolicy (EXISTS), ThresholdFaultPolicy (EXISTS), YamlFaultPolicy (EXISTS), YamlGraph (EXISTS), VariableResolver (EXISTS)
- `casehub-ops/deployment` — DeploymentNodeProvisioner handler pattern (EXISTS), InfraBackend delegation (EXISTS)
- `casehub-ops/infra` — InfraWrappingFactory (EXISTS, NodeSpecFactory reference implementation)
- `casehub-ras-api` — SituationDefinition (EXISTS), ChainMode (EXISTS), TriggerAction (EXISTS), CaseTriggerConfig (EXISTS)
- `docs/research/2026-09-08-self-healing-self-governing-infrastructure.md` — gap analysis, MAPE-K mapping
- `docs/specs/issue-116-yaml-language-design/` — YAML language extensions design, interpolation model
- Issue #86 — CrossplaneNodeProvisioner (future: delegate to Crossplane for provisioning)
- Issue #87 — this work
