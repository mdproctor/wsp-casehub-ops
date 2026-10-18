## D1: NodeSpec type strategy

**Choice:** Sealed interface — `ClaudonyNodeSpec` sealed interface permitting `ClaudonyAppNodeSpec` and `ClaudonyPoolNodeSpec`
**Alternatives:**
- Single spec with discriminator — apps and pools have different fields, leading to nullable fields everywhere
- Sealed with wrapper — claudony has one backend, wrapper adds nothing
**Rationale:** Apps and pools have genuinely different fields and provisioning semantics (app has image/config/namespace; pool has agentId/minActive/maxActive). This matches the IoT precedent in ARC42STORIES §4: "variants have different fields and provisioning semantics → sealed hierarchy."
**Trade-offs:** Two record types instead of one, but enables clean pattern matching in the provisioner and enforced dependency edges in the compiler.
**Sources:** ARC42STORIES.MD §4 Type Strategy Selection, IoTNodeSpec sealed hierarchy, ContainerNodeSpec sealed interface
**Exploration:** quick
**Status:** captured

## D2: Drift detection approach

**Choice:** Direct in ActualStateAdapter — query claudony's `listApps(namespace)` in one batch call, iterate desired nodes, map HealthStatus → NodeStatus. SpecHashStore for declaration-change detection.
**Alternatives:**
- NodeDriftChecker per type — deployment module pattern, adds CDI indirection and 3+ extra classes for only 2 node types
- Hybrid — helper methods without CDI indirection, middle ground
**Rationale:** Two node types, one API source, one batch query. The deployment module's complexity (8 types, different foundation APIs) isn't warranted.
**Trade-offs:** If the claudony domain grows beyond 2-3 node types, this may need refactoring to the drift checker pattern.
**Sources:** PodmanActualStateAdapter (container module), DeploymentActualStateAdapter (deployment module)
**Exploration:** quick
**Status:** captured

## D3: Provisioner structure

**Choice:** Inline switch — `ClaudonyNodeProvisioner` with `switch (node.spec())` dispatching to private methods
**Alternatives:**
- Handler classes per type — separate `@ApplicationScoped` handler beans, over-engineered for 2 types sharing the same client
**Rationale:** Two node types that both use `ClaudonyFleetClient`. No divergent dependency injection needed. Same reasoning as D2.
**Trade-offs:** Same as D2 — refactor if the domain grows significantly.
**Sources:** PodmanNodeProvisioner (container module), DeploymentNodeProvisioner (deployment module)
**Exploration:** quick
**Depends on:** D1 (sealed interface enables the switch pattern)
**Status:** captured

## D4: Pool provisioning via current API

**Choice:** Read-modify-write on the parent app — to provision a pool, read app spec, append/update pool in pools list, call `updateApp()`. To deprovision, read app, remove pool, update.
**Alternatives:**
- Block on claudony standalone pool CRUD endpoints — cleaner but blocks this issue
- Read-modify-write now, file issues for later — same as chosen, with explicit future work tracking
**Rationale:** Works today with no claudony changes. Desiredstate reconciliation is sequential per tenant, so no concurrent mutation races. When standalone pool endpoints land, the provisioner is a one-method change.
**Trade-offs:** Provisioning a pool requires reading and writing the entire app spec. Acceptable for the expected scale.
**Sources:** ClaudonyFleetClient.updateApp(), ClaudonyAppSpec.pools
**Exploration:** quick
**Depends on:** D1 (pools as separate graph nodes)
**Status:** captured

## D5: FaultPolicy escalation target

**Choice:** `ClaudonyReviewSpec` in ops-api (`io.casehub.ops.api.claudony`) — carries failed node ID, failure count, last error. `ClaudonyFaultPolicy` uses `ThresholdFaultPolicy` configured for `PROVISION_FAILED` on `claudony_app` and `claudony_pool`, tier 3 threshold.
**Alternatives:**
- Reuse generic review mechanism — no generic exists; each domain defines its own review spec
**Rationale:** Follows established convention. Every domain (container, infra, deployment) has its own review spec.
**Trade-offs:** Another domain-specific review spec, but this is the pattern the runtime expects.
**Sources:** ContainerFaultPolicy, ContainerReviewSpec, ThresholdFaultPolicy
**Exploration:** quick
**Status:** captured
