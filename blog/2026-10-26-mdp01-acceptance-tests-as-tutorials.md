---
layout: post
title: "Acceptance Tests as Tutorials — Proving Six Topology Levels Actually Work"
date: 2026-10-26
entry_type: note
subtype: diary
projects: [casehubio/casehub-ops]
tags: [testing, topology, integration, acceptance, tutorial]
series: issue-194-acceptance-test-retrofit
---

# Acceptance Tests as Tutorials — Proving Six Topology Levels Actually Work

The existing integration tests for CaseHub Ops proved that topologies *deploy* — nodes show up, reconciliation completes, CloudEvents emit. What they didn't prove is that the deployed system *works*. A broken accountability ledger would pass. A case management system that never processes a case would pass. The tests verified infrastructure, not capability.

I wanted each topology level to prove its claim end-to-end. Level 02 says it provides accountability? Deploy it, exercise the LLM, and show me the per-node audit trail. Level 08 says it provides fleet observability? Deploy three LLMs, exercise them, inject a fault, and prove the system detects it and recovers.

The pattern is the same across all six: deploy the topology via ReconciliationLoop with audit capture, exercise whatever LLMs the topology provides via FleetLlmExerciser, then assert the level-specific capability. Each test also prints a tutorial-style narrative — headers, checkmarks, what's happening and why — because these tests double as learning content. Someone running the test for the first time should see a story, not just PASS.

The interesting split was between topologies that have exercisable LLMs and those that don't. Levels 02 and 03 use `casehub_application` nodes with LLM configs — FleetLlmExerciser works directly. Levels 08, 10, 12, and 14 use `claudony_app` nodes. Level 08 has three fleet members (analyst, coder, assistant) that can be exercised and differentiated by role. Levels 10 and 14 manage pools — `support-app` has no LLM routing config, so proving the pool works means verifying the pool configuration through the fleet API and testing the scaling mechanics. That's the right test for what pools prove.

Level 12 was the most satisfying to wire up. It exercises the full YAML adaptation lifecycle: inject a RAS situation, have the deployer LLM evaluate it (mocked — this is about the rule mechanics, not the LLM judgment), provision a specialist node via the reconciliation loop, exercise the specialist, then resolve the situation and verify the specialist is deprovisioned. The entire deploy-exercise-resolve arc in one test.

The six tests bring the topology suite from "things deploy" to "things work." Each test now answers the question its topology level asks.
