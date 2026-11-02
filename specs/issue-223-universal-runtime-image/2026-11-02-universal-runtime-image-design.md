# Universal CaseHub Runtime Image — Design Spec

**Issue:** casehubio/casehub-ops#223
**Branch:** issue-223-universal-runtime-image
**Date:** 2026-11-02

---

## Problem

Level 01 topology tests reference four fictional per-service images (`casehubio/manifest:latest`, `casehubio/eidos:latest`, `casehubio/engine:latest`, `casehubio/my-llm-app:latest`) that don't exist — manifest is a library and eidos is a framework, neither is a standalone deployable. The `allow-image-fallback=true` workaround deploys `nginx:alpine` containers, proving deployment plumbing works but exercising zero CaseHub functionality. Scaffold#83 (Deploy Level 01 from UI) is blocked for the same reason.

## Solution

Build a single universal CaseHub runtime image with all platform and ops modules on the classpath. At deploy time, the config map provided in the topology YAML controls which functionality is exercised — no explicit module activation mechanism is needed because CaseHub modules are already designed to be inert without declared goals.

---

## Architecture

### New `runtime/` module

A new `runtime/` module in casehub-ops — a Quarkus application that aggregates every CaseHub platform module plus all ops domain modules into a single deployable.

```
runtime/
  pom.xml
  src/main/
    resources/
      application.properties
    docker/
      Dockerfile.jvm
```

**Compile-scope dependencies (platform stack):**

| Artifact | Family |
|----------|--------|
| `casehub-platform` | Platform runtime |
| `casehub-platform-bootstrap` | Platform startup |
| `casehub-platform-expression` | Expression evaluation |
| `casehub-engine` | Case engine runtime |
| `casehub-engine-rest` | Engine REST endpoints |
| `casehub-engine-work-adapter` | Work adapter |
| `casehub-engine-planning` | GOAP planning |
| `casehub-engine-persistence-memory` | In-memory persistence |
| `casehub-engine-scheduler-quartz` | Quartz scheduler |
| `casehub-eidos` | Agent identity runtime |
| `casehub-qhorus` | Agent communication runtime |
| `casehub-work` | Work/task runtime |
| `casehub-desiredstate` | Desired-state runtime |
| `casehub-desiredstate-yaml` | YAML goal frontend |
| `casehub-desiredstate-work` | Work integration |
| `casehub-agency-summarisation-yaml` | Summarisation |

**Compile-scope dependencies (ops stack):**

| Artifact | Family |
|----------|--------|
| `casehub-ops-service` | Ops service layer (embeds all SPIs, entities, migrations) |
| `casehub-ops-deployment` | Deployment domain SPI quad |
| `casehub-ops-container` | Container/Podman domain |
| `casehub-ops-compliance` | Compliance domain |
| `casehub-ops-claudony` | Claudony fleet management |

**Quarkus extensions:** `quarkus-rest`, `quarkus-rest-jackson`, `quarkus-hibernate-orm-panache`, `quarkus-flyway`, `quarkus-jdbc-h2`, `quarkus-smallrye-graphql`.

The module uses `quarkus-maven-plugin` for application packaging (same as `app/`). The `application.properties` file provides minimal defaults — HTTP port, datasource config for H2 in-memory (overridable via env vars for production databases).

### No activation mechanism

CaseHub modules are Jandex libraries that activate by CDI classpath discovery. They are designed to be inert without configuration:

- `GoalCompiler` implementations produce an empty `DesiredStateGraph` when no goal entries are declared
- `ReconciliationLoop` is started explicitly via API call, not auto-started at boot
- `EventSource` streams events only when subscribed
- `CrossDomainCompositionEngine` handles multi-domain coexistence on the same classpath
- `NodeProvisioner` implementations route by `NodeType` — unused types receive no calls

Runtime behavior is controlled entirely by the `config` map in `CaseHubApplicationSpec`, which flows as env vars into the container via `DeployConfig` → `PodmanDeploymentTarget.deploy()`. No new activation machinery is needed.

### One fat image

The universal image includes both platform and ops modules. Level 01 uses only platform functionality (engine, eidos for agent identity). Later topology levels (05+ config-drift, 06 lifecycle) deploy containers that themselves run ops/desiredstate to manage other systems. A single image avoids two-variant complexity (two builds, two pushes, two versions, topology YAML must pick the right one).

Unused modules impose disk cost only — CDI bean instances are lightweight and idle without goals.

---

## Level 01 Topology Rewrite

**Before:** 4 fictional service nodes with non-existent per-service images and a linear dependency chain.

**After:** 1 real node with the universal runtime image.

```yaml
# Level 01: Hello LLM
# Methodology step: DEPLOY — First App

desiredState:
  namespace: casehub
  name: hello-llm

nodes:
  my-llm:
    type: casehub_application
    spec:
      applicationId: my-llm
      image: casehubio/casehub-runtime:latest
      deploymentTarget: claudony
      replicas: 1
      config:
        llm.capability: fast
```

**Metadata update:** Description changes from "Deploy a complete LLM app with four services" to "Deploy a CaseHub LLM agent".

**Playbook update:** Single step: `provision-node: my-llm`.

The topology file in `service/src/main/resources/META-INF/desiredstate/topologies/01-hello-llm/` must be updated to match.

---

## ImageResolver Changes

Add universal image support alongside existing per-service resolution.

```java
public static final String UNIVERSAL_IMAGE_NAME = "casehub-runtime";

public static String resolveUniversal(String version) {
    return resolve(UNIVERSAL_IMAGE_NAME, version);
}

public static String resolveUniversal(String version, boolean allowFallback) {
    return resolve(UNIVERSAL_IMAGE_NAME, version, allowFallback);
}
```

Existing `resolve(serviceName, version)` API stays for backward compatibility. Tests that create `casehub_application` nodes for the universal image call `resolveUniversal()` instead.

The `ImageSource` enum and three-tier resolution (LOCAL → REGISTRY → FALLBACK) remain unchanged. The LOCAL tier checks for `casehubio/casehub-runtime:<version>` via `docker images -q`.

---

## Dockerfile

JVM-mode Dockerfile using the Quarkus fast-jar layout:

```dockerfile
FROM eclipse-temurin:21-jre-alpine

WORKDIR /app
COPY target/quarkus-app /app

EXPOSE 8080
ENTRYPOINT ["java", "-jar", "quarkus-run.jar"]
```

Config override via environment variables (standard Quarkus convention): `QUARKUS_HTTP_PORT`, `QUARKUS_DATASOURCE_*`, and any `casehub.*` properties converted to env var format.

---

## Build Flow

Local-only for this issue. CI/GHCR push is a follow-up.

```bash
# 1. Build the runtime module (produces fast-jar)
mvn --batch-mode install -pl runtime -am

# 2. Build the container image
docker build -t casehubio/casehub-runtime:latest -f runtime/src/main/docker/Dockerfile.jvm runtime/

# 3. Run topology tests with real image
mvn --batch-mode test -pl topology-tests -Pintegration-live
```

The topology tests use `ImageResolver.resolveUniversal("latest")` which checks for the local image. If not found, falls back to `nginx:alpine` when `allow-image-fallback=true` is set (existing behavior preserved).

---

## Scope

**In scope (this issue):**
1. New `runtime/` module with pom.xml aggregating all platform + ops deps
2. `Dockerfile.jvm` for JVM-mode image
3. Rewrite Level 01 topology YAML (both copies: topology-tests and service module)
4. Update Level 01 metadata and playbook
5. Add `UNIVERSAL_IMAGE_NAME` and `resolveUniversal()` to ImageResolver
6. Update ImageResolverTest

**Out of scope (follow-up issues):**
- CI pipeline: GitHub Actions workflow to build and push to `ghcr.io/casehubio/casehub-runtime`
- Rewrite of Level 02-15 topologies (each level gets its own issue)
- Scaffold integration (scaffold#83, scaffold#86)
- Native image build (Dockerfile.native)
- Per-service thin images (packaging optimisation for later)

---

## Testing

- `runtime/` module compiles and packages successfully (`mvn install -pl runtime -am`)
- Docker image builds from the fast-jar output
- Level 01 topology test passes with the real image (integration-live profile)
- ImageResolver.resolveUniversal() resolves correctly for LOCAL, REGISTRY, and FALLBACK tiers
- Existing topology tests (Level 02-15) continue to pass with `allow-image-fallback=true`

---

## Risks

| Risk | Mitigation |
|------|------------|
| Fat image size (all modules) | JVM mode + Alpine base keeps it reasonable. Size is a follow-up optimisation, not a blocker. |
| Dependency conflicts between platform modules | Managed via casehub-parent BOM. All modules already coexist in service/ module. |
| CDI ambiguity with multiple domain modules on classpath | `CrossDomainCompositionEngine` and `@DesiredStateQualifier` handle this — documented in ARC42STORIES.MD §2. |
| Build time increase | runtime/ is a thin Quarkus shell — most compile time is in existing deps. Incremental Maven builds skip unchanged modules. |

---

## References

- casehubio/casehub-ops#223 — issue
- `api/src/main/java/io/casehub/ops/api/deployment/CaseHubApplicationSpec.java` — casehub_application node spec with RuntimeConfig
- `api/src/main/java/io/casehub/ops/api/deployment/RuntimeConfig.java` — engine version + modules list
- `deployment/src/main/java/io/casehub/ops/deployment/handler/CaseHubAppProvisionHandler.java` — provisioning handler
- `container/src/main/java/io/casehub/ops/container/PodmanDeploymentTarget.java` — Podman deploy target
- `testing/src/main/java/io/casehub/ops/testing/env/ImageResolver.java` — image resolution
- `topology-tests/src/test/resources/topologies/deployment/01-hello-llm/` — current Level 01 topology
- `ARC42STORIES.MD` §2 — architectural constraints (CrossDomainCompositionEngine, handledTypes disjointness)
- `service/pom.xml` — existing platform runtime dependencies (validates coexistence)
