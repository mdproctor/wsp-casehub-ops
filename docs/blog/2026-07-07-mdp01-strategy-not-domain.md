---
layout: post
title: "Strategy, Not Domain"
date: 2026-07-07
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ops]
tags: [compliance, spi-design, evidence-collection]
---

The compliance module had six EvidenceCollector implementations. Five of them did the same thing: check whether a file exists and is recent. The only difference was the `controlType()` method returning a different string — `ACCESS_REVIEW`, `ENCRYPTION_AT_REST`, `DATA_PROCESSING`, `INCIDENT_RESPONSE`, `AI_RISK_ASSESSMENT`. Five classes with identical `collect()` bodies, each dispatched by domain name.

The design spec for #18 identified this as a conflation of two orthogonal concerns. `controlType` answers "what compliance domain is this?" — access review, encryption, log retention. `strategy` answers "how do you check it?" — file existence, directory scanning, certificate parsing, hash comparison. Routing collectors by domain meant every new control type required a new Java class, even when the verification logic was identical.

I renamed `controlType()` to `strategy()` on the `EvidenceCollector` interface and added a `strategy` field to `ComplianceControlSpec`. The service routes by strategy; the domain identity stays in `controlType` for reporting and posture scoring. New compliance controls that use an existing verification strategy now require YAML only — zero Java code.

The six stubs became four real collectors:

- **FileExistenceEvidenceCollector** — file exists and is fresh. Serves five control types.
- **LogDirectoryEvidenceCollector** — directory has recent activity and historical retention.
- **CertificateExpiryEvidenceCollector** — X.509 certificate validity with configurable warning threshold. Parses PEM or DER via `CertificateFactory`.
- **ConfigHashEvidenceCollector** — SHA-256 hash comparison against a declared baseline. Pluggable algorithm.

The service also gained a safety-net `try-catch` around `collector.collect()`. The SPI contract says collectors must never throw, but a rogue implementation shouldn't take down the reconciliation loop. Exceptions become `Unavailable("collector error: ...")` — the ledger records the failure, the control drifts, and the next cycle retries.

The branch also absorbed two upstream API changes that landed on main while it was paused: `FaultPolicy.onFault()` gained a third `ActualState` parameter, and `GoalCompiler.compile()` now returns `CompilationResult` instead of `DesiredStateGraph`. Both needed mechanical adaptation across compliance, infra, and iot modules — the kind of thing that surfaces when a branch sits paused for eight days while the runtime evolves.

The separation between domain and strategy feels like the right factoring. It's the same insight that the infra module applied with `InfraDesiredNodeSpec` — separating WHAT (the resource spec) from HOW (the backend). Here it's separating WHAT (the compliance control) from HOW (the verification method). When the same structural pattern appears independently in two domains, it's probably load-bearing.
