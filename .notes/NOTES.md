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
- Goal: find a known challenge that exposes Terraform's declarative limits and demonstrate CaseHub's improved declarativeness
- Could come from fsitrading naturally, or from a known internet challenge/competition
- No selection made yet — pick one to implement as a reference demo
- Top 5 candidates researched:

**1. Google Online Boutique multi-region (best candidate)**
11 polyglot microservices with inter-service gRPC dependencies, multi-region across 3 clusters. Terraform can provision it but can't express "deploy redis first, wait for health, then deploy cartservice" as lifecycle phases — only static `depends_on`. Can't adapt when a region degrades. CaseHub's lifecycle phases, forEach (stamp per region), invariants (every region must have redis), and continuous reconciliation handle this naturally. 17k+ GitHub stars — most recognisable.

**2. MLOps model pipeline — canary + rollback**
Deploy model → shadow-test on live traffic → canary 5% → monitor accuracy/drift (not just HTTP 200) → human approval gate before full promotion → automated rollback if metrics degrade. Terraform has zero concept of progressive rollout or quality-gated promotion. CaseHub's FaultPolicy handles metric-based rollback, HumanNodeHandler gates promotion. MLOps is $15B+ market.

**3. Cloud Resume Challenge multi-cloud**
Canonical entry-level DevOps challenge (12k+ Discord community). Simple but crosses CDN, DNS, serverless, database, CI/CD. Doing it across AWS + GCP + Cloudflare simultaneously — with reconciliation detecting when a free-tier resource disappears — is something no existing tool does declaratively. Most accessible demo for developers encountering CaseHub.

**4. Multi-region CockroachDB + app stack**
CockroachDB across 3 regions with automatic failover, app tier following database topology, DNS weighted routing shifting with health. Ordering is brutal: cluster bootstrap → node join → schema migration → app deploy → DNS cutover, each requiring the previous to be healthy, not just "created." Terraform's `depends_on` can't express "healthy." CaseHub's ActualStateAdapter verifies health, lifecycle phases enforce ordering.

**5. K8s The Hard Way multi-cloud**
Kelsey Hightower's manual K8s bootstrapping (40k+ GitHub stars). Extended to provision across AWS + GCP + bare metal — certificate rotation, etcd backup reconciliation, human approval for control plane upgrades. Hits every Terraform limitation: no continuous reconciliation, no approval gates, no cross-provider awareness, no adaptive response to node failure.

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
