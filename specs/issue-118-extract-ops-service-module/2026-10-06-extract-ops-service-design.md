# Extract Service Layer from ops/app into ops/service

**Issue:** casehubio/casehub-ops#118
**Date:** 2026-10-06
**Status:** Draft

## Summary

Extract all Java source, JPA entities, CDI services, Flyway migrations, and
resources from `ops/app` (a Quarkus application that can't be consumed as a
library) into `ops/service` (a plain JAR library). `ops/app` becomes a thin
Quarkus shell with only `application.properties` and the `quarkus-maven-plugin`
configuration.

This enables Scaffold (and any future aggregator) to embed ops by depending
on `casehub-ops-service` directly. It's the architectural prerequisite for
the deployment profile model (#117) — the in-process provisioner backend
requires all services in one JVM.

## New Module: ops/service

**Artifact:** `casehub-ops-service`
**Packaging:** `jar` (plain library, Jandex-indexed)
**Root package:** `io.casehub.ops.service`

### What moves (everything)

| Package | Files | Content |
|---------|-------|---------|
| `api/` | 8 | `@McpDomain` API classes (OpsApplicationApi, OpsDeploymentApi, etc.) |
| `entity/` | 5 | JPA entities (ApplicationEntity, ApprovalPlanEntity, ClusterReferenceEntity, CveEntity, DeploymentRecordEntity) |
| `service/` | 15 | CDI service beans (ApplicationLifecycleService, ClusterService, ScalingService, etc.) |
| `model/` | 12 | Domain records and enums |
| `persistence/` | 3 | JPA stores (JpaPlanStore, JpaCveStore, CveStore) |
| `k8s/` | 12 | Kubernetes SPI implementations (provisioner, handler registry, watch manager) |
| `case_/` | 9 | Case descriptors |
| `lifecycle/` | 6 | Service lifecycle + RAS situation providers |
| `rest/` | 6 | DTOs + filters |
| `spi/` | 5 | `@DefaultBean` stubs for desiredstate SPIs |
| `container/` | 1 | PodmanWatchManager |
| `goal/` | 2 | ApplicationGoalCompiler, ApplicationNodeTypes |
| `authorization/` | 1 | ConfigDrivenApprovalAuthorizer |

**Resources that move:**
- `db/app/migration/V1-V6` — Flyway migrations (follow entities)
- `META-INF/casehub-mcp-app.properties` — MCP domain registration
- `META-INF/summarisation/deployment-monitoring.yaml` — summarisation config

### Package rename

`io.casehub.ops.app.*` → `io.casehub.ops.service.*`

All internal imports update accordingly. External consumers that previously
couldn't depend on ops/app won't have existing references to rename.

### Dependencies

ops/service inherits the non-Quarkus-plugin dependencies from ops/app:

```xml
<!-- CaseHub foundation -->
casehub-ops-api
casehub-ops-container
casehub-engine
casehub-work-engine-adapter
casehub-engine-planning
casehub-engine-persistence-memory
casehub-engine-scheduler-quartz
casehub-desiredstate
casehub-desiredstate-work
casehub-ras-api
casehub-work
casehub-platform (runtime scope)
casehub-platform-expression
casehub-blocks-summarisation-yaml

<!-- Quarkus (library-compatible) -->
quarkus-rest
quarkus-rest-jackson
quarkus-hibernate-orm-panache
quarkus-flyway
quarkus-jdbc-h2
quarkus-jdbc-postgresql
quarkus-hibernate-validator

<!-- Kubernetes -->
io.fabric8:kubernetes-client
```

### Annotation processor

The `casehub-platform-graphql-generator` annotation processor and its
`-AdomainFilter` / `-AgenerateGraphQL` compiler args move to ops/service's
`pom.xml`. The `@McpDomain` annotations are on classes in ops/service, so
the processor must run there.

## Residual: ops/app

**After extraction, ops/app contains:**

```
ops/app/
  ├── pom.xml           — quarkus-maven-plugin + depends on casehub-ops-service
  └── src/main/resources/
      └── application.properties
```

No Java source files. The `quarkus-maven-plugin` with `extensions=true` in
`pom.xml` is the only thing that makes this a Quarkus application rather
than a library.

### ops/app pom.xml (after)

```xml
<dependencies>
    <dependency>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-ops-service</artifactId>
        <version>${project.version}</version>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-maven-plugin</artifactId>
            <version>${quarkus.platform.version}</version>
            <extensions>true</extensions>
            <executions>
                <execution>
                    <goals><goal>build</goal></goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

## Consumers

| Consumer | Depends on | Gets |
|----------|-----------|------|
| ops/app (standalone) | casehub-ops-service | Full ops capability as a standalone Quarkus app |
| Scaffold (aggregator) | casehub-ops-service | Full ops capability embedded in Scaffold's Quarkus app |
| Future aggregators | casehub-ops-service | Same — any Quarkus app can embed ops |

## Module Build Order

```
ops/api → ops/container → ops/service → ops/app
                                      → (scaffold, externally)
```

ops/service depends on ops/api and ops/container (existing modules).
ops/app depends only on ops/service.

## Testing

Existing tests in ops/app that use `@QuarkusTest` stay in ops/app — they
need the full Quarkus application context. The test source files move but
the test execution remains in ops/app's test phase since `@QuarkusTest`
requires the quarkus-maven-plugin.

Unit tests that don't need `@QuarkusTest` (plain JUnit) move to ops/service.

The `skip-quarkus-tests` profile stays in ops/app.

## Implementation approach

Use `ide_move_file` for all source file moves — this preserves IntelliJ's
reference tracking. The package rename (`app` → `service`) is handled by
`ide_refactor_rename` on the package.

Steps:
1. Create `service/` module with pom.xml
2. Move all Java source from app to service (ide_move_file)
3. Rename package `io.casehub.ops.app` → `io.casehub.ops.service`
4. Move Flyway migrations and META-INF resources
5. Slim down ops/app pom.xml to thin shell
6. Remove all Java source from ops/app
7. Update parent pom.xml module list (add service before app)
8. Verify: `mvn install` compiles, tests pass

## References

- `ops/app/pom.xml` — current Quarkus application configuration
- `OpsApplicationApi.java` — @McpDomain API pattern
- `StubActualStateAdapter.java` — @DefaultBean SPI stub pattern
- casehubio/casehub-ops#117 — deployment profiles (requires embeddable ops)
- casehubio/casehub-ops#119 — GraphQL generation (should target ops/service)
- casehubio/scaffold#53 — wire ops into scaffold (blocked by this)
