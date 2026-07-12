---
layout: post
title: "When the Watch Sees It First"
date: 2026-07-12
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ops]
tags: [kubernetes, fabric8, drift-detection, event-source]
---

The ops console's drift detection had a 5-minute blind spot. The ReconciliationLoop polls on a configurable resync interval — good enough for convergence, but if someone runs `kubectl delete` on a managed deployment, you're waiting up to 5 minutes before the system notices. For production workloads, that's too slow.

We closed three issues in a single branch, all touching the same infrastructure from different angles. The backbone was #40 — adding an active `K8sWatchManager` that subscribes fabric8 Watch instances per managed namespace and cluster, filtered by the `managed-by=casehub-ops` label. When a managed resource is modified or deleted externally, the Watch fires a `StateEvent` into the existing `KubernetesEventSource`. The ReconciliationLoop picks it up immediately instead of waiting for the next poll.

The interesting design question was NodeId resolution. Watch events arrive as raw fabric8 `HasMetadata` objects — a Deployment named "inventory" in namespace "casehub". We need to map that back to the desiredstate runtime's `NodeId`. The clean solution would be annotations on every managed resource (`casehub.io/node-id`), but that requires threading the NodeId through the handler chain, which none of the handlers currently receive. We went with convention-based derivation instead — `clusterId:resourceName:resourceSlug` — matching the `ApplicationGoalCompiler`'s naming. Both the compiler and the Watch manager live in the same module, so the coupling is contained. An annotation-based approach is the right next step if the naming convention ever needs to change.

The other design choice: we don't filter our own modifications. When the provisioner applies a resource, the Watch fires a MODIFIED event, which triggers a reconciliation check that finds no drift and does nothing. One wasted comparison per provisioning action. The alternative — maintaining a set of "recently modified by us" resource versions to suppress — adds complexity for a case that's already handled correctly by the reconciliation loop's actual-vs-desired comparison.

Two supporting changes went in on the same branch. #41 fixed `InfraBackend.readState()` and `detectDrift()` — both took only a `NodeId`, which is opaque. External backends (K8s, Terraform) need the resource's namespace, name, and kind to query actual state. We added `InfraNodeSpec` as a second parameter. #46 swept up five Phase 2 review findings: key parsing that broke on colons in tenancyId, a duplicate `parseServices()` method, a K8s Service selector coupled to the full label set instead of a minimal `app` subset, and missing `@Scheduled` timeout methods for stuck deployments and decommissions.

The selector fix is worth calling out. `K8sServiceHandler` was using `spec.labels().values()` for both the Service metadata labels and the pod selector. That includes `managed-by: casehub-ops`. If that label is ever added or removed, the selector changes and disconnects the Service from its pods. We added a separate `Labels selector` field to `K8sServiceSpec` — the goal compiler now passes `{app: serviceId}` as the selector, independent of the full label set.
