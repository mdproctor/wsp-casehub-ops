---
layout: post
title: "When Two Provisioners Walk Into a Router"
date: 2026-10-14
entry_type: note
subtype: diary
projects: [casehubio/casehub-ops]
tags: [desiredstate, claudony, deployment-target, spi, architecture]
series: issue-127-claudony-desiredstate-integration
---

# When Two Provisioners Walk Into a Router

The issues looked simple: wire claudony's fleet API into the desiredstate
reconciliation loop as a NodeProvisioner (#127) and ActualStateAdapter (#128).
Both S/Low. Map one API to another. Ship it.

I got about twenty minutes in before the architecture said no.

## The type collision

The desiredstate runtime routes provisioners by `NodeType`. Each provisioner
declares which types it handles via `handledTypes()`. The router —
`DefaultNodeProvisionerRouter` — builds a lookup table at construction time.
If two provisioners claim the same type, it throws `IllegalArgumentException`
and the app won't start. Same rule for `ActualStateAdapterRouter`.

The existing `DeploymentNodeProvisioner` already handles `casehub_application`.
A standalone `ClaudonyNodeProvisioner` handling the same type would crash on
startup. There's no priority mechanism, no preference annotation, no fallback
— it's a hard 1:1 constraint.

## The right extension point was already there

Issue #117 had already built multi-target routing into the deployment module.
`DeploymentApplicationNodeSpec` carries a `deploymentTargetId` field.
`CaseHubAppProvisionHandler` resolves the target by ID from CDI's
`Instance<ApplicationDeploymentTarget>`. The infrastructure for "same node
type, different backend" was sitting there waiting.

So instead of a standalone provisioner, we built `ClaudonyDeploymentTarget
implements ApplicationDeploymentTarget` with `targetId = "claudony"`. The
mapping is the same one the issues described — `deploy` calls `registerApp`,
`undeploy` calls `deregisterApp`, `status` reads health from the fleet API —
but it plugs into existing routing rather than fighting the type system.

## Suspend and resume

Claudony supports stateful lifecycle — suspend an app without destroying it,
resume it later. The `NodeProvisioner` interface has `suspend()` and `resume()`
methods, but `ApplicationDeploymentTarget` didn't. We extended the interface
with default methods that return `Failed` by default, so existing targets
(in-process, Podman, Kubernetes) are unaffected. The claudony target overrides
them to call `suspendApp` and `resumeApp` on the fleet API.

The `DeploymentNodeProvisioner` now delegates suspend/resume through the same
handler chain as provision/deprovision. When the spec carries
`deploymentTargetId = "claudony"`, it resolves to the claudony target and the
suspend goes through.

## A CDI trap

Claude caught something during the robustness audit: `ClaudonyDeploymentTarget`
with bare `@ApplicationScoped` would suppress the `@DefaultBean` fallback
(`NoOpDeploymentTarget`) and become the default target for *all* deployments.
Every app would route through claudony whether it was meant to or not. A
`@Named("claudony")` qualifier fixed it — the target is still discoverable via
`@Any Instance<>` but doesn't hijack the default injection point.

## What's in the module

The `claudony/` module is self-contained: a `ClaudonyFleetClient` interface,
an `HttpClaudonyFleetClient` implementation using Java's built-in `HttpClient`
(no Quarkus REST client dependency — keeps the module lightweight), mirror DTOs
for claudony's `ApplicationSpec` and `ApplicationInfo`, and the deployment
target itself.

The drift checker (`CaseHubAppDriftChecker`) got multi-target resolution —
when checking a `DeploymentApplicationNodeSpec` with a `deploymentTargetId`,
it resolves the correct target rather than always querying the default. This
means claudony-targeted apps get their drift checks from the claudony fleet
API, not the in-process or Podman backend.

## The design decision

This is recorded as ADR-0002. The short version: standalone provisioner was
the obvious approach from the issue text, but the type-collision constraint
made it impossible without removing `casehub_application` from the deployment
module (which handles seven other types) or going classpath-exclusive (which
forces all-or-nothing target selection). The `ApplicationDeploymentTarget` SPI
was the right level of abstraction — it was purpose-built for exactly this kind
of backend variation.
