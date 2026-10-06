# Handover — issue-118-extract-ops-service-module

**Branch:** `issue-118-extract-ops-service-module`
**Issue:** casehubio/casehub-ops#118
**Status:** Spec complete, needs writing-plans → implementation

## What happened

Delivered #116 (PoolNodeSpec — 7th deployment node type). Full brainstorm → implement → work-end cycle. 226 deployment tests green, merged to main, pushed to both remotes.

Architecture discussion about deployment profiles, bootstrap model, and qhorus mesh deployment led to three new issues: #117 (deployment profiles), #118 (extract ops/service), #119 (GraphQL generation). Updated #118 scope with deployment profile context.

Brainstormed and wrote spec for #118: extract all Java source from ops/app into ops/service library module. Package rename `io.casehub.ops.app` → `io.casehub.ops.service`. ops/app becomes a thin Quarkus shell.

## Decisions

- Everything moves to ops/service (APIs, entities, services, K8s, SPI stubs, case descriptors, migrations)
- ops/app retains only pom.xml + application.properties
- Flyway migrations follow entities into ops/service

## Next action

Run `work continue` → invoke writing-plans on the spec, then execute the extraction.

## Cross-repo blockers

- scaffold#53 blocked by this issue (needs ops/service to embed ops)

## References

| Artifact | Path |
|----------|------|
| Spec | `wksp/specs/issue-118-extract-ops-service-module/2026-10-06-extract-ops-service-design.md` |
| Decisions | `wksp/specs/issue-118-extract-ops-service-module/decisions.md` |
| Journal | `wksp/JOURNAL.md` |
