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

**Choice:** Topology-aware as a first-class construct
**Alternatives:**
- Flat service list per cluster — simpler but YAML doesn't communicate intent
- Topology as metadata — topology is an annotation, not a structural concept
**Rationale:** The topology type informs validation (invariants), auto-wiring (rules), and node generation (modules). Making it first-class means the YAML communicates what kind of system this is, not just what it contains.
**Trade-offs:** More opinionated — users must pick a topology type
**Sources:** casehub-desiredstate-yaml tutorials, InfraGoalCompiler patterns
**Exploration:** quick
**Status:** captured

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

## D9: Gap Analysis — Extend, Don't Reinvent

**Choice:** Only InfraNodeSpec extensions (Java) + topology modules (YAML) + GOAP actions (Java) are new. Everything else exists.
**Alternatives:**
- Build parallel infrastructure — new compiler, new planner, new test framework
**Rationale:** CaseHub already has TransitionPlanner (steady-state), GoapPlanner (migration), Engine Cases (orchestration), Service Lifecycle (monitoring), and a rich YAML frontend. Building parallel machinery wastes effort and fragments the architecture.
**Trade-offs:** Tighter coupling to existing components. If the desiredstate YAML frontend has limitations, they constrain topology support.
**Sources:** TransitionPlanner.java, GoapPlanner.java, GoapPlanningStrategy.java, ServiceCaseContext, DimensionStatusService
**Exploration:** deep-analysis
**Depends on:** D8 (YAML-native approach)
**Status:** captured

## D10: CaseHub Full-Stack Integration

**Choice:** Topology management uses the full CaseHub stack: TransitionPlanner + GOAP + Engine Cases + Service Lifecycle
**Alternatives:**
- Standalone topology management — simpler but misses the integration story
**Rationale:** Each CaseHub component handles a different timescale: TransitionPlanner (seconds, steady-state), GOAP (minutes, migration), Engine (hours, orchestration with human gates), Service Lifecycle (days, ongoing monitoring). This is the architectural breakthrough — CaseHub's own stack IS the deployment management engine.
**Trade-offs:** Later phases (GOAP, service lifecycle integration) depend on those components being ready
**Sources:** TransitionPlanner.java analysis, GoapPlanner.java analysis, Chapter 5 service lifecycle
**Exploration:** deep-analysis
**Status:** captured
