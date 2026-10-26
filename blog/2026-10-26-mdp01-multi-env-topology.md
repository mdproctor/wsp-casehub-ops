---
layout: post
title: "Same YAML, different worlds"
date: 2026-10-26
entry_type: note
subtype: diary
projects: [casehubio/casehub-ops]
tags: [topology, kubernetes, podman, environment-abstraction, testing]
series: issue-150-multi-env-deployment
---

The topology test suite has been hardwired to claudony since day one. Every `casehub_application` node in the YAML deploys through the claudony fleet API — which means the YAML is implicitly coupled to a single deployment backend, even though the entire desiredstate architecture was designed to be backend-agnostic.

The goal for #150 was simple to state: same topology YAML deploys to local (claudony), Kubernetes, or Podman, selected at test runtime via a system property. No YAML changes.

## The discovery that changed the scope

I expected to design the multi-environment abstraction from scratch. Instead, while exploring the codebase for the brainstorm, Claude surfaced the #117 deployment profiles spec — a design doc from an earlier session that already defined the full production architecture: three deployment profiles (`local-dev`, `local-distributed`, `enterprise`), an `ApplicationDeploymentTarget` SPI with `targetId()` routing, `PodmanDeploymentTarget` already implemented, `KubernetesNodeProvisioner` already in `service/`, and `BootstrapDomainRegistrar` wiring domains by profile.

This collapsed the design space. The production multi-environment mechanism was already designed. What #150 actually needed was the test-level equivalent: manually construct the right `ApplicationDeploymentTarget` based on a system property, without CDI.

## The architecture

Two layers, split by dependency weight:

**In `testing/` (lightweight, depends only on `casehub-ops-api`):**
- `TargetEnvironment` enum — reads `casehub.topology.target-environment`, defaults to `LOCAL`
- `EnvironmentAwareNodeProvisioner` — implements `NodeProvisioner`, delegates `casehub_application` provisioning to whichever `ApplicationDeploymentTarget` the environment provides

**In `topology-tests/` (heavy, depends on claudony, container, fabric8):**
- `TargetEnvironmentFactory` — constructs the right target per environment
- `KubernetesDeploymentTarget` — creates K8s Deployment + Service via fabric8
- `K3dClusterExtension` — JUnit extension managing testcontainers k3s lifecycle
- `PodmanEnvironmentExtension` — JUnit extension managing Podman network + container cleanup

The split was a self-review catch. My first plan put everything in `testing/`, which would have pulled fabric8, testcontainers-k3s, and the Podman client into a module that currently depends on almost nothing. The lightweight/heavy split keeps `testing/` clean while letting `topology-tests/` — which already carries these dependencies — own the wiring.

## What makes it work

The `EnvironmentAwareNodeProvisioner` is a near-copy of the existing `ClaudonyNodeProvisioner` in topology-tests, but parameterised on the target. Where the old provisioner hardcoded `ClaudonyDeploymentTarget`, the new one accepts any `ApplicationDeploymentTarget`:

```java
public class EnvironmentAwareNodeProvisioner implements NodeProvisioner {
    private final ApplicationDeploymentTarget target;

    @Override
    public ProvisionResult provision(DesiredNode node, ProvisionContext context) {
        CaseHubApplicationSpec appSpec = extractAppSpec(node);
        DeployResult result = target.deploy(
            appSpec.image(),
            new DeployConfig(appSpec.applicationId(), appSpec.replicas(),
                appSpec.config(), Map.of()));
        // ...
    }
}
```

Maven profiles (`-Ptarget-k8s`, `-Ptarget-podman`) set the system property via surefire, so environment selection is a build command: `mvn test -pl topology-tests -Ptarget-k8s`.

The `KubernetesDeploymentTarget` follows the existing `Fabric8NodeProvisioner` pattern — `createOr(NonDeletingOperation::update)` for idempotent apply, label-based selector matching, standard Deployment + Service pair per application. The k3d extension uses testcontainers-k3s, which was already in the pom.

## What's ahead

The infrastructure is in place but no topology level has actually been deployed to K8s or Podman yet — that requires Docker running for k3d and a Podman socket for the container target. The next step is wiring an existing level (01 or 04) to use the factory and verifying it passes on all three environments. That's where the image loading strategy gets real: wrapping `quarkus-run.jar` in a lightweight container and importing it into k3d.

The larger question is whether the container module's `PodmanDeploymentTarget` — designed for individual app containers — handles the topology test's multi-service orchestration pattern well enough, or whether the test harness needs its own deployment coordination on top.
