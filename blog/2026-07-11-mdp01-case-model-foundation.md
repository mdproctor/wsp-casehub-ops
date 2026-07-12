---
date: 2026-07-11
author: mdp
tags: [casehub-ops, case-model, drift-remediation, desiredstate]
---

# Case model foundation and drift remediation

Built the case model infrastructure for the ops console app — the Phase 3 foundation that
connects the casehub-engine case lifecycle to the desiredstate reconciliation loop.

The core pattern: `ApplicationCaseDescriptor` builds a long-lived `ops:application-lifecycle`
case with six context-change bindings. Each binding watches a JQ-filtered context path
(`.driftDetected`, `.cveDetected`, etc.) and spawns a child sub-case. The drift-remediation
child case is fully working with classify/remediate/escalate workers; the other five are
wired but stubbed.

The signal bridge was the design challenge. The original spec had the bridge subscribing to
`KubernetesEventSource` — but that's a passive inbound emitter, not an outbound observation
channel. The reconciliation loop's drift detection flows through `ReconciliationEventEmitter`
as CloudEvents. The adversarial design review caught this, and the fix uses the established
`@ObservesAsync CloudEvent` pattern from `DeploymentOutcomeTracker` and
`DecommissionCompletionHandler`.

Convergence detection uses `NODE_RECOVERED` CloudEvents rather than `RECONCILIATION_COMPLETED`
— the completed event only has aggregate counts, not per-node status. `NODE_RECOVERED` gives
exactly the granularity needed for tracking which drifted nodes have been remediated.

The credential resolver (#44) was the cleaner piece — the platform already has `CredentialResolver`
as an SPI with a config-based default. Wiring it into `K8sClientRegistry` was straightforward.
Made `trustCerts` per-cluster instead of hardcoded true while there.
