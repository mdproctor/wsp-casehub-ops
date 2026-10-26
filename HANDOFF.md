# Handoff — casehub-ops

## Active Work

**Branch:** `main` (clean)
**Last landed:** #203

## What to Do

`work start #205` — container image build for CaseHub services. This is the prerequisite that turns multi-env deployment from nginx:alpine placeholders into real running services on K8s and Podman.

## Context

**Multi-env deployment is fully wired and proven:**
- #150 (epic, 4 issues) — environment abstraction layer, `TargetEnvironment` enum, `EnvironmentAwareNodeProvisioner`, `KubernetesDeploymentTarget`, `K3dClusterExtension`, `PodmanEnvironmentExtension`, Maven profiles
- #203 — Level 01 deploys to all three environments (LOCAL, K8S, PODMAN), verified passing

**What's missing:** real container images. K8s and Podman tests use `nginx:alpine` as an image override because CaseHub service images don't exist locally. #205 adds `quarkus-container-image-jib` to each service repo so `mvn package -Dquarkus.container-image.build=true` produces local images.

**Services needing Jib:** claudony, eidos, engine, manifest (Level 01 minimum). Then work, ledger for higher levels. Each is XS/Low — add the Jib dependency and image config to `application.properties`.

**Infrastructure available:**
- Podman machine running (`podman-machine-default`, 7 CPUs, 8GB RAM)
- `DOCKER_HOST` set to Podman socket — testcontainers uses Podman as Docker replacement
- k3s testcontainer works on Podman (verified in #203)
- Netty native kqueue transport configured for domain sockets

**Claudony issues resolved this session:**
- #294 — orphan config properties (Jandex index missing)
- #295 — connector config defaults in wrong file
- #296 — defaults moved to main application.properties
- #297 — root cause: `casehub-platform-registry-inmem` JAR had no Jandex index, `NoOpRegistryService` `@DefaultBean` silently discarded all registrations

## Key files

- `testing/src/main/java/io/casehub/ops/testing/env/` — `TargetEnvironment`, `EnvironmentAwareNodeProvisioner`
- `topology-tests/src/test/java/io/casehub/ops/topology/env/` — `TargetEnvironmentFactory`, `KubernetesDeploymentTarget`, `K3dClusterExtension`, `PodmanEnvironmentExtension`, `ImageOverridingTarget`, `HelloLlmMultiEnvTest`
- `docs/specs/issue-150-multi-env-deployment/` — design spec and decisions
- `docs/specs/issue-117-deployment-profiles-bootstrap/` — production deployment profiles spec (#150 builds on this)
