# YAML-First Plugin Architecture — Design Spec

## Overview

A three-layer architecture enabling infrastructure plugins to be authored primarily in YAML, with Java as the escape hatch for transport and protocol concerns. Each plugin is a complete self-healing unit: it provisions a resource, detects its own problems, resolves them using CBR-guided strategies, and learns from outcomes.

**Repos:** casehub-desiredstate (domain-agnostic primitives), casehub-ops (domain-specific plugins)

**Key constraint:** Type safety throughout. Java records are canonical; JSON Schema is generated for IDE validation and refactoring. Every reference between YAML files is typed and resolvable.

## Architecture

### Three Layers

```
┌─────────────────────────────────────────────────────┐
│  Layer 3: YAML Plugins (casehub-ops/extensions/)    │
│  k8s-deployment.yaml, cloudflare-dns.yaml, ...      │
│  Each: actual-state + provisioner + fault-policy     │
│        + cbr + ras                                   │
├─────────────────────────────────────────────────────┤
│  Layer 2: YAML Primitives (casehub-desiredstate/    │
│           extensions/yaml-primitives/)               │
│  rest-call, graphql-call, json-extract,              │
│  compare-state, paginate, poll-until, auth-ref,      │
│  retry                                               │
│  Compose over each other AND over Layer 1            │
├─────────────────────────────────────────────────────┤
│  Layer 1: Java SPIs (casehub-desiredstate/           │
│           extensions/)                               │
│  RestClient, GraphQlClient, AuthProvider,            │
│  JsonPathExtractor, StreamingStateSource,            │
│  RateLimiter, NodeSpecSchemaGenerator                │
└─────────────────────────────────────────────────────┘
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
        url: "{baseUrl}/apis/apps/v1/namespaces/{namespace}/deployments"
        auth: "{auth-ref}"
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

These are transport and computation primitives that YAML cannot express.

### api-client (shared)

| SPI | Purpose | Key Methods |
|-----|---------|------------|
| `AuthProvider` | Authentication for any API | `authenticate(SecretRef) → HttpHeaders` |
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

Runs at Quarkus build time via `YamlDesiredStateProcessor`. Feeds IDE plugins for validation and autocomplete.

### streaming

| SPI | Purpose | Key Methods |
|-----|---------|------------|
| `StreamingStateSource` | WebSocket/SSE for live state | `subscribe(StreamConfig) → Flow<StateEvent>` |

For resources that push state changes (K8s watch API, CloudEvents). Falls back to polling when streaming unavailable.

## Layer 2: YAML Primitives

Composable YAML operations backed by Layer 1 Java SPIs.

### rest-call

```yaml
rest-call:
  method: POST | GET | PUT | DELETE | PATCH
  url: "{baseUrl}/path/{variable}"
  auth: "{auth-ref}"
  headers:
    Content-Type: application/json
  body:
    field: "{spec.field}"
  expect: 200          # expected status code
  extract:
    nodeId: "$.result.id"
```

### graphql-call

```yaml
graphql-call:
  endpoint: "{baseUrl}/graphql"
  auth: "{auth-ref}"
  query: |
    mutation CreateRecord($input: RecordInput!) {
      createRecord(input: $input) { id status }
    }
  variables:
    input:
      name: "{spec.name}"
      type: "{spec.recordType}"
  extract:
    nodeId: "$.data.createRecord.id"
```

### json-extract

```yaml
json-extract:
  source: "{response}"
  mappings:
    nodeId: "$.id"
    status: "$.status"
    ip: "$.addresses[0].ip"
```

### compare-state

```yaml
compare-state:
  fields: [name, replicas, image, resources]
  ignore: [metadata.resourceVersion, status.observedGeneration]
  tolerance:
    cpu: 10%      # allow 10% drift before flagging
```

### paginate

Composes over `rest-call`:

```yaml
paginate:
  strategy: cursor | offset | link-header | relay
  rest-call: { ... }    # the call to repeat
  cursor-path: "$.metadata.continue"   # for cursor strategy
  results-path: "$.items"
  page-size: 100
```

### poll-until

Composes over `rest-call`:

```yaml
poll-until:
  rest-call:
    method: GET
    url: "{baseUrl}/operations/{operationId}"
  condition: "$.status == 'READY'"
  interval: 5s
  timeout: 5m
  backoff: linear | exponential
```

### auth-ref

```yaml
auth-ref:
  type: bearer-token | oauth2 | api-key | basic
  secret-ref: my-secret-name     # K8s secret or Vault path
  # oauth2-specific
  token-url: "https://auth.example.com/oauth/token"
  client-id-ref: oauth-client-id
  scopes: [read, write]
```

### retry

```yaml
retry:
  max-attempts: 3
  backoff: exponential
  initial-delay: 1s
  max-delay: 30s
  retryable: [429, 500, 502, 503]
  fatal: [401, 403, 404]
```

### Composability: YAML over YAML

`paginate` composes over `rest-call`. `poll-until` composes over `rest-call`. A plugin's `actual-state` composes over `paginate` which composes over `rest-call`. Three layers of YAML, Java only at the bottom.

A plugin can also compose primitives in sequence:

```yaml
provisioner:
  create:
    - rest-call:         # step 1: create the resource
        method: POST
        url: "{baseUrl}/resources"
        body: { name: "{spec.name}" }
        extract: { operationId: "$.operation.id" }
    - poll-until:        # step 2: wait for it to be ready
        rest-call:
          method: GET
          url: "{baseUrl}/operations/{operationId}"
        condition: "$.status == 'READY'"
        interval: 5s
        timeout: 5m
```

## Layer 3: Plugin Schema

Each plugin YAML has five required sections. Schema validation rejects plugins missing any section.

### Complete Plugin Example — KubernetesDeploymentSpec

```yaml
plugin:
  name: k8s-deployment
  version: 1.0
  nodeType: k8s/deployment

auth:
  type: bearer-token
  secret-ref: k8s-service-account-token

defaults:
  baseUrl: "https://kubernetes.default.svc"
  namespace: default

# ── Section 1: Actual State ──────────────────────────

actual-state:
  list:
    paginate:
      strategy: link-header
      rest-call:
        method: GET
        url: "{baseUrl}/apis/apps/v1/namespaces/{namespace}/deployments"
        auth: "{auth-ref}"
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

provisioner:
  create:
    rest-call:
      method: POST
      url: "{baseUrl}/apis/apps/v1/namespaces/{namespace}/deployments"
      auth: "{auth-ref}"
      body:
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: "{spec.name}"
          labels: "{spec.labels}"
        spec:
          replicas: "{spec.replicas}"
          selector:
            matchLabels: "{spec.labels}"
          template:
            metadata:
              labels: "{spec.labels}"
            spec:
              containers:
                - name: "{spec.name}"
                  image: "{spec.image}"
                  resources: "{spec.resources}"
      extract:
        nodeId: "$.metadata.name"

  update:
    rest-call:
      method: PUT
      url: "{baseUrl}/apis/apps/v1/namespaces/{namespace}/deployments/{nodeId}"
      auth: "{auth-ref}"
      body: "{merge:create.body}"

  delete:
    rest-call:
      method: DELETE
      url: "{baseUrl}/apis/apps/v1/namespaces/{namespace}/deployments/{nodeId}"
      auth: "{auth-ref}"

# ── Section 3: Fault Policy ──────────────────────────

fault-policy:
  retryable: [429, 500, 502, 503, 504]
  retry:
    max-attempts: 3
    backoff: exponential
    initial-delay: 2s
  fatal: [401, 403, 409]
  tiers:
    - threshold: 3
      action: restart-pod
    - threshold: 5
      action: escalate-human
      human-gating: required

# ── Section 4: CBR — Learning Surface ────────────────

cbr:
  case-features:
    - error-type          # OOMKilled, CrashLoopBackOff, ImagePullBackOff
    - resource-utilisation # CPU/memory percentage at failure time
    - time-of-day         # peak vs off-peak
    - replica-count       # how many replicas were running
    - previous-image      # the image before the failing one
  resolution-strategies:
    - restart-pod         # delete pod, let deployment recreate
    - rollback-image      # revert to previous known-good image
    - scale-horizontal    # increase replica count
    - scale-vertical      # increase resource limits
    - cordon-node         # mark K8s node unschedulable
  outcome-signals:
    - pod-healthy-after: 5m       # pod stays healthy for 5 minutes
    - no-restart-within: 30m      # no restarts in 30 minutes
    - ready-replicas-match: true  # readyReplicas == desiredReplicas

# ── Section 5: RAS — Detection Situations ────────────

ras:
  situations:
    - id: pod-crash-loop
      description: "Pod restarting repeatedly"
      eventTypes: [k8s.pod.restart]
      chainMode:
        type: count
        ganglionId: restart-counter
        requiredCount: 3
      correlationWindow: PT10M
      triggerAction:
        type: create-case

    - id: memory-pressure
      description: "Memory usage trending toward OOM"
      eventTypes: [k8s.metrics.memory]
      chainMode:
        type: threshold
        ganglionId: mem-usage
        minConfidence: 0.85
      correlationWindow: PT5M
      triggerAction:
        type: notify-only

    - id: image-pull-failure
      description: "Container image cannot be pulled"
      eventTypes: [k8s.pod.image-pull-failed]
      chainMode:
        type: count
        ganglionId: pull-failure
        requiredCount: 1
      correlationWindow: PT2M
      triggerAction:
        type: create-case

    - id: replica-unavailable
      description: "Ready replicas below desired for sustained period"
      eventTypes: [k8s.deployment.replica-unavailable]
      chainMode:
        type: streak
        ganglionId: unavailable-streak
        requiredCount: 3
      correlationWindow: PT15M
      triggerAction:
        type: create-case
```

## Plugin Catalogue — 10 Plugins

| # | Plugin | nodeType | API Protocol | CaseHub Integration |
|---|--------|----------|-------------|---------------------|
| 1 | KubernetesDeployment | k8s/deployment | REST (K8s API) | Trust on cluster health; Engine case for degradation |
| 2 | KubernetesService | k8s/service | REST (K8s API) | Blocks summarisation: endpoint events → service health |
| 3 | KubernetesIngress | k8s/ingress | REST (K8s API) | Pre-emptive: cert renewal 30 days before expiry |
| 4 | KubernetesSecret | k8s/secret | REST (K8s API) | Ledger: tamper-evident rotation audit trail |
| 5 | CloudflareDns | cloudflare/dns-record | REST (Cloudflare API) | Trust on propagation speed vs other DNS providers |
| 6 | CloudflareWorker | cloudflare/worker | REST (Cloudflare API) | CBR: cold start avoidance strategies |
| 7 | SupabaseDatabase | supabase/database | REST (Supabase API) | Pre-emptive: storage trending toward free-tier limit |
| 8 | FlyIoMachine | flyio/machine | REST (Fly.io API) | Trust: region reliability scores from CBR history |
| 9 | OracleCloudVm | oci/vm-instance | REST (OCI API) | Pre-emptive: free-tier usage approaching limits |
| 10 | LetsEncryptCert | letsencrypt/certificate | REST (ACME) | Ledger: certificate lifecycle audit trail |

## Type Safety and IDE Support

### Build-Time Validation Pipeline

```
Java records (@NodeTypeId) → Jandex scan at build time
    → NodeSpecRegistry (type string → class)
    → NodeSpecSchemaGenerator (class → JSON Schema)
    → YAML validation against schema
    → Build fails on type mismatch
```

### Plugin YAML Validation

The Quarkus build-time extension (`YamlDesiredStateProcessor`) validates plugins:
- `nodeType` resolves to a registered `@NodeTypeId`
- `spec.field` references in templates resolve to fields on the NodeSpec record
- `auth-ref` resolves to a declared auth block
- `cbr.case-features` and `cbr.resolution-strategies` are non-empty
- `ras.situations` has at least one situation defined
- `compare-state.fields` reference valid NodeSpec fields

### IDE Plugin Contract

JSON Schema per plugin enables:
- **Autocomplete:** `spec.` triggers field list for the declared nodeType
- **Validation:** red squiggle on `spec.nonexistent`
- **Rename:** rename a NodeSpec field in Java → schema regenerates → IDE flags all YAML files using the old name
- **Navigate to definition:** `nodeType: k8s/deployment` → jump to `KubernetesDeploymentSpec.java`

IntelliJ and VS Code plugins consume the generated JSON Schema. Plugin development is a future phase.

## Testing Strategy

| Plugin type | Local test | Integration test |
|-------------|-----------|-----------------|
| K8s (1-4) | Kind on Podman | Real K8s cluster (CI) |
| Cloudflare (5-6) | WireMock container | Real Cloudflare free account |
| Supabase (7) | WireMock container | Real Supabase free project |
| Fly.io (8) | WireMock container | Real Fly.io free account |
| Oracle Cloud (9) | WireMock container | Real OCI always-free |
| Let's Encrypt (10) | Pebble (ACME test server) | Let's Encrypt staging |

All local tests run on Podman. No Docker dependency.

## Cross-Vendor Topology Example

A topology YAML composing plugins from multiple vendors:

```yaml
modules:
  - k8s-deployment-plugin
  - k8s-service-plugin
  - k8s-ingress-plugin
  - cloudflare-dns-plugin
  - supabase-database-plugin

nodes:
  - spec:
      nodeType: supabase/database
      name: myapp-db
      region: eu-west-1
      plan: free
    dependsOn: []

  - spec:
      nodeType: k8s/deployment
      name: api-server
      image: myapp/api:latest
      replicas: 3
      env:
        DATABASE_URL: "{supabase-db.connectionString}"
    dependsOn: [myapp-db]

  - spec:
      nodeType: k8s/service
      name: api-service
      port: 8080
      targetPort: 8080
    dependsOn: [api-server]

  - spec:
      nodeType: k8s/ingress
      name: api-ingress
      host: api.myapp.com
      service: api-service
      tls: true
    dependsOn: [api-service]

  - spec:
      nodeType: cloudflare/dns-record
      name: api.myapp.com
      recordType: CNAME
      target: "{api-ingress.loadBalancerHostname}"
    dependsOn: [api-ingress]

lifecycle:
  phases:
    - name: data-tier
      nodes: [myapp-db]
    - name: compute-tier
      nodes: [api-server, api-service]
    - name: edge-tier
      nodes: [api-ingress, api.myapp.com]
```

The reconciliation loop manages all five resources across three vendors continuously. CBR learns per-vendor reliability. RAS detects cross-vendor degradation. Faults in one vendor can trigger adaptation in others.

## References

- `casehub-desiredstate` — YamlDesiredStateProcessor, NodeSpecFactory, NodeSpecRegistry, CbrFaultPolicy, YamlFaultPolicy
- `casehub-ops/deployment` — DeploymentNodeProvisioner handler pattern, InfraBackend delegation
- `casehub-ops/infra` — InfraWrappingFactory (NodeSpecFactory reference implementation)
- `casehub-ras-api` — SituationDefinition, ChainMode, TriggerAction
- `docs/research/2026-09-08-self-healing-self-governing-infrastructure.md` — gap analysis, MAPE-K mapping
- Issue #86 — CrossplaneNodeProvisioner (future: delegate to Crossplane for provisioning)
- Issue #87 — this work
