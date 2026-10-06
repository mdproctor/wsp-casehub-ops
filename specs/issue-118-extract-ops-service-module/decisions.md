## D1: Extraction boundary — everything moves to ops/service

**Choice:** All Java source, entities, services, K8s implementations, SPI stubs, case descriptors, and Flyway migrations move to ops/service. ops/app becomes a thin Quarkus shell with only application.properties and the quarkus-maven-plugin configuration.
**Alternatives:**
- Partial extraction (APIs + entities only) — leaves service layer split across modules, complicates CDI wiring
- K8s in separate ops/k8s module — additional module boundary for 12 files, YAGNI
- Case descriptors stay in app — prevents Scaffold from reusing them
**Rationale:** Clean cut. ops/service is a complete, self-contained library. ops/app adds nothing except the Quarkus application packaging. Any consumer (ops/app, Scaffold) gets the full ops capability by depending on ops/service.
**Trade-offs:** ops/app becomes nearly empty — but that's the point. A thin shell is easy to maintain and makes the library/app boundary obvious.
**Sources:** ops/app/pom.xml, OpsApplicationApi.java, StubActualStateAdapter.java, casehub-ops#118
**Exploration:** quick
**Status:** captured

## D2: Flyway migrations follow entities

**Choice:** Migrations (V1-V6) move to ops/service alongside the JPA entities they define schemas for. Single source of truth.
**Alternatives:**
- Shared ops/migrations module — overkill for 6 files
- Stay in ops/app — splits schema definition from entity classes
**Rationale:** Quarkus Flyway classpath scanning picks up migrations from any dependency JAR. Keeping migrations with entities means any consumer gets a working schema automatically.
**Trade-offs:** None — standard Quarkus pattern.
**Sources:** ops/app/src/main/resources/db/app/migration/V1-V6
**Exploration:** quick
**Status:** captured

## D3: Module naming — ops/service

**Choice:** New module named `service/` producing `casehub-ops-service` artifact, root package `io.casehub.ops.service`.
**Alternatives:**
- ops/runtime — Quarkus convention for extension runtimes, but this isn't a Quarkus extension
- ops/core — conflicts with the conceptual role of ops/api
**Rationale:** `service` communicates that this is the service layer between the API types (ops/api) and the application shell (ops/app). Matches the pattern described in #118.
**Trade-offs:** None.
**Sources:** casehub-ops#118 issue description
**Exploration:** quick
**Status:** captured
