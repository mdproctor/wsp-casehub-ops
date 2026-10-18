# Handover — Topology Compiler Service landed

**Branch:** `main` (clean — `issue-179-topology-compiler-service` landed as `0c437be`)
**Closed this session:** #179 (topology compiler service facade), #180 (PodmanDeploymentTarget CDI fix), #164 (epic: rewrite YAML/README to methodology)
**Next work:** #140 (claudony NodeProvisioner — M/Med)

## What was done this session

### Implementation — #179, #180 (landed as single squashed commit)

**TopologyCompilerService** (`service/src/main/java/.../topology/`) — composition-aware facade for the scaffold console:
- `deploy(tenancyId, basePath, topologyName)` — compile YAML → compose graph → start ReconciliationLoop
- `listTopologiesFromPath(path)` — scan filesystem for topology directories
- `compileDomain(yamlPath, typeRegistry)` — parse YAML via YamlGraphRecorder
- `stop(tenancyId, topologyName)` / `isRunning(tenancyId, topologyName)` — lifecycle management
- Loop key format: `topology:<tenancyId>:<topologyName>` (mutually exclusive with K8s path)

**NodeTypeRegistryProducer** — discovers node types via sealed interface `@NodeTypeId` scanning + `NodeSpecFactoryProvider` collection (CDI or explicit varargs)

**PodmanDeploymentTarget** — added `@ApplicationScoped` + `@Inject`. No `@LookupIfProperty` — all deployment targets must be CDI beans simultaneously for per-node routing via `targetId()`.

**Topology YAML packaging** — `01-hello-llm` shipped in `META-INF/desiredstate/topologies/`. Dev default: `%dev.casehub.ops.deployment.target=in-process`.

14 unit tests, 585-test full build green.

### Design decisions (spec at `specs/issue-179-topology-compiler-service/`)

- D1: Composition-aware — each YAML is a domain contribution, not standalone
- D2: Two-level API — high-level `deploy()` + low-level steps
- D3: Dual type discovery — sealed scanning + NodeSpecFactoryProvider (both paths)
- D4: META-INF/desiredstate/topologies/ with filesystem override
- D5: Explicit API only (no auto-start on boot)
- D6: @ApplicationScoped only, no @LookupIfProperty

### Epic closure

**#164 (Rewrite YAML/README to methodology)** — all children (#174, #175) complete. Closed.

### Issue filed

**Hortora/soredium#462** — `ctx.py` plan detection returns `HAS_PLAN=no` when `.plan` file exists. Reproducer: create `<branch-name>.plan` in workspace, run `plan_manager.py detect` — reports no plan. Workaround: pass `plan_path=` explicitly.

## Known limitations

- Factory-based type wrapping (InfraWrappingFactory, DeploymentNodeSpecFactoryProvider) not yet integrated into the `deploy()` compilation path. Sealed scanning finds the types but deserialization goes through Jackson rather than the domain-specific factories. Works for deployment topologies; infra nodes needing `backendId` wrapping will need `NodeSpecRegistry` integration.
- `CrossDomainCompositionEngine` integration is architectural but not yet exercised in `deploy()` — current implementation merges graphs directly. The engine wire-up is designed but deferred until multi-domain topologies are needed.

## Next session

**#140 — claudony NodeProvisioner** (M/Med, unblocked):
- `ClaudonyGoalCompiler` — compile YAML fleet declarations into DesiredStateGraph
- `ClaudonyActualStateAdapter` — read live state from claudony registry API
- `ClaudonyNodeProvisioner` — map desiredstate provision/deprovision/suspend/resume to claudony REST calls
- Reference implementations: `CaseHubAppProvisionHandler`, `DeploymentActualStateAdapter` in ops/deployment
- Cross-repo dep: claudony#282 (app lifecycle API — landed), platform#615 (HealthStatus.SUSPENDED — landed)
