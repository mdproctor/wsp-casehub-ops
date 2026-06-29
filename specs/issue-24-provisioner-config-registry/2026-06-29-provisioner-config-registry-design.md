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
| `store(agentId, List<ProviderConfig>)` | Stores list directly | Converts List→Map keyed by providerName. Last-write-wins on duplicate providerName (log warning). |
| `forAgent(agentId)` | Returns `List<ProviderConfig>` | Returns `Map<String, ProviderConfig>` (unmodifiable). `Map.of()` for unknown agents. |
| `agentIds()` | Unchanged | Unchanged |
| `remove(agentId)` | Unchanged | Unchanged |

The `store()` input signature stays as `List<ProviderConfig>` — callers pass Lists from
`AgentNodeSpec.providerConfigs()`. Conversion happens at the store boundary.

## Registry Implementation

New class: `DeploymentProvisionerConfigRegistry` in `io.casehub.ops.deployment`.

**Annotations:** `@Alternative @Priority(1) @ApplicationScoped`

**Implements:** `ProvisionerConfigRegistry` (from `io.casehub.api.spi`)

**Dependency:** injects `DeploymentProviderConfigStore`

### Methods

`configFor(String providerName, String agentId) → Map<String, Object>`:
Look up `store.forAgent(agentId).get(providerName)`. Return the config map if found, `Map.of()` if not.

`declaredAgentIds(String providerName) → Set<String>`:
Iterate `store.agentIds()`, filter to those whose `forAgent()` map contains the providerName key,
collect to unmodifiable Set.

### CDI Displacement

`@Alternative @Priority(1)` auto-activates in Quarkus without `selected-alternatives` config
(per GE-20260429-a79d0e). When ops-deployment is on the classpath, this displaces
`NoOpProvisionerConfigRegistry @DefaultBean @ApplicationScoped` in engine-runtime.

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

### Existing Test Updates

9 call sites across `AgentProvisionHandlerTest` and `DeploymentLifecycleIntegrationTest` need
mechanical updates: `List` assertions → `Map` assertions (`.get(0)` → `.get("providerName")`,
`.hasSize(1)` → `.containsKey("providerName")`, etc.).

No CDI integration test for `@Alternative` displacement — that is Quarkus framework behavior.

## Scope Exclusions

- No POM changes (engine-api already a dependency in deployment/pom.xml)
- No reverse index on the store (agent count is small, O(n) iteration is fine)
- No convenience methods beyond existing store API
- `AgentNodeSpec.providerConfigs` stays as `List<ProviderConfig>` — the store boundary handles conversion
