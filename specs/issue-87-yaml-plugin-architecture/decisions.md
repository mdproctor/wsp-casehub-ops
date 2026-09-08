# Decisions — YAML-First Plugin Architecture

## D1: Two-repo extension split

**Choice:** Domain-agnostic primitives in casehub-desiredstate/extensions, domain-specific plugins in casehub-ops/extensions
**Alternatives:**
- New repo (casehub-yaml-plugins) — clean separation but adds a repo and splits the compilation pipeline
- All in ops — ops is integration-tier; generic primitives don't belong there
**Rationale:** Preserves the existing boundary: desiredstate owns YAML frontend + graph compilation (domain-agnostic), ops owns operational domain implementations (domain-specific)
**Trade-offs:** Cross-repo work for the initial build; primitives and plugins can't be developed in isolation
**Sources:** casehub-ops/CLAUDE.md (module structure), casehub-desiredstate architecture
**Exploration:** quick
**Status:** captured

## D2: Dual protocol — REST + GraphQL

**Choice:** Support both REST and GraphQL as first-class API surface protocols in the primitive layer
**Alternatives:**
- REST only — simpler, covers K8s and most vendor APIs
- GraphQL only — more type-safe but K8s API is REST
**Rationale:** CaseHub itself has a dual REST/GraphQL platform. GraphQL is more type-safe (schema introspection enables build-time validation and IDE refactoring). REST is needed for K8s and most vendor APIs. Both are needed.
**Trade-offs:** Two client implementations, two extraction models (JSONPath for REST, query-native for GraphQL)
**Sources:** CaseHub platform API design, user input on type safety and refactorability
**Exploration:** quick
**Status:** captured

## D3: Mixed plugin set — K8s + named free-tier vendors

**Choice:** 4 K8s core plugins + 6 named vendor plugins (Cloudflare, Supabase, Fly.io, Oracle Cloud, Let's Encrypt)
**Alternatives:**
- K8s only (10 K8s resource types) — testable entirely on Kind/Podman but doesn't prove cross-vendor
- K8s + OpenShift — Red Hat aligned but narrower audience
- Abstract vendors (DnsRecordSpec without naming a provider) — a spec with no provisioner
**Rationale:** Proves the multi-vendor free-tier aggregation story from day one. Named vendors force real API integration, not abstract interfaces. K8s handles compute, vendors handle supporting infrastructure.
**Trade-offs:** Vendor API changes can break plugins; need WireMock for offline testing
**Sources:** Free-tier research, multi-cloud discussion
**Exploration:** quick
**Status:** captured

## D4: CBR inline in plugin YAML (required section)

**Choice:** Every plugin YAML must declare a `cbr:` section defining case-features, resolution-strategies, and outcome-signals
**Alternatives:**
- Separate CBR config (per deployment) — more flexible but CBR becomes an afterthought
- Optional cbr: section — pragmatic but plugins ship without learning surfaces
**Rationale:** Forces plugin authors to think about "what does this learn from?" before the plugin is valid. Schema validator rejects plugins without CBR. Cements CaseHub's differentiator: every managed resource learns.
**Trade-offs:** Higher barrier to writing a plugin; some simple resources may have trivial CBR surfaces
**Sources:** CbrFaultPolicy in desiredstate, user input on CBR as differentiator
**Exploration:** quick
**Status:** captured

## D5: RAS inline in plugin YAML (required section)

**Choice:** Every plugin YAML must declare a `ras:` section defining detection situations (pre-emptive + reactive)
**Alternatives:**
- RAS at topology level only — detection configured per deployment, not per plugin
- Defaults in plugin, overridable in topology — best of both but more complex schema
**Rationale:** Completes the detect→diagnose→resolve→learn loop in one file. Plugin = complete self-healing unit. Pre-emptive detection (identified as a gap in the self-healing research) is forced by schema.
**Trade-offs:** Plugin author must understand RAS situation model; some situations may be generic across resource types
**Sources:** RAS API (SituationDefinition, ChainMode, TriggerAction), self-healing research gap analysis
**Exploration:** quick
**Status:** captured
**Depends on:** D4 (CBR inline — together they form the full self-healing loop)

## D6: Podman-based testing

**Choice:** Kind-on-Podman for K8s, WireMock containers for vendor APIs, real free-tier accounts for integration tests
**Alternatives:**
- Minikube — heavier, less Podman-friendly
- Testcontainers only — doesn't give a real K8s API for K8s plugins
**Rationale:** Podman is available, Kind works on Podman (KIND_EXPERIMENTAL_PROVIDER=podman), WireMock gives offline vendor API testing. Real accounts for CI integration tests.
**Trade-offs:** Kind on Podman requires RYUK disabled for Testcontainers; WireMock stubs must track vendor API changes
**Sources:** Podman availability, Kind documentation
**Exploration:** quick
**Status:** captured

## D7: Java/YAML boundary

**Choice:** Java for transport (HTTP/GraphQL/WebSocket clients, auth, JSONPath evaluation, rate limiting, schema generation). YAML for configuration (rest-call, graphql-call, json-extract, compare-state, paginate, poll-until). YAML primitives compose over other YAML primitives.
**Alternatives:**
- More in Java (each primitive is a Java class configured by a single YAML property) — less composable
- More in YAML (template-based HTTP client in YAML) — YAML can't manage sockets/TLS
**Rationale:** The boundary is "can YAML express this?" Transport, auth flows, expression evaluation = no. Configuration of what to call, how to map, when to retry = yes. YAML over YAML over Java gives three composable layers.
**Trade-offs:** The YAML primitive layer needs its own schema and validation; more moving parts than pure Java plugins
**Sources:** InfraBackend pattern in ops, YamlFaultPolicy in desiredstate (proof YAML SPI impl works)
**Exploration:** quick
**Status:** captured

## D8: Five required plugin sections

**Choice:** Every plugin YAML has five required sections: actual-state, provisioner, fault-policy, cbr, ras
**Alternatives:**
- Three required (actual-state, provisioner, fault-policy) + two optional (cbr, ras) — lower barrier
- Two required (actual-state, provisioner) + three optional — minimal plugin
**Rationale:** The five sections form the complete self-healing unit. Requiring all five means every plugin is a "why CaseHub" demonstration. The product story is: "every resource you manage detects its own problems, resolves them with strategies that improve over time, and escalates to humans when confidence is low."
**Trade-offs:** Higher barrier to entry; plugin authors must understand CBR + RAS. Mitigated by good defaults and plugin templates.
**Sources:** D4 (CBR inline), D5 (RAS inline)
**Exploration:** quick
**Depends on:** D4, D5
**Status:** captured
