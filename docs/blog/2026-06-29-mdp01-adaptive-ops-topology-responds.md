---
layout: post
title: "Adaptive Ops — Teaching Agent Topology to Respond to Reality"
date: 2026-06-29
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ops]
tags: [adaptive-ops, desired-state, RAS, fsitrading, soc]
---

## Adaptive Ops — Teaching Agent Topology to Respond to Reality

The four casehub-ops domain modules have been complete for a while now — infra, deployment, compliance, IoT. Each proves the desiredstate SPIs work against a real domain. But they share a limitation: the desired state is static. Declare your topology in YAML, the reconciliation loop provisions it, drift detection catches divergence. Solid. Also insufficient.

A static topology declaration doesn't help when a strategy agent dies at 3 AM and nobody's watching. It doesn't scale risk monitoring when volatility spikes. It doesn't tighten trust policies during an active breach. The topology needs to respond to conditions — not just drift from a fixed declaration.

I started by asking what use case would actually demonstrate this. The existing demo issues (#15-17) would show the SPIs working, but they wouldn't impress anyone. An in-memory reconciliation loop provisioning into stub stores looks like a test with extra steps.

The deep research confirmed something I hadn't expected: financial trading multi-agent systems already use hierarchical orchestration with agent pools, heartbeat monitoring, and structured task dispatch — but none of them have formal desired-state reconciliation, self-healing, or auto-scaling. The AgenticTrading paper from NeurIPS 2025 documents the architecture and the gap in the same paper. Meanwhile, 47.6% of self-adaptive systems research focuses on infrastructure. Zero percent focuses on the application level. The gap casehub-ops sits in is a documented research void.

Two CaseHub applications turned out to be ideal first consumers: casehub-fsitrading (trading automation, overnight incident response) and casehub-soc (cyber incident response, 24/7 operations). Both were already scaffolded with domain background docs. SOC is arguably the stronger case for adaptive ops — the topology change IS the incident response. When a breach is detected, scaling up forensics agents and tightening containment authority isn't a side effect of the response, it's the response itself.

The architecture that emerged has a clean separation: GoalCompiler handles planned adaptation (topology changes driven by conditions), FaultPolicy handles unplanned responses (agent failure, provision errors). Two mechanisms for two genuinely different concerns. I pushed for keeping the adaptive logic in the deployment module initially, but the better call was pushing the `SituationSource` SPI upstream into `casehub-desiredstate-api` — any domain module can then implement situation-aware adaptation, not just deployment.

The signal source is casehub-ras. RAS already detects situations (volatility spikes, active breaches) through Ganglion strategies. The missing piece: RAS situations currently disappear from the store when they trigger a case. For topology adaptation, they need to persist as long as the condition is active. That's now tracked as ras#20.

The YAML format gained an `adaptations:` section with three action types — scale (confidence-proportional instance count), add (inline node specs), and update (Jackson tree-merge for partial field overrides). The adversarial design review raised 12 issues across 4 rounds. The most useful additions were hysteresis bands (preventing churn when confidence oscillates near a threshold), cooldown windows between state changes, and the normalised scaling formula that maps the confidence range above the activation threshold to the full [min, max] instance range.

One surprise during implementation: we committed three desiredstate changes to the wrong branch. The subagent was dispatched to work in casehub-desiredstate, but that repo's working tree was on `issue-47-cte-pending-approval` from a paused session. The commits looked correct — right files, right tests passing, right structure. The branch was wrong. No error, no warning. Cherry-picked to main, reset the branch, but it's the kind of silent failure that could have been much worse if the other session had resumed and pushed.

The implementation itself landed cleanly: `SituationSource` SPI and `ActiveSituation` in the desiredstate API, `requestReconciliation()` on the ReconciliationLoop, the full adaptation rule stack in casehub-ops (YAML types, scale/add/update actions, AdaptiveTopologyManager with per-tenant state, hysteresis, cooldown, periodic re-poll), and an integration test proving the pipeline from YAML through graph mutation.

The thing I keep coming back to: this isn't just infrastructure management at a higher level of abstraction. It's a pattern where the system's desired state is itself a function of the system's observed state — goals that change in response to reality. RAS detects a situation, the GoalCompiler recompiles, the planner transitions, and the topology adapts. The reconciliation loop doesn't know or care that the desired state changed — it just sees a new graph and plans accordingly.
