---
layout: post
title: "Deployment Module: The Graph That Wasn't"
date: 2026-06-16
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ops]
tags: [deployment, desiredstate, architecture]
---

I started the deployment module expecting to mirror the infra module's design. Three-layer architecture, typed specs, backend abstraction, dependency graph — the infra PoC established the pattern, and the deployment domain should follow it.

The first assumption to fall was the backend abstraction. Infra routes provisioning through backends (Terraform, Ansible, standalone) because the same resource type can be handled by different tools. A k8s_deployment can go through Terraform or through standalone provisioning. The deployment domain has no such choice: agents always register in eidos, channels always create in qhorus, case types always deploy to engine, trust policies store in memory. One target per node type. A backend layer would add a dispatch indirection that always resolves to the same place.

The second — and more interesting — assumption was that the dependency graph would have edges. Infra has natural dependencies: a k8s_deployment depends on a k8s_namespace. I assumed the same structure: agents depend on channels, case types depend on agents. I was told to investigate from first principles.

The investigation turned up nothing. `AgentDescriptor` has no channel fields. `CaseDefinition` lists workers but doesn't bind them to agents by ID — that happens at case instantiation through capability matching. Channels don't reference agents at all. Trust policies are per-capability, not per-agent. Every entity type is independently creatable. All binding is runtime.

The graph is flat. No inherent edges. Everything provisions in parallel. Dependencies only exist if the user explicitly declares `dependsOn` in the YAML.

This changes what the deployment module is for. It's not an orchestrator managing provisioning order — it's a topology declaration with reconciliation. Declare what should exist, detect drift, self-heal. The value is the continuous reconciliation and audit trail, not the ordering.

The spec went through four review rounds. Each round caught real issues: CaseDefinitionRegistry throws on not-found (no existence query), ChannelService's type constraint setter normalises null to Set.of() creating a lossy round-trip, channel drift detection must exclude immutable fields to avoid infinite reconciliation loops, the desiredstate SPI itself lacks tenancyId on readActual() (filed as casehubio/casehub-desiredstate#36).

Implementation was 12 commits, 32 tests, 28 new files. Four handlers, a compiler, an actual state adapter with mutable-field drift detection, a node provisioner with sealed-type dispatch. The whole thing fits cleanly alongside the infra module — same SPIs, different domain assumptions.
