---
layout: post
title: "Deployment Module: From Registration to Configuration"
date: 2026-06-17
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ops]
tags: [deployment, desiredstate, drift-detection, provider-config]
---

The deployment module from last session registered things — agents in eidos, channels in qhorus, case definitions in the engine. Declarative topology with reconciliation. But it stopped at the registration boundary: "this agent exists" was as far as the YAML went. Not "this agent runs with these tools, this model, this system prompt."

That gap is what separates a topology declaration from a deployment declaration. A real CaseHub application has agents configured for specific providers (claudony session settings, openclaw gateway URLs), case definitions loaded from YAML files with capabilities and workers and bindings, and trust policies that need to be compared field-by-field when someone changes them.

The core design question was how to carry provider-specific configuration without the deployment module interpreting it. The answer: `ProviderConfig(String providerName, Map<String, Object> config)` — a typed marker wrapping an opaque payload. The deployment module stores it and makes it queryable. Claudony and openclaw read it when they're ready. No coupling in either direction.

Case definition loading raised an interesting problem. I put `CaseDefinition` directly on the `CaseTypeNodeSpec` record, which seemed natural until code review caught that `CaseDefinition.hashCode()` only uses namespace, name, and version. Capabilities, workers, bindings, goals — all invisible to hash comparison. The spec hash drift detection I'd just built would silently miss every content change. The fix was to carry the raw YAML as `Map<String, Object>` instead — correct `hashCode()`, immutable, and the handler constructs the domain object at provision time. A second catch: `Map.copyOf()` throws NPE on null values, which YAML parsers commonly produce. `Collections.unmodifiableMap(new LinkedHashMap<>())` preserves nulls while staying immutable.

The drift detection refactoring was the most architecturally significant change. The old `DeploymentActualStateAdapter` had hardcoded per-type checking methods — `checkAgentStatus()`, `checkChannelStatus()`, each with its own comparison logic baked in. The new design extracts this into a `NodeDriftChecker` SPI with four default implementations. The adapter discovers checkers by node type and delegates. Two layers: external SPI check first (does the thing exist and match?), then spec hash comparison for PRESENT nodes only (did the declaration change since last provision?).

The layer ordering matters. I had it backwards initially — spec hash first, then external. On first run there are no stored hashes, so `hasDrifted()` returns true for everything. Every node reports DRIFTED when they should report ABSENT. External truth first, internal bookkeeping second.

A deeper issue surfaced during review: `TransitionPlanner` has no code path for DRIFTED. It provisions ABSENT/UNKNOWN nodes and deprovisions PRESENT-not-in-desired nodes. DRIFTED falls through both checks — silently ignored. I investigated whether graph mutations could work around this. They can't: `RemoveNode` doesn't trigger deprovision because the planner checks `status == PRESENT` specifically, and DRIFTED is not PRESENT. The correct fix is a one-line change in `TransitionPlanner` — treat DRIFTED like ABSENT. Filed as casehubio/casehub-desiredstate#38. Until it lands, drift detection provides OTel observability but not self-healing.

The `NodeDriftChecker` SPI lives in `casehub-ops-api`, not `casehub-desiredstate-api`. The infra module uses a completely different decomposition — `InfraBackend.readState()` delegates by backend, not by node type. If drift checking were a platform concept, both domains would use it. They don't. Foundation repos that want richer drift detection can ship optional bridge modules (`casehub-eidos-desiredstate`, `casehub-qhorus-desiredstate`) that override the defaults at higher CDI priority. Six companion issues filed across the ecosystem.
