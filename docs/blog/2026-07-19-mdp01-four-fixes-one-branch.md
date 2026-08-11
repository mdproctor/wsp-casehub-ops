---
layout: post
title: "Four Fixes, One Branch"
date: 2026-07-19
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ops]
tags: [approval, desiredstate, cdi, jackson, k8s, authorization]
---

The approval workflow from yesterday's session left a trail of known debt — small items, each S-scale, none individually worth a branch. I batched four of them onto one branch and worked through them sequentially.

The first fix crossed repos. The desiredstate runtime's three `Cdi*` bridge classes — `CdiMergedEventSource`, `CdiNodeProvisionerRouter`, `CdiActualStateAdapterRouter` — lacked the `protected` no-args constructors CDI needs for `@ApplicationScoped` proxy generation. Ops had been working around this by excluding all three and providing identical `App*` clones with the missing constructors. The workaround had a bonus bug: `AppNodeProvisionerRouter` dropped the `PreferenceProvider` injection that the original `CdiNodeProvisionerRouter` wired, silently losing preference-based resync interval overrides. Adding no-args constructors to the desiredstate classes, rebuilding, and deleting the `App*` clones fixed it and restored the lost feature.

The `InMemoryPlanStore` was next — a `ConcurrentHashMap` that evaporated on restart, forcing humans to re-approve everything. JPA-backed persistence was straightforward except for the serialization layer. `ApprovalPlan` holds a `NodeSpec` field, and `NodeSpec` is an upstream interface with multiple implementations across ops — `InfraDesiredNodeSpec` wrapping sealed `InfraNodeSpec` subtypes, `DeploymentNodeSpec` with its own sealed subtypes. Jackson needs explicit type info for polymorphic round-trip fidelity. I added `@JsonTypeInfo` directly to the sealed interfaces — and immediately broke the YAML config file parser, which uses the same types without discriminator properties. The fix was to scope the type info to a dedicated `ObjectMapper` via `addMixIn()`, leaving the global mapper untouched. Clean separation, but the failure mode isn't obvious — the connection between "I annotated a data type" and "YAML config loading broke" requires knowing both serialization paths share the same types.

Context-aware K8s risk classification was the cleanest change. The existing `classifyRisk()` method operated on `NodeType × StepAction` only — deleting a `dev` namespace carried the same CRITICAL classification as deleting `production`. The spec data was right there but unused. I extracted the namespace from the `InfraNodeSpec`, added configurable `critical-namespaces` and `production-namespaces` lists that set a risk floor, and wired it through Quarkus config properties.

Approval authorization closed the set. `ApprovalResource` checked tenancy isolation — anyone with the right tenancy could approve a CRITICAL operation. An `ApprovalAuthorizer` SPI in ops-api, a config-driven default mapping risk levels to required roles, and a `CurrentPrincipal` injection into the REST endpoints adds the gate. When no roles are configured, everything is authorized — backward compatible with existing deployments that haven't set up RBAC.

Along the way I hit a pre-existing Quarkus issue worth noting: after a `mvn clean`, all `@QuarkusTest` tests fail because Hibernate 6.6 generates `check ((dtype in ()))` with an empty discriminator list for JOINED inheritance entities whose subtypes live in external jars. H2 rejects the syntax, tables don't get created, and Panache throws a misleading `implementationInjectionMissing` error. The stale augmentation cache masked it on incremental builds — only `clean` exposed it. Not caused by my changes, not fixed here, but worth tracking.
