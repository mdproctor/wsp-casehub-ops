# Handover — Three housekeeping issues landed

**Branch:** `main` (clean)
**Landed this session:** #181, #182, #176

## What was done

Three issues cleared in one session — all mechanical, no design decisions:

1. **#181** `@DesiredStateQualifier` backfilled on DeploymentGoalCompiler, ComplianceGoalCompiler, InfraGoalCompiler, ContainerGoalCompiler. Added `casehub-desiredstate-annotations` dependency to all four modules.
2. **#182** SpecHashStore deduplicated — three identical copies (deployment, compliance, claudony) consolidated into `io.casehub.ops.api.SpecHashStore`. Single CDI bean, no collision via NodeId scoping.
3. **#176** Dual-track tutorial structure created — UC01-UC08 use case track alongside L01-L08 topology track. Bidirectional cross-references, INDEX.md for both tracks, path conventions documented.

## Immediate Next Step

#177 (First use case level — UC04 or UC05) is now unblocked by #176. Also unblocked: #171 (audit assertion test infrastructure).

## References

- Use case track: `topology-tests/src/test/resources/topologies/use-cases/INDEX.md`
- Topology track: `topology-tests/src/test/resources/topologies/deployment/INDEX.md`
