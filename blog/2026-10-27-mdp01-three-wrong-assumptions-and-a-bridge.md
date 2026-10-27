---
layout: post
title: "Three Wrong Assumptions and a Bridge"
date: 2026-10-27
entry_type: note
subtype: diary
projects: [casehubio/casehub-ops]
tags: [desired-state, topology, container-images, multi-env, testing, design-review]
series: issue-206-unified-service-url-resolution
---

# Three Wrong Assumptions and a Bridge

The session started with a clean goal: make the multi-env topology tests deploy real CaseHub services instead of nginx:alpine placeholders. The infrastructure from #203 proved deployment works across LOCAL, K8S, and PODMAN, but everything deployed was a dummy webserver. The whole point of multi-env testing is running the actual thing.

Scoping was the first decision. The issue listed six services needing container images. Only claudony has a buildable app module — eidos, engine, work, and ledger are library projects without standalone applications. Manifest doesn't even exist as a repo. I scoped to claudony only. One real service proves the pattern; the rest adopt it when they have app modules.

Then came the first wrong assumption: Jib for building images. I'd picked Google's `jib-maven-plugin` as a framework-neutral build tool that works for both Quarkus and Spring services. Claude ran a decision review and came back with a blocker I'd missed entirely: claudony's Dockerfile installs `tmux` via `apk add`. Jib can't execute `RUN` commands — it layers files onto a base image without a shell. The framework-neutral choice was unusable for the only service in scope.

The pivot was obvious. Claudony already has a working Dockerfile. Use it. Dockerfiles are the most framework-neutral option anyway — they work identically for Quarkus and Spring, handle system packages, and the one we need already exists.

With the image build sorted, we wired it into the test infrastructure. `ImageResolver` checks whether a real image exists locally and fails fast if it doesn't — no silent fallback to nginx:alpine unless you explicitly opt in. The fail-fast pattern matches `ServiceProcess.resolveJar()`, which already throws when a service JAR is missing. Consistent behaviour: if you haven't built it, the test tells you how.

For K8S, the `K3dClusterExtension` now exports images from the local Docker daemon and imports them into k3s's containerd via tar. For Podman, removing the `ImageOverridingTarget` wrapper lets the real image flow through.

That landed as #205. Then the next question: can we actually verify these services work?

The answer was no. LOCAL runs full functional assertions — health checks, LLM endpoint exercise. K8S and PODMAN just check the deployment exists. "Container is present" is not the same as "service is healthy and responding."

I created #206 to fix this: add `serviceUrl()` to `ApplicationDeploymentTarget` so the test can resolve a reachable URL per environment, wait for health, and assert behaviour — identically across all three.

Claude ran a spec review and caught the second and third wrong assumptions.

Wrong assumption two: claudony proxies HTTP traffic to deployed services. It doesn't. The fleet API registers apps as metadata entries — it's a fleet manager, not a reverse proxy. In LOCAL mode, the actual service processes are started by `ServiceClusterExtension`, and their URLs come from `ServiceProcess.baseUrl()`. The `ClaudonyDeploymentTarget` never sees them. The fix: track URLs in the target during deployment, with `ServiceClusterExtension` injecting process URLs after startup.

Wrong assumption three: shell-exec `kubectl port-forward` inside the k3s testcontainer. This doesn't work because Testcontainers only maps ports declared before container start — dynamically opened ports inside the container aren't reachable from the host. The correct approach is fabric8's `KubernetesClient.portForward()` API, which tunnels through the K8s API server that's already exposed.

Three design assumptions, three catches by adversarial review — two from spec review, one from decision review. The design is stronger for it. Each pivot was clean: the wrong approach had a specific technical reason it couldn't work, and the replacement was obvious once the constraint was visible.

The revised #206 spec has `serviceUrl()` using fabric8 port-forward for K8S, container port mapping for Podman, and URL injection for LOCAL. `ServiceHealthWaiter` extracts the health-polling logic from `ServiceProcess` into a reusable utility. `DeployConfig` gets a `healthPath` field so placeholder services (nginx:alpine on `/`) and real services (`/q/health/ready`) both work. The test becomes environment-agnostic: deploy → resolve URL → wait for health → assert. The only remaining branch is `FleetLlmExerciser` on LOCAL — an in-JVM exercise that can't work against containerised services by design.
