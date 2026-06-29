# ProvisionerConfigRegistry from DeploymentProviderConfigStore

**Issue:** casehubio/casehub-ops#24
**Date:** 2026-06-29
**Module:** deployment

## Problem

`DeploymentProviderConfigStore` holds per-agent provider config populated during provisioning.
Provisioners (Claudony, OpenClaw) need this config when ops is co-deployed, but the store is
internal to the deployment module — not accessible via the engine SPI.

The engine defines `ProvisionerConfigRegistry` (in `io.casehub.api.spi`) with a no-op
`@DefaultBean` implementation. When ops-deployment is on the classpath, the no-op should be
displaced by a real implementation backed by the store.

**Consumer status:** `ProvisionerConfigRegistry.configFor()` and `declaredAgentIds()` currently
have zero call sites in the engine or consumer codebases. This spec implements the producer side
(ops). Consumer migration is tracked via engine#584 which remains open until at least one consumer
(Claudony, OpenClaw) migrates. claudony#164 was closed for the earlier `ProviderConfigSource` SPI,
not `ProvisionerConfigRegistry` — claudony#165 now tracks migration to the engine SPI.
ops#24 remains open until this implementation is merged.

## Root Fix: Store Data Model

The store currently uses `ConcurrentHashMap<String, List<ProviderConfig>>` — keyed by agentId,
values are Lists. But the domain invariant is one config per provider per agent. A List admits
duplicate providerNames, which has no valid interpretation.

**Fix:** change the internal representation to `ConcurrentHashMap<String, Map<String, ProviderConfig>>`
where the inner map is keyed by `providerName`. This makes the invariant structural.

### Store Changes

| Method | Before | After |
|--------|--------|-------|
| Internal field | `ConcurrentHashMap<String, List<ProviderConfig>>` | `ConcurrentHashMap<String, Map<String, ProviderConfig>>` |
| `store(agentId, List<ProviderConfig>)` | Stores list directly | Converts List→Map keyed by providerName. Last-write-wins on duplicate providerName (log warning via `private static final Logger LOG = Logger.getLogger(DeploymentProviderConfigStore.class)` — matches module's established static logger pattern). |
| `forAgent(agentId)` | Returns `List<ProviderConfig>` | Returns `Map<String, ProviderConfig>` (unmodifiable). `Map.of()` for unknown agents. |
| `agentIds()` | Unchanged | Unchanged |
| `remove(agentId)` | Unchanged | Unchanged |

The `store()` input signature stays as `List<ProviderConfig>` — callers pass Lists from
`AgentNodeSpec.providerConfigs()`. Conversion happens at the store boundary.

## Registry Implementation

New class: `DeploymentProvisionerConfigRegistry` in `io.casehub.ops.deployment`.

**Annotations:** `@ApplicationScoped`

**Implements:** `ProvisionerConfigRegistry` (from `io.casehub.api.spi`)

**Dependency:** injects `DeploymentProviderConfigStore`

### Methods

`configFor(String providerName, String agentId) → Map<String, Object>`:
Look up `store.forAgent(agentId).get(providerName)`. Return the config map if found, `Map.of()` if not.

`declaredAgentIds(String providerName) → Set<String>`:
Iterate `store.agentIds()`, filter to those whose `forAgent()` map contains the providerName key,
collect to unmodifiable Set.

**Consistency model:** `declaredAgentIds` iterates `ConcurrentHashMap.keySet()` (weakly consistent)
then calls `forAgent()` per agent. Between iteration and lookup, agents may be added or removed
concurrently via `provision()`/`deprovision()`. The result is a point-in-time snapshot — acceptable
for a config-lookup SPI consumed during provisioner initialization, but consumers must not assume
strong consistency across the returned set.

### CDI Displacement

`NoOpProvisionerConfigRegistry` in engine-runtime is annotated `@DefaultBean @ApplicationScoped`.
Per the platform's DefaultBean SPI convention (engine spec `2026-05-14-defaultbean-spi-noops-design`),
any non-default qualifying bean for the same type automatically takes precedence. A plain
`@ApplicationScoped` implementation is sufficient to displace the `@DefaultBean` no-op — no
`@Alternative`, no `@Priority`, no configuration required.

## Testing

### Store Tests (update existing)

- Update `DeploymentProviderConfigStoreTest` assertions for `Map<String, ProviderConfig>` return type.
- Add test: duplicate providerName in input list → last-write-wins, store contains one entry per providerName.

### Registry Tests (new)

New `DeploymentProvisionerConfigRegistryTest` — unit tests with real `DeploymentProviderConfigStore`
(no mocking, store is a simple in-memory ConcurrentHashMap).

| Test case | Expectation |
|-----------|-------------|
| `configFor` — agent+provider exists | Returns config map |
| `configFor` — unknown agent | Returns `Map.of()` |
| `configFor` — known agent, unknown provider | Returns `Map.of()` |
| `declaredAgentIds` — agents with given provider | Returns matching agent IDs |
| `declaredAgentIds` — no agents with given provider | Returns `Set.of()` |

### Structural Verification Test (new)

Plain JUnit test (no CDI bootstrap) that verifies `DeploymentProvisionerConfigRegistry` via
reflection:
- Has `@ApplicationScoped` annotation
- Implements `ProvisionerConfigRegistry`
- Has a constructor injectable by CDI (single constructor accepting `DeploymentProviderConfigStore`)

This catches the most common CDI wiring failures (wrong annotation, wrong interface, unsatisfiable
constructor) without requiring Quarkus container bootstrap. The deployment module has zero
`@QuarkusTest` tests and 21 `@ApplicationScoped` beans — bootstrapping a container for a 2-bean
test would require extensive `quarkus.arc.exclude-types` configuration. Full displacement testing
(NoOp vs Deployment) is deferred to consumer repo integration suites when consumers migrate to
`ProvisionerConfigRegistry` (tracked via engine#584).

### Existing Test Updates

`forAgent()` currently has zero production callers — all call sites are in test code. The return
type change (`List<ProviderConfig>` → `Map<String, ProviderConfig>`) has zero production impact.

5 lines across `AgentProvisionHandlerTest` and `DeploymentLifecycleIntegrationTest` have compile
failures requiring changes:

| Location | Change |
|----------|--------|
| `provisionStoresProviderConfigs` — type declaration | `List<ProviderConfig>` → `Map<String, ProviderConfig>` |
| `provisionStoresProviderConfigs` — extracting assertion | `extracting(ProviderConfig::providerName)` → Map key assertions |
| `provisionStoresProviderConfigs` — `.get(0)` | `stored.get(0)` → `stored.get("claudony")` |
| `provisionStoresProviderConfigs` — `.get(1)` | `stored.get(1)` → `stored.get("openclaw")` |
| `fullLifecycle` — `.get(0)` | `.get(0)` → `.get("claudony")` |

3 additional lines use `hasSize()` which works on both `List` and `Map` via AssertJ but should
be updated for semantic clarity (e.g. `.containsKey("claudony")`). 1 line (`isEmpty()`) works
identically on both types and needs no change.

## Scope Exclusions

- No POM changes (engine-api already a dependency in deployment/pom.xml)
- No reverse index on the store — agent count is small, O(n) iteration is acceptable
  (tracked: casehubio/casehub-ops#27)
- No convenience methods beyond existing store API
- `AgentNodeSpec.providerConfigs` stays as `List<ProviderConfig>` — the store boundary handles conversion
