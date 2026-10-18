# Handover — Claudony desiredstate domain landed

**Branch:** `main` (clean — `issue-140-claudony-desiredstate-spi` landed as `abcefb8`)
**Closed this session:** #140 (claudony desiredstate domain — full SPI Quad)

## What was done

Claudony became a standalone desiredstate domain with full SPI Quad: GoalCompiler, ActualStateAdapter, NodeProvisioner, FaultPolicy, EventSource, plus GoalLoader and NodeSpecFactoryProvider. Two node types (`claudony_app`, `claudony_pool`) as a sealed hierarchy in ops-api. Coexists with `ClaudonyDeploymentTarget` (ADR-0002) — different concern, different callers.

Key design decisions:
- **D0:** Claudony as separate domain (ADR-0003 complements ADR-0002)
- **D1:** Sealed hierarchy for app/pool (different fields, different provisioning)
- **D4:** Pools provisioned via read-modify-write on parent app spec (bridge until claudony adds pool CRUD)

ADR-0003 written. ARC42STORIES, consumer guide, CLAUDE.md updated. 73 tests in claudony module.

## Issues filed

- claudony#290 — standalone pool CRUD endpoints
- claudony#291 — ETag/version support on app specs
- ops#181 — backfill `@DesiredStateQualifier` on existing GoalCompilers
- ops#182 — promote SpecHashStore to shared utility

## Forage candidates (garden was down)

- `CompilationResult` has no `graphs()` — must cast to `SingleGraph` and call `graph()`
- Context constructors (`ProvisionContext`, `SuspendContext`, etc.) require non-null graph
- `casehub-desiredstate-annotations` is not transitive through `desiredstate-api`
- `CrossDomainCompositionEngine` sorts domains with no `requires` before those that have them

## Next candidates

- **#181** (XS/Low) — backfill `@DesiredStateQualifier` on existing GoalCompilers
- **#182** (S/Low) — promote SpecHashStore to shared utility

## References

- Design spec: `wksp specs/issue-140-claudony-nodeprovisioner/2026-10-18-claudony-desiredstate-spi-design.md`
- Decisions: `wksp specs/issue-140-claudony-nodeprovisioner/decisions.md`
- Plan: `wksp plans/2026-10-18-claudony-desiredstate-spi.md`
- ADR: `docs/adr/0003-claudony-desiredstate-domain.md`
- Diary: `wksp blog/2026-10-18-mdp01-two-paths-to-the-same-fleet.md`
