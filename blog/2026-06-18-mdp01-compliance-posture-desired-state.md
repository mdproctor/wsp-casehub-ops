---
layout: post
title: "Compliance Posture: From Audit Prep to Desired State"
date: 2026-06-18
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ops]
tags: [compliance, desiredstate, drift-detection, ledger, posture-scoring]
---

The compliance module started from a question that kept surfacing during the deployment work: what would continuous compliance look like if you treated regulatory controls the same way you treat infrastructure — as a declared desired state that drifts and self-heals?

The GRC market is ~$50B and growing fast, but the tools are all point-in-time. Vanta, Drata, Secureframe — they help you prepare for an audit. They don't tell you at 2am on a Tuesday whether your SOC2 posture just degraded because someone turned off encryption on a staging database. That's the gap.

## The two-dimensional control model

A compliance control has two faces. One is configuration assertion — "is encryption actually AES-256 right now?" The other is evidence freshness — "when did we last verify that?" Both can drift independently. The encryption might still be AES-256, but if nobody checked in 90 days, the auditor doesn't care what you believe is true.

I modelled both dimensions into a single `ComplianceControlSpec` that carries `evidenceMaxAgeDays` as a first-class field. Drift detection queries the ledger for the latest evidence entry. If it's older than the threshold — DRIFTED. If the outcome was FAIL or UNAVAILABLE — also DRIFTED. Fresh PASS is the only path to PRESENT. Conservative by design: if you can't verify, assume drift.

The configuration assertion side delegates to an `EvidenceCollector` SPI — a pluggable interface where real integrations (cloud APIs, identity providers, monitoring systems) would live. For now, six stubs that always return PASS. The SPI is the extension point; the stubs prove the architecture works.

## Generic spec, not sealed hierarchy

The deployment module uses a sealed hierarchy — `AgentNodeSpec`, `ChannelNodeSpec`, `CaseTypeNodeSpec`, `TrustPolicyNodeSpec` — because those types are structurally different. An agent has capabilities and a provider; a channel has semantics and allowed writers. They share almost nothing.

Compliance controls are the opposite. All six types share the same structure: framework mappings, evidence requirements, drift thresholds. The type-specific parts (`cipher: AES-256`, `retentionDays: 365`, `cadence: quarterly`) are configuration consumed by the evidence collector, not structural differences in the spec. A sealed hierarchy would have been six nearly-identical records with 1-3 unique fields each.

So `ComplianceControlSpec` is a single generic record with a `controlType` discriminator and `Map<String, Object> properties`. Adding a new control type means writing one `EvidenceCollector` implementation and defining the YAML schema — not cascading a new sealed variant through every switch expression.

## Evidence in the ledger

Evidence is stored as `ComplianceLedgerEntry extends LedgerEntry` — the tamper-evident audit trail that's the whole point of building this on CaseHub. Every evidence collection event gets its own Merkle chain (deterministic `subjectId` from `UUID.nameUUIDFromBytes("compliance:" + controlId)`), its own sequence numbers, its own hash frontier. The auditor doesn't trust your word that encryption was verified on March 15th — the cryptographic chain proves it.

The spec review process caught a critical gap here: `LedgerEntryRepository.save()` requires a non-null `subjectId`, `entryType`, and actor fields. The spec had "creates and persists a ComplianceLedgerEntry" without specifying any of them. At runtime, that's an `IllegalArgumentException` on the first evidence collection. The fix follows the `LedgerErasureService.buildReceipt()` pattern — deterministic subjectId, `LedgerEntryType.EVENT`, `ActorType.SYSTEM`.

## The posture service — the differentiator

The `CompliancePostureService` is what turns a control reconciliation loop into a compliance tool. It answers "what is our SOC2 compliance score right now?" by aggregating control statuses per framework through a five-category model: passing, failing, unavailable, stale, missing. An earlier version conflated FAIL and UNAVAILABLE into a single `failingControls` count — the spec review caught this. "2 controls failing" and "2 controls unverifiable" are different risk conclusions for an auditor.

Each control maps to multiple frameworks. `EncryptionAtRestControl` satisfies SOC2 CC6.1, GDPR Art.32, DORA Art.9, NIS2 Art.21, and ISO27001 A.8.24 simultaneously. You implement the control once; framework compliance is a projection over the control graph filtered by framework mapping. This is how real GRC works — the framework-centric alternative (separate graphs per framework with duplicated controls) would have been simpler to build and worse in every other way.

## What's still missing

The `TransitionPlanner` has no code path for DRIFTED — it only provisions ABSENT/UNKNOWN nodes and deprovisions unwanted PRESENT nodes. DRIFTED falls through silently. This means evidence collection works on first run (ABSENT -> PROVISION), but stale evidence never triggers re-collection. The compliance module degrades from continuous compliance to one-shot audit until `desiredstate#38` lands. The same prerequisite the deployment module documented — one-line fix, but it gates self-healing for every domain module.
