## D1: Which use case level to build first

**Choice:** UC04 Multi-Specialist Desk (maps to L04 Fleet Scaling)
**Alternatives:**
- UC05 Policy Rollout — stronger operational story, but depends on UC04 existing first; the interesting part is a diff between two states, not a single YAML
- Both UC04 and UC05 — more complete, but over-scopes the issue; UC05 follows naturally as a sequel
**Rationale:** UC04 exercises the richest YAML features (forEach, data blocks, per-instance config) and establishes the most template surface area. Its narrative doesn't depend on understanding a prior state. UC05's story only makes sense after UC04 exists.
**Trade-offs:** UC05 (the drift/reconciliation story) is deferred — but it's a natural follow-up that reuses UC04's fleet definition.
**Sources:** topology-tests/.../use-cases/INDEX.md, topology-tests/.../deployment/04-llm-fleet/04-llm-fleet.yaml, casehubio/casehub-ops#177
**Exploration:** quick
**Status:** captured

## D2: Domain scenario for UC04

**Choice:** Customer support desk — billing specialist, technical support, returns agent
**Alternatives:**
- Sales team (enterprise/SMB/partner) — less universally relatable
- Medical triage (GP/specialist/emergency) — niche audience
**Rationale:** Customer support is the most universally understood multi-agent scenario. Three distinct specialties map 1:1 to L04's three fleet members. Data block uses support-domain columns (specialty, greeting, escalation rules, knowledge base).
**Trade-offs:** None significant — customer support is broadly applicable.
**Sources:** UC04-multi-specialist-desk/README.md (existing stub)
**Exploration:** quick
**Status:** captured

## D3: Use case README structure — lean on topology companion

**Choice:** UC README links to topology for mechanism details; focuses purely on business scenario, domain data, and assertions
**Alternatives:**
- Fully self-contained — repeats topology walkthrough content, higher maintenance burden
- Hybrid — self-contained for scenario but links for internals
**Rationale:** Avoids content duplication between tracks. The topology track teaches how, the use case track shows why. Same information shouldn't live in two places.
**Trade-offs:** Reader must open a second file for mechanism details. Acceptable since the cross-reference is explicit.
**Sources:** use-cases/INDEX.md ("Same YAML schema, different narrative"), casehubio/casehub-ops#177
**Exploration:** quick
**Status:** captured
