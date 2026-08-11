---
layout: post
title: "ARC42STORIES.MD for casehub-ops — from zero to architecture record"
date: 2026-06-23
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ops]
tags: [documentation, arc42stories, architecture]
---

casehub-ops had four domain modules, four design specs, five blog entries, and no architecture documentation. The specs describe each domain in isolation. The blog entries capture session narratives. Neither answers the question a new session needs answered: what is the system, how are its parts connected, and what patterns does each domain use?

I'd been putting off the arc42stories migration because the repo felt different from the foundation and application tiers where we'd done it before. casehub-ops is integration tier — it doesn't own a runtime (that's casehub-desiredstate), doesn't serve REST endpoints, doesn't run standalone. It implements SPIs. The foundation tier preamble template almost fits, but the layer taxonomy is wrong — integration modules define layers by domain, not by architectural concern.

The migration template from casehubio/work#246 gave us the structure. We adapted it: foundation tier preamble with integration tier framing, layers defined per domain module (L1 shared API, L2–L5 one per domain), chapters mapped 1:1 to the four epics.

The interesting structural decision was in the layer entries. Three domains, three different type strategies — and each one is the right choice for its domain:

- **Infra** uses a sealed `InfraNodeSpec` hierarchy because resource types differ structurally (K8s namespace vs database cluster vs Terraform workspace). Plus a composite wrapper (`InfraDesiredNodeSpec`) to carry backend routing metadata.
- **Deployment** uses a sealed `DeploymentNodeSpec` that extends `NodeSpec` directly — no wrapper needed because there's no backend routing. Four types, exhaustive switch dispatch.
- **Compliance** uses a single `ComplianceControlSpec` record with a `controlType` discriminator. Controls are structurally identical — they differ in configuration, not shape. A sealed hierarchy would create six nearly-identical records.

Documenting all three in the same L1 layer entry makes the pattern selection visible. A developer choosing a type strategy for a new domain reads one section and sees when each approach fits.

The post-generation quality gate from the garden (GE-20260601-85afd0) caught nothing this time — all 33 Key files class names verified, both §12 issue references still OPEN. But the gate is worth running. The devtown migration found 10+ errors with it.

The ARC42STORIES.MD is 732 lines. §9 (Journeys and Chapters) carries the bulk — four chapter entries and four layer entries with key files, wiring, gotchas, and pattern-to-replicate sections. §8 has three anti-patterns: the CDI ambiguity from multiple domain modules, the TransitionPlanner DRIFTED gap, and the eidos binary incompatibility. All three have bitten us in real sessions.

This is the first integration-tier ARC42STORIES.MD in the CaseHub ecosystem. The foundation tier reference is casehub-work. The application tier reference is devtown. Now there's a reference for the integration tier too.
