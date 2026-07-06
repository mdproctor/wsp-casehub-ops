---
layout: post
title: "Everything Is a Case"
date: 2026-07-06
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ops]
tags: [service-as-case, desired-state, cdi, design-review]
series: issue-29-service-lifecycle-ops-console
---

The question that started this work was whether CaseHub's Case abstraction could model a long-lived service — something deployed for months or years, not hours or days. A service that accumulates CVEs, upgrades, incidents, scaling events, and compliance checks across its entire lifetime.

The answer turned out to be straightforward. Case already handles it. WAITING has no timeout. There's nothing in the engine that assumes a case should resolve quickly — no staleness checks, no SLA monitoring on duration, no scheduler scanning for old WAITING cases. A service case enters WAITING on deploy and stays there until decommission. That might be a month. It might be ten years. The model doesn't care.

## The shape that fell out

One application case per managed app. The services inside it are nodes in a desired-state graph, not sub-cases. When something happens to a service — a CVE, an upgrade request, a scaling trigger — a child case spawns via the engine's declarative binding mechanism. Signal the case context, binding evaluates, child case opens. The service's operational history is the sequence of child cases opened against it. Cross-service coordination (rolling upgrade across four microservices) goes through FlowWorker.

This means the deployment topology and the case lifecycle are separate concerns that compose cleanly. The ReconciliationLoop manages actual-vs-desired for infrastructure. The engine manages the case lifecycle. They share the same application — one owns the physical state, the other owns the accountability trail.

## What the design review caught

I ran an adversarial review against the spec — five rounds, 28 issues raised, all 28 verified and fixed. The most significant finding: the original spec had the app module pulling in casehub-ops-infra, casehub-ops-deployment, and casehub-ops-compliance as runtime dependencies. That violates the single-domain CDI constraint. Each of those is a domain module with its own SPI implementations — you can't stack them on the same classpath.

The fix was cleaner than the original. The app implements the desiredstate SPI quad directly — its own GoalCompiler, ActualStateAdapter, NodeProvisioner, FaultPolicy, EventSource. It uses the K8s spec types from casehub-ops-api (shared types, not domain logic) and wraps them in a KubernetesBackend. No domain modules on the classpath. The app is the application, not a consumer of domain libraries.

The decommission flow was another good catch. I'd originally had it stop the ReconciliationLoop, then deprovision resources manually. The reviewer pointed out this bypasses the entire desiredstate architecture — the same architecture that handles fault recovery, dependency ordering, and approval gates. The fix: deploy an empty graph. The TransitionPlanner sees all PRESENT nodes as orphans and generates DEPROVISION steps. The loop handles the rest, including faults. Decommission is just another reconciliation.

## CDI wiring: the wrong instinct

The first implementation attempt excluded all engine and desiredstate runtime beans via `quarkus.arc.exclude-types`. The reasoning was logical — those beans have unsatisfied SPI injection points, so remove them. But excluding a bean creates new unsatisfied dependencies for everything that injects it. The cascade is worse than the original problem.

The correct fix is additive: provide `@DefaultBean` stub implementations for each SPI. The stubs return Success/empty/ABSENT — enough for CDI to wire the graph. Phase 2 replaces them with real Kubernetes-backed implementations, and `@DefaultBean` yields automatically.

One undocumented quirk: `FaultPolicyEngine` injects `List<FaultPolicy>` as a plain Java List, not `@All Instance<FaultPolicy>`. Quarkus Arc doesn't auto-collect beans into plain List injection points — it needs a `@Produces` method that bridges `Instance<FaultPolicy>` to `List<FaultPolicy>`. This is the kind of thing that costs an hour if you don't know it exists.

## Where this is now

The foundation is in place. The app module has domain models, JPA persistence, a goal compiler that transforms service definitions into K8s desired-state graphs, and a REST API covering the full surface — application CRUD, cluster management, deployment, case visibility, approvals, security, reconciliation. Key paths are wired to real services; the rest return well-formed stubs.

The goal compiler is the interesting piece architecturally. It takes an application-level `ServiceDefinition` (name, image, replicas, ports, dependencies) and produces infrastructure-level K8s specs — one Deployment, one Service, one optional Ingress per microservice, with graph edges encoding the dependency order. Multi-cluster works by running `compileForCluster()` once per target cluster, each producing a cluster-specific graph keyed by a composite `tenancyId:clusterId`. Each cluster gets its own ReconciliationLoop TenantLoop. Per-cluster eventual consistency — no cross-cluster coordination, same as ArgoCD and Flux.

Phase 2 is the Kubernetes integration — real fabric8 provisioners, actual drift detection against live clusters, startup recovery for the ReconciliationLoop. That's when this stops being a skeleton and starts being an operational system.
