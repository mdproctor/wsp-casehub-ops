# YAML-First Plugin Architecture Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #87 — YAML-first plugin architecture
**Issue group:** #87

**Goal:** Enable infrastructure plugins to be authored in YAML with CBR learning and RAS detection inline, compiling to the existing SPI Quad at Quarkus build time.

**Architecture:** Three-layer stack — Java transport SPIs (Layer 1) in casehub-desiredstate, composable YAML primitives (Layer 2) in casehub-desiredstate, and domain-specific plugins (Layer 3) in casehub-ops. Plugin YAML compiles to ActualStateAdapter + NodeProvisioner + ThresholdFaultPolicy + CBR metadata + SituationDefinition registrations. Extends the existing YamlGraph format — does not replace it.

**Tech Stack:** Quarkus 3.x, Jackson YAML, Jayway JSONPath, JQ (via existing JQExpressionEvaluator), Jandex, Quarkus build-time extensions, fabric8 (K8s), WireMock (testing)

## Global Constraints

- Java records are canonical; YAML is the authoring surface
- Uses existing `${prefix.name}` interpolation model (VariableResolver)
- Plugin YAML compiles to SPI Quad implementations at Quarkus build time
- No parallel YAML language — extends existing YamlGraph/YamlDesiredStateProcessor
- `nodeType` must resolve to a registered `@NodeTypeId`
- Layer 1+2 go in casehub-desiredstate; Layer 3 plugins go in casehub-ops
- All tests run on Podman (Kind for K8s, WireMock for vendor APIs)

---

## Batch 1: Layer 1 Java SPIs — Transport Foundation [desiredstate]

The Java primitives that YAML cannot express. All in casehub-desiredstate.

### Task 1: AuthProvider SPI + implementations

**Files:**
- Create: `yaml/spi/src/main/java/io/casehub/desiredstate/yaml/spi/auth/AuthProvider.java`
- Create: `yaml/spi/src/main/java/io/casehub/desiredstate/yaml/spi/auth/AuthSession.java`
- Create: `yaml/spi/src/main/java/io/casehub/desiredstate/yaml/spi/auth/SecretRef.java`
- Create: `yaml/spi/src/main/java/io/casehub/desiredstate/yaml/spi/auth/BearerTokenAuth.java`
- Create: `yaml/spi/src/main/java/io/casehub/desiredstate/yaml/spi/auth/ApiKeyAuth.java`
- Create: `yaml/spi/src/main/java/io/casehub/desiredstate/yaml/spi/auth/BasicAuth.java`
- Test: `yaml/spi/src/test/java/io/casehub/desiredstate/yaml/spi/auth/AuthProviderTest.java`

**Interfaces:**
- Produces: `AuthProvider.authenticate(SecretRef) → AuthSession` (headers + TTL + refresh callback)
- Produces: `AuthSession` record (headers, expiresAt, refreshFn)

- [ ] **Step 1:** Write failing tests for BearerTokenAuth — authenticate returns headers with Authorization: Bearer
- [ ] **Step 2:** Run tests, verify fail
- [ ] **Step 3:** Implement AuthProvider SPI + BearerTokenAuth
- [ ] **Step 4:** Run tests, verify pass
- [ ] **Step 5:** Add ApiKeyAuth + BasicAuth with tests
- [ ] **Step 6:** Commit: `feat: AuthProvider SPI — bearer, api-key, basic auth Refs #87`

### Task 2: RestClient SPI + JsonPathExtractor

**Files:**
- Create: `yaml/spi/src/main/java/io/casehub/desiredstate/yaml/spi/rest/RestClient.java`
- Create: `yaml/spi/src/main/java/io/casehub/desiredstate/yaml/spi/rest/RestRequest.java`
- Create: `yaml/spi/src/main/java/io/casehub/desiredstate/yaml/spi/rest/RestResponse.java`
- Create: `yaml/spi/src/main/java/io/casehub/desiredstate/yaml/spi/rest/QuarkusRestClient.java`
- Create: `yaml/spi/src/main/java/io/casehub/desiredstate/yaml/spi/extract/JsonPathExtractor.java`
- Test: `yaml/spi/src/test/java/io/casehub/desiredstate/yaml/spi/rest/RestClientTest.java`
- Test: `yaml/spi/src/test/java/io/casehub/desiredstate/yaml/spi/extract/JsonPathExtractorTest.java`

**Interfaces:**
- Consumes: `AuthProvider` (Task 1)
- Produces: `RestClient.execute(RestRequest) → RestResponse`
- Produces: `JsonPathExtractor.extract(JsonNode, Map<String, String>) → Map<String, Object>`

- [ ] **Step 1:** Write failing test for JsonPathExtractor — extract fields from a JSON document using JSONPath
- [ ] **Step 2:** Implement JsonPathExtractor using Jayway JSONPath
- [ ] **Step 3:** Write failing test for RestClient — execute GET returns response with status and body
- [ ] **Step 4:** Implement QuarkusRestClient using Quarkus REST client
- [ ] **Step 5:** Integration test: RestClient + AuthProvider + WireMock
- [ ] **Step 6:** Commit: `feat: RestClient SPI + JsonPathExtractor Refs #87`

### Task 3: PaginationStrategy + RetryPolicy + RateLimiter

**Files:**
- Create: `yaml/spi/src/main/java/io/casehub/desiredstate/yaml/spi/rest/PaginationStrategy.java`
- Create: `yaml/spi/src/main/java/io/casehub/desiredstate/yaml/spi/rest/CursorPagination.java`
- Create: `yaml/spi/src/main/java/io/casehub/desiredstate/yaml/spi/rest/LinkHeaderPagination.java`
- Create: `yaml/spi/src/main/java/io/casehub/desiredstate/yaml/spi/rest/OffsetPagination.java`
- Create: `yaml/spi/src/main/java/io/casehub/desiredstate/yaml/spi/retry/RetryPolicy.java`
- Create: `yaml/spi/src/main/java/io/casehub/desiredstate/yaml/spi/retry/ExponentialBackoff.java`
- Create: `yaml/spi/src/main/java/io/casehub/desiredstate/yaml/spi/ratelimit/RateLimiter.java`
- Test: corresponding test files

**Interfaces:**
- Consumes: `RestClient`, `RestResponse` (Task 2)
- Produces: `PaginationStrategy.nextPage(RestResponse) → Optional<RestRequest>`
- Produces: `RetryPolicy.shouldRetry(statusCode, attempt) → RetryDecision`
- Produces: `RateLimiter.acquire(endpoint) → Permit`

- [ ] **Step 1:** Write tests for CursorPagination — extracts cursor from response, builds next request
- [ ] **Step 2:** Implement CursorPagination, LinkHeaderPagination, OffsetPagination
- [ ] **Step 3:** Write tests for RetryPolicy — exponential backoff, retryable vs fatal codes
- [ ] **Step 4:** Implement RetryPolicy + ExponentialBackoff
- [ ] **Step 5:** Write tests for RateLimiter — token bucket, blocks when exhausted
- [ ] **Step 6:** Implement RateLimiter
- [ ] **Step 7:** Commit: `feat: PaginationStrategy + RetryPolicy + RateLimiter Refs #87`

### Task 4: GraphQlClient SPI

**Files:**
- Create: `yaml/spi/src/main/java/io/casehub/desiredstate/yaml/spi/graphql/GraphQlClient.java`
- Create: `yaml/spi/src/main/java/io/casehub/desiredstate/yaml/spi/graphql/GraphQlRequest.java`
- Create: `yaml/spi/src/main/java/io/casehub/desiredstate/yaml/spi/graphql/GraphQlResponse.java`
- Create: `yaml/spi/src/main/java/io/casehub/desiredstate/yaml/spi/graphql/QuarkusGraphQlClient.java`
- Create: `yaml/spi/src/main/java/io/casehub/desiredstate/yaml/spi/graphql/RelayPagination.java`
- Test: corresponding test files

**Interfaces:**
- Consumes: `AuthProvider` (Task 1), `RestClient` (Task 2 — GraphQL over HTTP)
- Produces: `GraphQlClient.execute(GraphQlRequest) → GraphQlResponse`
- Produces: `RelayPagination.nextPage(GraphQlResponse, connectionField) → Optional<GraphQlRequest>`

- [ ] **Step 1:** Write failing test — execute query returns typed response
- [ ] **Step 2:** Implement QuarkusGraphQlClient (GraphQL over HTTP POST using RestClient)
- [ ] **Step 3:** Write tests for RelayPagination — Relay cursor convention
- [ ] **Step 4:** Implement RelayPagination
- [ ] **Step 5:** Integration test with WireMock GraphQL endpoint
- [ ] **Step 6:** Commit: `feat: GraphQlClient SPI + RelayPagination Refs #87`

---

## Batch 2: Layer 2 YAML Primitives — Composable Operations [desiredstate]

YAML operations backed by Layer 1. Each primitive is a YAML-parseable record that composes over Layer 1 SPIs.

### Task 5: Plugin YAML parser + rest-call primitive

**Files:**
- Create: `yaml/primitives/src/main/java/io/casehub/desiredstate/yaml/primitives/RestCallPrimitive.java`
- Create: `yaml/primitives/src/main/java/io/casehub/desiredstate/yaml/primitives/RestCallExecutor.java`
- Create: `yaml/primitives/src/main/java/io/casehub/desiredstate/yaml/primitives/PluginYamlParser.java`
- Create: `yaml/primitives/src/main/java/io/casehub/desiredstate/yaml/primitives/PluginDescriptor.java`
- Test: corresponding test files

**Interfaces:**
- Consumes: `RestClient`, `AuthProvider`, `JsonPathExtractor` (Batch 1)
- Produces: `RestCallExecutor.execute(RestCallPrimitive, VariableScope) → RestResponse`
- Produces: `PluginYamlParser.parse(InputStream) → PluginDescriptor`

- [ ] **Step 1:** Write failing test — parse a minimal plugin YAML into PluginDescriptor
- [ ] **Step 2:** Implement PluginYamlParser + PluginDescriptor records
- [ ] **Step 3:** Write failing test — RestCallExecutor executes a rest-call with variable interpolation
- [ ] **Step 4:** Implement RestCallExecutor using RestClient + VariableResolver
- [ ] **Step 5:** Write test — rest-call with auth-ref, body template, extract
- [ ] **Step 6:** Commit: `feat: plugin YAML parser + rest-call primitive Refs #87`

### Task 6: Remaining YAML primitives (json-extract, compare-state, paginate, poll-until, retry)

**Files:**
- Create: `yaml/primitives/src/main/java/io/casehub/desiredstate/yaml/primitives/JsonExtractPrimitive.java`
- Create: `yaml/primitives/src/main/java/io/casehub/desiredstate/yaml/primitives/CompareStatePrimitive.java`
- Create: `yaml/primitives/src/main/java/io/casehub/desiredstate/yaml/primitives/PaginatePrimitive.java`
- Create: `yaml/primitives/src/main/java/io/casehub/desiredstate/yaml/primitives/PollUntilPrimitive.java`
- Create: `yaml/primitives/src/main/java/io/casehub/desiredstate/yaml/primitives/RetryPrimitive.java`
- Create: executor classes for each
- Test: corresponding test files

**Interfaces:**
- Consumes: `RestCallExecutor` (Task 5), `JsonPathExtractor` (Task 2), `PaginationStrategy` (Task 3)
- Produces: `PaginateExecutor.execute() → List<Map<String, Object>>`
- Produces: `CompareStateExecutor.compare(desired, actual) → DriftResult`
- Produces: `PollUntilExecutor.poll() → PollResult`

- [ ] **Step 1:** Write tests for CompareStatePrimitive — field comparison with tolerance
- [ ] **Step 2:** Implement compare-state with percentage and absolute tolerance
- [ ] **Step 3:** Write tests for PaginatePrimitive — composes over rest-call, cursor/link-header
- [ ] **Step 4:** Implement paginate composing over RestCallExecutor
- [ ] **Step 5:** Write tests for PollUntilPrimitive — polls until condition met or timeout
- [ ] **Step 6:** Implement poll-until composing over RestCallExecutor
- [ ] **Step 7:** Write tests for RetryPrimitive — exponential backoff, retryable codes, on-401 refresh
- [ ] **Step 8:** Implement retry wrapping RestCallExecutor
- [ ] **Step 9:** Composability integration test — paginate(rest-call) end-to-end with WireMock
- [ ] **Step 10:** Commit: `feat: YAML primitives — compare-state, paginate, poll-until, retry Refs #87`

### Task 7: graphql-call primitive + error-classifier

**Files:**
- Create: `yaml/primitives/src/main/java/io/casehub/desiredstate/yaml/primitives/GraphQlCallPrimitive.java`
- Create: `yaml/primitives/src/main/java/io/casehub/desiredstate/yaml/primitives/GraphQlCallExecutor.java`
- Create: `yaml/primitives/src/main/java/io/casehub/desiredstate/yaml/primitives/ErrorClassifier.java`
- Test: corresponding test files

**Interfaces:**
- Consumes: `GraphQlClient` (Task 4)
- Produces: `GraphQlCallExecutor.execute(GraphQlCallPrimitive, VariableScope) → GraphQlResponse`
- Produces: `ErrorClassifier.classify(statusCode, body) → ErrorClassification`

- [ ] **Step 1:** Write tests for GraphQlCallExecutor — query with variables, extract
- [ ] **Step 2:** Implement GraphQlCallExecutor
- [ ] **Step 3:** Write tests for ErrorClassifier — vendor-specific error mapping
- [ ] **Step 4:** Implement ErrorClassifier (body-contains matching, semantic categories)
- [ ] **Step 5:** Commit: `feat: graphql-call primitive + error-classifier Refs #87`

---

## Batch 3: Plugin Compiler — YAML to SPI Quad [desiredstate]

Extends YamlDesiredStateProcessor to compile plugin YAML into SPI implementations at build time.

### Task 8: Plugin discovery + NodeSpec validation

**Files:**
- Modify: `deployment/src/main/java/.../YamlDesiredStateProcessor.java` — add plugin discovery step
- Create: `yaml/compiler/src/main/java/io/casehub/desiredstate/yaml/compiler/PluginCompiler.java`
- Create: `yaml/compiler/src/main/java/io/casehub/desiredstate/yaml/compiler/PluginValidation.java`
- Test: corresponding test files

**Interfaces:**
- Consumes: `PluginDescriptor` (Task 5), `NodeSpecRegistry` (EXISTS)
- Produces: `PluginCompiler.compile(PluginDescriptor) → CompiledPlugin`
- Produces: Build-time validation errors for invalid nodeType, ${spec.*} references

- [ ] **Step 1:** Write failing test — discover plugin YAML files on classpath under `plugins/`
- [ ] **Step 2:** Implement plugin discovery in YamlDesiredStateProcessor
- [ ] **Step 3:** Write failing test — validate nodeType resolves to registered @NodeTypeId
- [ ] **Step 4:** Write failing test — validate ${spec.field} references resolve to NodeSpec record fields via Jandex
- [ ] **Step 5:** Implement PluginValidation with Jandex introspection
- [ ] **Step 6:** Write failing test — build fails on duplicate nodeType (YAML plugin vs Java provisioner)
- [ ] **Step 7:** Implement duplicate detection
- [ ] **Step 8:** Commit: `feat: plugin compiler — discovery + validation Refs #87`

### Task 9: Generated ActualStateAdapter + NodeProvisioner

**Files:**
- Create: `yaml/compiler/src/main/java/.../GeneratedActualStateAdapter.java`
- Create: `yaml/compiler/src/main/java/.../GeneratedNodeProvisioner.java`
- Create: `yaml/compiler/src/main/java/.../PluginVariableSourceFactory.java`
- Test: corresponding test files

**Interfaces:**
- Consumes: `PluginDescriptor`, `RestCallExecutor`, `CompareStateExecutor` (Batch 2)
- Produces: `ActualStateAdapter` implementation (handledTypes + readActual with drift detection)
- Produces: `NodeProvisioner` implementation (handledTypes + provision/deprovision via rest-call)

- [ ] **Step 1:** Write failing test — GeneratedActualStateAdapter reads state from WireMock endpoint using plugin's actual-state section
- [ ] **Step 2:** Implement GeneratedActualStateAdapter using RestCallExecutor + compare-state
- [ ] **Step 3:** Write failing test — GeneratedNodeProvisioner provisions via create/update/delete rest-calls
- [ ] **Step 4:** Implement GeneratedNodeProvisioner using RestCallExecutor
- [ ] **Step 5:** Write test — plugin variable scopes (${plugin.*}, ${spec.*}, ${response.*}, ${step.N.*})
- [ ] **Step 6:** Implement PluginVariableSourceFactory registering prefixes with VariableResolver
- [ ] **Step 7:** Integration test — full cycle: read actual state → detect drift → provision → verify
- [ ] **Step 8:** Commit: `feat: generated ActualStateAdapter + NodeProvisioner from plugin YAML Refs #87`

### Task 10: Fault policy + CBR + RAS compilation

**Files:**
- Create: `yaml/compiler/src/main/java/.../PluginFaultPolicyCompiler.java`
- Create: `yaml/compiler/src/main/java/.../PluginCbrCompiler.java`
- Create: `yaml/compiler/src/main/java/.../PluginRasCompiler.java`
- Create: `yaml/compiler/src/main/java/.../PluginApprovalCompiler.java`
- Test: corresponding test files

**Interfaces:**
- Consumes: `PluginDescriptor`, existing ThresholdFaultPolicy, CbrFaultPolicy, SituationDefinition
- Produces: ThresholdFaultPolicy.Tier entries from fault-policy section
- Produces: CBR context-feature metadata from cbr section
- Produces: SituationDefinition registrations from ras.situations
- Produces: ApprovalEvaluator from approval section

- [ ] **Step 1:** Write tests for PluginFaultPolicyCompiler — tiers compile to ThresholdFaultPolicy.Tier entries
- [ ] **Step 2:** Implement fault policy compilation with FaultType enum validation
- [ ] **Step 3:** Write tests for PluginRasCompiler — situations compile to SituationDefinition records
- [ ] **Step 4:** Implement RAS compilation with ganglion auto-prefixing and JQ expression validation
- [ ] **Step 5:** Write tests for PluginCbrCompiler — context-features and resolution-strategies recorded as metadata
- [ ] **Step 6:** Implement CBR metadata compilation (enriches CbrFaultPolicy retrieval context)
- [ ] **Step 7:** Write tests for PluginApprovalCompiler — risk levels compile to ApprovalEvaluator
- [ ] **Step 8:** Implement approval compilation with namespace-pattern overrides
- [ ] **Step 9:** Integration test — compile a complete plugin YAML → all 5 SPI contributions generated
- [ ] **Step 10:** Commit: `feat: plugin compiler — fault-policy + CBR + RAS + approval compilation Refs #87`

---

## Batch 4: First Plugin — K8s Secret [ops]

The simplest production plugin. Uses existing K8s API patterns. Proves the full stack end-to-end.

### Task 11: K8sSecretSpec sealed variant + NodeSpecFactory

**Files:**
- Create: `api/src/main/java/io/casehub/ops/api/infra/K8sSecretSpec.java` (sealed variant of InfraNodeSpec)
- Modify: `api/src/main/java/io/casehub/ops/api/infra/InfraNodeSpec.java` — add K8sSecretSpec to permits
- Modify: `infra/src/main/java/.../InfraNodeSpecFactoryProvider.java` — register K8sSecretSpec factory
- Test: `api/src/test/java/io/casehub/ops/api/infra/K8sSecretSpecTest.java`

**Interfaces:**
- Produces: `K8sSecretSpec` record with fields: name, namespace, type, data, labels
- Produces: Factory registration for nodeType `k8s_secret`

- [ ] **Step 1:** Write failing test — K8sSecretSpec record with required fields
- [ ] **Step 2:** Implement K8sSecretSpec as InfraNodeSpec sealed variant with @NodeTypeId("k8s_secret")
- [ ] **Step 3:** Register in InfraNodeSpecFactoryProvider
- [ ] **Step 4:** Write test — factory creates K8sSecretSpec from YAML map
- [ ] **Step 5:** Commit: `feat: K8sSecretSpec sealed variant + factory Refs #87`

### Task 12: K8s Secret plugin YAML + end-to-end test

**Files:**
- Create: `plugins/k8s-secret-plugin.yaml` — complete plugin with all 6 sections
- Create: `plugins/src/test/java/.../K8sSecretPluginTest.java`
- Create: `plugins/src/test/java/.../K8sSecretPluginIT.java` — Kind-on-Podman integration test

**Interfaces:**
- Consumes: K8sSecretSpec (Task 11), plugin compiler (Batch 3), REST primitives (Batch 2)

- [ ] **Step 1:** Write the k8s-secret-plugin.yaml — actual-state, provisioner, approval, fault-policy, cbr, ras
- [ ] **Step 2:** Write unit test — plugin compiles without errors, nodeType resolves, ${spec.*} fields validate
- [ ] **Step 3:** Write unit test — generated ActualStateAdapter reads K8s secrets from WireMock
- [ ] **Step 4:** Write unit test — generated NodeProvisioner creates/updates/deletes secrets via WireMock
- [ ] **Step 5:** Write unit test — fault-policy tiers compile to ThresholdFaultPolicy entries
- [ ] **Step 6:** Write unit test — RAS situations compile to SituationDefinition records
- [ ] **Step 7:** Write Kind-on-Podman integration test — full reconciliation cycle: create secret → read → drift → update
- [ ] **Step 8:** Commit: `feat: k8s-secret plugin — complete YAML plugin with end-to-end test Refs #87`

---

## Batch 5: Vendor Plugins — Cloudflare [ops]

Two plugins against real vendor APIs with WireMock for local testing.

### Task 13: CloudflareDnsRecordSpec + plugin YAML

**Files:**
- Create: `api/src/main/java/io/casehub/ops/api/infra/CloudflareDnsRecordSpec.java`
- Create: `plugins/cloudflare-dns-plugin.yaml`
- Modify: InfraNodeSpec permits, InfraNodeSpecFactoryProvider
- Test: unit + WireMock integration tests

- [ ] Steps: sealed variant → factory → plugin YAML → compile test → WireMock test → commit

### Task 14: CloudflareWorkerSpec + plugin YAML

**Files:**
- Create: `api/src/main/java/io/casehub/ops/api/infra/CloudflareWorkerSpec.java`
- Create: `plugins/cloudflare-worker-plugin.yaml`
- Test: unit + WireMock integration tests

- [ ] Steps: sealed variant → factory → plugin YAML → compile test → WireMock test → commit

---

## Batch 6: Vendor Plugins — Supabase, Fly.io, Oracle Cloud, Let's Encrypt [ops]

### Task 15: SupabaseDatabaseSpec + plugin YAML
### Task 16: FlyIoMachineSpec + plugin YAML
### Task 17: OracleCloudVmSpec + plugin YAML
### Task 18: LetsEncryptCertSpec + plugin YAML (uses Pebble ACME test server)

Each follows the same pattern: sealed variant → factory → plugin YAML → compile test → WireMock/Pebble test → commit.

---

## Batch 7: Schema Generation + Cross-Vendor Topology Test [desiredstate + ops]

### Task 19: NodeSpecSchemaGenerator

**Files:**
- Create: `yaml/spi/src/main/java/.../NodeSpecSchemaGenerator.java`
- Modify: `YamlDesiredStateProcessor` — add schema generation step
- Test: generate schema for K8sSecretSpec, validate it matches record fields

- [ ] Steps: Jandex record introspection → JSON Schema generation → build-time integration → commit

### Task 20: Cross-vendor topology integration test

**Files:**
- Create: `plugins/src/test/resources/cross-vendor-topology.yaml`
- Create: `plugins/src/test/java/.../CrossVendorTopologyIT.java`

- [ ] Steps: YamlGraph topology using K8sSecret + CloudflareDns + SupabaseDatabase → compile → reconcile with WireMock endpoints → verify ordering → commit

---

## References

- [2026-09-08-yaml-plugin-architecture-design.md] — design spec (adversarially reviewed)
- [decisions.md] — 8 design decisions
- [casehub-desiredstate] — YamlDesiredStateProcessor, NodeSpecFactory, NodeSpecRegistry, VariableResolver, CbrFaultPolicy, ThresholdFaultPolicy
- [casehub-ops/infra] — InfraNodeSpec, InfraWrappingFactory, InfraNodeSpecFactoryProvider
- [casehub-ras-api] — SituationDefinition, ChainMode, TriggerAction
- [docs/research/2026-09-08-self-healing-self-governing-infrastructure.md] — gap analysis
- GitHub #87 — this work
- GitHub #86 — CrossplaneNodeProvisioner (future)
