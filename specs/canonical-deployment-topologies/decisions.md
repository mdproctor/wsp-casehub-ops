# Canonical Deployment Topologies — Decisions

## D1: Scope

**Choice:** Matrix of application architectures × infrastructure topologies
**Alternatives:**
- Application architectures only — proves YAML expressiveness but not infra scaling
- Infrastructure topologies only — proves scaling but not application diversity
**Rationale:** Real value is proving intersections. A "multi-tier app on HA multi-AZ" is fundamentally different from either dimension alone.
**Trade-offs:** More combinations to test (~14 intersections vs ~5 or ~4)
**Sources:** Web research on deployment patterns, IBM topology docs, AWS/Azure multi-region reference architectures
**Exploration:** quick
**Status:** captured

## D2: Application Architectures

**Choice:** All 5: single service, multi-tier, microservices, event-driven, sidecar/service-mesh
**Alternatives:**
- Start with 3 core (single, multi-tier, microservices) — faster but misses event-driven and mesh patterns
- Start with 2 minimal (single, multi-tier) — too narrow to be a credible catalogue
**Rationale:** These 5 cover the major patterns people actually deploy. Each exercises different YAML primitives (linear deps, mesh deps, broker deps, sidecar injection rules).
**Trade-offs:** More YAML exemplars and tests to write
**Sources:** ArchitectureDiagram.ai patterns survey, IBM app architecture types, UpCloud 2026 patterns guide
**Exploration:** quick
**Status:** captured

## D3: Infrastructure Topologies

**Choice:** All 4: single node, load-balanced cluster, HA multi-AZ, multi-region active-passive
**Alternatives:**
- Start with 3 (defer multi-region) — simpler but misses DR patterns
- Start with 2 (single node + LB cluster) — minimum viable but doesn't prove HA or DR
**Rationale:** Covers dev through enterprise DR. Active-active deferred — it requires data replication semantics (CRDTs, consensus) beyond infrastructure topology.
**Trade-offs:** Multi-region requires DnsFailoverSpec and DataReplicationSpec Java types
**Sources:** Kubernetes HA topology docs, Azure AKS multi-region reference architecture, Google Cloud deployment archetypes
**Exploration:** quick
**Status:** captured

## D4: Verification Layers

**Choice:** 3-layer pyramid with Maven profiles: compilation (default), reconciliation (`-Preconciliation`), live deployment (`-Pinfra-live`)
**Alternatives:**
- Compilation tests only — fast but doesn't prove the system actually works
- All tests always — too slow for CI, blocks development
**Rationale:** Each layer proves something the others can't. Compilation proves YAML structure. Reconciliation proves planning correctness. Live proves real K8s integration. Maven profiles keep the default build fast.
**Trade-offs:** More test infrastructure to maintain
**Sources:** ARC42STORIES existing test patterns, Maven profile conventions in casehub repos
**Exploration:** quick
**Status:** captured

## D5: Toy Services

**Choice:** Real Docker images (nginx, postgres, redis, RabbitMQ, envoy)
**Alternatives:**
- Synthetic stubs (placeholder names) — faster to write but can't verify real deployment
- Mix (real for core, synthetic for exotic) — pragmatic but inconsistent
**Rationale:** When the live profile runs, these actually deploy, respond to health checks, and communicate. Maximum confidence. Each image is lightweight, well-documented, and industry standard.
**Trade-offs:** Requires Docker/K8s available for live tests
**Sources:** Industry standard images, Kubernetes tutorials
**Exploration:** quick
**Status:** captured

## D6: Module Location

**Choice:** New `topology-tests/` module (test-scope only)
**Alternatives:**
- Inside `app/` tests — keeps things together but bloats app's test concerns
- Inside `testing/` module — but testing/ is for shared stubs, not integration scenarios
**Rationale:** Clean separation. The topology tests depend on infra + app + desiredstate-yaml, which is a unique dependency set. Doesn't bloat existing modules.
**Trade-offs:** One more module to maintain in the Maven build
**Sources:** Existing `testing/` module pattern in casehub-ops
**Exploration:** quick
**Status:** captured

## D7: YAML Format

**Choice:** Topology as a metadata field + YAML-native composition (revised per R1-02)
**Alternatives:**
- Flat service list per cluster — simpler but YAML doesn't communicate intent
- New compiler with topology-specific schema — was the initial proposal, superseded by D8
**Rationale:** The topology type is a **metadata field** (`topology: multi-tier/ha-multi-az`) that provides observability — dashboards, audit trails, GOAP migration triggers can query it. But the compilation uses existing YAML primitives (modules, invariants, rules, forEach). Topology identity is declared, not inferred. Implementation is generic.
**Trade-offs:** Users must pick a topology type. The metadata field doesn't enforce — invariants and modules do the actual work.
**Sources:** casehub-desiredstate-yaml tutorials, InfraGoalCompiler patterns, decision review R1-02
**Exploration:** quick → revised after review
**Status:** revised

## D8: Implementation Approach

**Choice:** YAML-native composition using existing desiredstate-yaml primitives (modules, invariants, rules, forEach, lifecycle phases)
**Alternatives:**
- New TopologyGoalCompiler Java class — was the initial proposal but the YAML frontend already has every primitive needed
- Archetype catalogue with parameter substitution — rigid, doesn't prove YAML expressiveness
**Rationale:** Auditing `casehub-desiredstate-yaml` revealed modules, forEach, invariants, rules, lifecycle phases, and NodeSpecRegistry. A topology IS a composition of these primitives. No new compiler needed.
**Trade-offs:** Depends on the YAML frontend being feature-complete for all topology patterns. Any gaps in the YAML frontend become gaps in topology support.
**Sources:** YamlGraphRecorder.java, ForEachExpander.java, ModuleExpander.java, GraphInvariantEngine, webapp-yaml tutorials
**Exploration:** deep-analysis
**Depends on:** D7 (topology as first-class concept)
**Status:** captured

## D9: Gap Analysis — Extend, Don't Reinvent (revised per R1-03)

**Choice:** InfraNodeSpec extensions (Java) + topology modules (YAML) + GOAP bridge code (Java) are new.
**Alternatives:**
- Build parallel infrastructure — new compiler, new planner, new test framework
**Rationale:** CaseHub already has TransitionPlanner (steady-state), GoapPlanner (migration), Engine Cases (orchestration), Service Lifecycle (monitoring), and a rich YAML frontend. Building parallel machinery wastes effort and fragments the architecture.
**Trade-offs:** Tighter coupling to existing components. GOAP bridge code is larger than initially estimated — requires world state builder from ActualStateAdapter, action-to-provisioner bridge, and migration mode coordinator (see D10).
**Sources:** TransitionPlanner.java, GoapPlanner.java, GoapPlanningStrategy.java, ServiceCaseContext, DimensionStatusService, decision review R1-03
**Exploration:** deep-analysis → revised after review
**Depends on:** D8 (YAML-native approach)
**Status:** revised

## D10: CaseHub Full-Stack Integration (revised per R1-04, R1-05)

**Choice:** Full stack, phased. Single-cluster topologies (Phases 1–5) first. GOAP migration with coordination mechanism (Phase 6). Multi-cluster coordination for multi-region (Phase 7+).
**Alternatives:**
- Standalone topology management — simpler but misses the integration story
- Full multi-cluster from the start — too much new architecture before proving single-cluster
**Rationale:** Each CaseHub component handles a different timescale. But GOAP migration requires a coordination mechanism — a migration mode flag that suspends TransitionPlanner's naive diff-and-converge while a migration case is active. Multi-region active-passive spans multiple reconciliation loops and requires cross-cluster coordination that doesn't exist yet — this is Phase 7+ work, not Phase 1–5.
**Trade-offs:** Multi-region is aspirational until cross-cluster coordination is designed. Phase 1–5 prove the topology story on single-cluster topologies.
**Sources:** TransitionPlanner.java, GoapPlanner.java, GoapPlanningStrategy.java, decision review R1-04, R1-05, R1-13
**Exploration:** deep-analysis → revised after review
**Status:** revised

## D11: Matrix Purpose Split (new, per R1-06)

**Choice:** Separate test intersections (~5) from tutorial exemplars (~9)
**Alternatives:**
- All 14 intersections get full 3-layer testing — diminishing returns past ~5
- Only 5 intersections, no tutorials — misses the YAML-first engagement value
**Rationale:** 5 intersections cover every YAML primitive at least once (forEach, modules, rules, invariants, lifecycle phases). The remaining 9 are tutorial exemplars with compilation tests only. This separates testing rigour from documentation value.
**Trade-offs:** Tutorial exemplars get less verification. Test exemplars may be less readable than tutorial-optimised versions.
**Sources:** Decision review R1-06
**Exploration:** quick (surfaced by review)
**Status:** captured

### Core Test Intersections (5)

1. **Multi-tier on single-node** — lifecycle phases, linear deps, baseline
2. **Microservices on HA multi-AZ** — forEach across AZs, service discovery, modules
3. **Event-driven on LB cluster** — broker-centric topology, invariants
4. **Sidecar/mesh on LB cluster** — rules for sidecar injection, mesh module
5. **Single service on single-node** — minimum viable, validation baseline
