---
layout: post
title: "Infra PoC — Three Layers and a Sealed Hierarchy"
date: 2026-06-14
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ops]
tags: [desiredstate, infra, architecture, design]
---

I've been circling the infrastructure domain in casehub-ops for a while — it's listed fourth in priority, behind deployment, compliance, and IoT. But it's also the most universally understood domain. Everyone knows Terraform. Everyone knows Ansible. If the desiredstate runtime's SPI contracts can handle infrastructure provisioning cleanly, they can handle anything.

The design question that mattered most was where to draw the boundary between CaseHub and the tools it wraps. CaseHub should own orchestration, governance, continuous reconciliation, and fault policy. Terraform should keep doing what it does well — holistic workspace planning with 1,500+ providers. Ansible should keep doing host configuration. The right design doesn't bridge the granularity gap between them; it respects it.

What emerged is three layers, each ignorant of the one below. The desiredstate runtime calls `NodeProvisioner` — it has no idea `InfraBackend` exists. The `InfraNodeProvisioner` dispatches to an `InfraBackend` by `backendId` — it has no idea whether Terraform or a cloud SDK is underneath. And each backend handles its own execution model internally.

The type design for `InfraNodeSpec` went through more iteration than I expected. The sealed hierarchy — K8s types, cloud types, wrapping types, a generic fallback — is straightforward. What took work was getting the WHAT/HOW separation right. A `K8sNamespaceSpec` describes a namespace. It shouldn't carry knowledge of which tool provisions it. Backend routing is an operational concern, not a domain concern.

The solution is `InfraDesiredNodeSpec` — a composite wrapper that carries the resource spec (WHAT) plus the backend ID (HOW), and implements the runtime's `NodeSpec`. The critical invariant: `InfraNodeSpec` does NOT extend `NodeSpec`. If it did, you could put a bare `K8sNamespaceSpec` directly on a `DesiredNode` and it would compile — then fail at runtime when the provisioner tries to unwrap it. Removing the `extends` makes the type system enforce the wrapper at compile time. The breakage is the point.

For task execution in standalone mode, I separated the "make this thing exist" step into its own `ResourceProvisioner` SPI. This keeps it orthogonal to orchestration and opens the door to LLM-generated provisioning scripts — where the advantage is zero maintenance (no 1,500 provider modules to keep current) but the question is trust. The generate → cache → human review → reuse lifecycle maps directly to casehub-ledger's Bayesian Beta trust model. An untested generated script has low trust and gets gated. A script with 50 successful executions runs automatically. That's a research track worth pursuing.

The spec went through seven review rounds. Claude caught a genuine compile-time safety issue — `InfraNodeSpec extends NodeSpec` would create a misuse path where bare specs could bypass the wrapper. Removing the extends tightened the type system in a way I wouldn't have caught from reading the code alone. The fault policy also got corrected: the initial design removed nodes from the graph on any provision failure, but the runtime's `FaultType` enum has no transient/permanent distinction. Removing a node that just had a network timeout is wrong. The safe default is no graph mutation — let the runtime handle retry.

The desiredstate runtime evolved independently during this work — the SPIs changed from reactive to blocking, `ProvisionResult` simplified, `GoalCompiler` now takes a factory instead of raw constraint parameters. Adapting to a moving target validated something about the architecture: because the infra module's internal SPIs (`InfraBackend`, `ResourceProvisioner`) are decoupled from the runtime types, the adaptation was entirely in the dispatcher layer. The backends and domain types didn't change at all.

The PoC validates the SPI contracts with 96 tests covering the full lifecycle: declare goals in YAML, compile to a typed graph, dispatch provisioning to the right backend, read state, detect drift, evaluate fault policy. Everything runs in-memory — no real K8s cluster, no Terraform CLI. The research question was whether the model works, not whether the integrations do. It does.

The interesting next step isn't Terraform or Ansible — it's the standalone backend with LLM-generated provisioning scripts. Every other infrastructure tool maintains a library of static provider implementations. CaseHub could generate them on demand, cache what works, and let trust accumulate through the ledger. The architecture already supports this cleanly because task execution is orthogonal. The question is whether earned trust on generated artifacts can match implicit trust on maintained static providers. That's genuinely worth finding out.
