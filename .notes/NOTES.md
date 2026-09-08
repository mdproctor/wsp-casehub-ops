# Notes — casehub-ops

## 2026-09-08

### Slot Housekeeping
- Archive slot 170 — superseded by 172, still marked active in worklog DB
- Archive slot 172 — work completed (all 10 epic #74 issues delivered), user confirmed should be landed and archived
- Workflow improvement: work-end should warn before archiving a slot when `.plan` has unchecked items (slot 165 post-mortem — user ran work-end on planning branch, whole slot archived, implementation queue lost)

### Uncommitted Changes
- CLAUDE.md in ops project has symlink path normalization (absolute paths → `proj/`/`wksp/`) — needs committing

### YAML-First Plugin Architecture
- Three-layer design: YAML plugins → YAML primitives → Java SPIs
- 5 core primitives first: `rest-call`, `json-extract`, `compare-state`, `paginate`, `poll-until`
- Java records are canonical, JSON Schema generated per NodeSpec type, consumed by IDE plugins (IntelliJ + VS Code)
- Type safety throughout — every field that can be referenced elsewhere must have a declared type, not just a string
- Schema generation should come early, before plugin count grows
- Cloudflare DNS plugin sketched as reference example
- Needs issue filed

### Multi-Cloud Free Tier Aggregation
- Novel concept: build a platform by aggregating free tiers across vendors (Oracle Cloud 4 OCPU ARM, Cloudflare Workers, Supabase, Fly.io, AWS Lambda, GCP Cloud Run)
- Desiredstate runtime reconciles across all vendors — drift detected, FaultPolicy re-routes
- YAML declares topology; runtime makes it true across vendors
- Needs issue filed

### CrossplaneNodeProvisioner
- Issue #86 filed — delegate provisioning to Crossplane's ~200 providers, CaseHub handles governance
- Crossplane does multi-cloud provisioning; CaseHub adds governance + type-safe declarative (the gap)

### fsitrading — First App Adoption
- fsitrading ranked #1 candidate for first YAML desiredstate adoption (ops issue #25)
- soc #2 (#26), devtown #3
- Sequence: possibly #83 (casehub-application NodeSpec) → #25 (fsitrading) → #26 (soc)
- Check whether #25 actually requires #83 or can use existing topology primitives directly
- Work driven from fsitrading repo, not ops

### DevOps Challenge Scenarios (for demonstrating CaseHub vs Terraform/Ansible)
- Goal: find a known challenge that exposes Terraform's declarative limits
- Top 5 candidates researched:
  1. Google Online Boutique multi-region — 11 microservices, lifecycle ordering, forEach per region (17k GitHub stars)
  2. MLOps model pipeline — canary + human-gated promotion + metric-based rollback
  3. Cloud Resume Challenge multi-cloud — most accessible, 12k+ Discord community
  4. Multi-region CockroachDB — health-aware ordering, quorum reconciliation
  5. K8s The Hard Way multi-cloud — 40k stars, cert rotation, human-gated upgrades
- No selection made yet — pick one to implement as a reference demo

### Competitive Positioning
- Crossplane solves multi-cloud provisioning (CNCF, production-grade)
- KubeVela/OAM adds app delivery with CUE templates and manual approval
- Pulumi has type safety but is imperative
- Nobody has type-safe declarative — CaseHub's differentiator
- CaseHub's gap vs Crossplane: governance (FaultPolicy, HumanNodeHandler, trust-weighted execution, Ledger audit)

### IoT as Ops Domain
- IoT exercises HumanNodeHandler (physical device replacement = human work)
- Mixed model: automated for firmware OTA, human-gated for physical replacement
- Compelling demo for human-in-the-loop reconciliation that no competitor matches
- Not prioritised over fsitrading — just broadening the picture
