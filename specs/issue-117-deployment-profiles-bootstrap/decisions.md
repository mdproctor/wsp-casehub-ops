## D1: Topology YAML layer representation

**Choice:** Extend the existing desiredstate module system
**Alternatives:**
- New TopologyManifest record — introduces parallel loading mechanism, duplicates what modules already do
- Convention-based directories — implicit ordering, no dependency declaration between layers
**Rationale:** The desiredstate module system already supports directory-based merging, classpath+filesystem resolution, and cross-layer dependency ordering via provides/requires on NodeType. Reusing it avoids a second composition mechanism and leverages CrossDomainCompositionEngine for layer ordering.
**Trade-offs:** Module system was designed for infrastructure topology modules (ha-multi-az, load-balancer). Using it for platform service topology stretches the original intent, but the mechanics are identical.
**Sources:** infra/src/main/resources/META-INF/desiredstate/modules/, ARC42STORIES.MD §4 CrossDomainCompositionEngine
**Exploration:** quick
**Status:** captured

## D2: "Use existing or deploy if missing" mechanism

**Choice:** ActualStateAdapter probes for existing services
**Alternatives:**
- Two-pass compilation — GoalCompiler removes satisfied nodes; breaks SPI separation, loses drift detection on existing services
- Service registry — new component for endpoint discovery; adds abstraction layer without matching benefit
**Rationale:** Follows the existing desiredstate contract exactly. ActualStateAdapter.status() returns PRESENT/ABSENT, the runtime handles provisioning decisions. Per-node YAML override (existing: <url> or backend: <type>) takes precedence over probing. Drift detection on existing services comes for free — if a pre-deployed qhorus goes down, the adapter reports ABSENT and the provisioner can respond.
**Trade-offs:** Probing adds latency to each reconciliation cycle. Services that are slow to respond may cause false ABSENT reports. Need timeout/retry configuration per service type.
**Sources:** desiredstate-api ActualStateAdapter contract, ARC42STORIES.MD §6 Reconciliation Cycle
**Exploration:** quick
**Depends on:** D1 (topology representation determines what nodes the adapter probes)
**Status:** captured

## D3: Profile-to-backend mapping architecture

**Choice:** Config-driven CDI selection via Quarkus profiles
**Alternatives:**
- Backend SPI per service type — type-safe but creates 9+ interface/implementation combinations (3 services × 3 backends)
- Unified strategy pattern — fewer interfaces but each strategy becomes complex, handling all service types
**Rationale:** Follows the established InfraBackend pattern. Quarkus profiles naturally select CDI alternatives. Adding a new backend = new CDI bean + config profile entry. Node-level YAML override (backend: container) takes precedence over profile default. Config property casehub.ops.profile.default-backend selects the default.
**Trade-offs:** Less type-safety than per-service SPIs. A single backend implementation must handle multiple service types, or fall back to service-specific handlers internally.
**Sources:** infra InfraBackend SPI pattern, Quarkus arc.selected-alternatives mechanism
**Exploration:** quick
**Depends on:** D1 (layers determine which nodes need backend resolution)
**Status:** captured

## D4: Self-maintenance bootstrap model

**Choice:** Phased reconciliation with self-registration
**Alternatives:**
- Separate bootstrap controller — introduces a second persistent process that itself needs lifecycle management
- Blue-green with external gate — requires infrastructure (load balancer, DNS) not available in all profiles
**Rationale:** Matches the kubelet model: seed starts, provisions dependencies, then registers itself as a managed node. SelfMaintenanceProvisioner handles upgrades: provision replacement → health-check → swap → shutdown old. The running instance never kills itself — it orchestrates its replacement. In local-dev this is a restart; in enterprise it's a rolling deployment.
**Trade-offs:** The self-registration moment is a state transition that must be idempotent (the ops app may restart during phase 1). The "swap" mechanism differs per profile — in-process restart vs container replacement vs K8s rolling update.
**Sources:** Kubernetes kubelet self-hosting model, CaseHubApplicationSpec + ApplicationDeploymentTarget SPI
**Exploration:** quick
**Depends on:** D2 (adapter probing determines when dependencies are healthy), D3 (backend selection determines how the replacement is provisioned)
**Status:** captured

## D5: Topology YAML location

**Choice:** Classpath default + filesystem override
**Alternatives:**
- Config property per layer — more granular but verbose, harder to manage as an installation artifact
- Git-backed topology — auditable but adds cloning complexity; better as a future enhancement on top of filesystem override
**Rationale:** Follows the established goal loader pattern (DeploymentGoalLoader, ComplianceGoalLoader). Classpath default means java -jar works with zero config for local-dev. Filesystem override (casehub.ops.topology.path) for production installations. Directory-based merging already handled by the module system.
**Trade-offs:** No built-in versioning of topology changes — filesystem overrides are mutable. Git-backed topology would address this but is out of scope.
**Sources:** DeploymentGoalLoader classpath-first pattern, ComplianceGoalLoader
**Exploration:** quick
**Depends on:** D1 (module system determines how layers are loaded from the topology location)
**Status:** captured

## D6: Platform service node type location

**Choice:** New PlatformServiceSpec sealed hierarchy in api/ module, provisioner/adapter in service/
**Alternatives:**
- Extend container module — ties platform services to container backend; in-process and K8s backends would need separate types elsewhere
- Use existing infra domain — mixes CaseHub-specific platform services with generic infrastructure types; platform services have richer semantics (health, API compatibility, versioning)
**Rationale:** Platform services are distinct from both generic infrastructure and domain modules. Following the established pattern: node spec types in api/ (like DeploymentNodeSpec, InfraNodeSpec, ComplianceControlSpec), provisioner/adapter implementations in the consuming module (service/). PlatformServiceProvisioner delegates to profile-selected backend CDI beans.
**Trade-offs:** Adds a new sealed hierarchy to api/ and a provisioner to service/. api/ grows but this follows the existing cross-module type sharing pattern.
**Sources:** api/ existing sealed hierarchies (DeploymentNodeSpec, InfraNodeSpec, ContainerNodeSpec), service/ module structure
**Exploration:** quick
**Depends on:** D3 (backend CDI selection determines how PlatformServiceProvisioner dispatches)
**Status:** captured
