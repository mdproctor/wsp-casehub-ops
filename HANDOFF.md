# Handoff — casehub-ops

## Last Session

Research and design session for canonical deployment topologies and infrastructure operations problem taxonomy. Three major outputs:

1. **Topology research** (`docs/research/2026-08-29-canonical-deployment-topologies.md`) — 5 app architectures x 4 infra topologies matrix. Key discovery: the desiredstate YAML frontend already has modules, forEach, invariants, rules, lifecycle phases — no new compiler needed. Architectural breakthrough: CaseHub's own stack (TransitionPlanner + GOAP + Engine Cases + Service Lifecycle) maps exactly to deployment management at different timescales.

2. **Topology design spec** (`docs/specs/2026-08-29-canonical-deployment-topologies-design.md`) — went through standard 4-dimension design review ($68, 71 issues, 0 unresolved). Spec substantially hardened: NodeSpecFactory SPI, cross-repo coordination, null-coalescing, dynamic handledTypes(), provisioning semantics. Implementation plan written (6 batches, 14 tasks).

3. **Infrastructure operations problem taxonomy** (`docs/research/2026-08-29-infrastructure-operations-problem-taxonomy.md`) — 10 problem domains, 3 techniques (declarative convergence, staleness-based re-execution, GOAP planning), full coverage of the infrastructure lifecycle. Adversarial stress test against top 43 Ansible modules: 85% coverage, 5 gaps identified and addressed. Library landscape mapped (Quarkus-first). CaseHub component utilisation audit: currently ~40%, opportunities identified for blocks, ganglion, eidos, ledger across all domains.

Key decisions:
- YAML is the primary interface; Java is the escape hatch
- Java records are canonical; YAML schema + TS types are generated
- Jinjava (HubSpot, Apache 2.0) solves the templating gap
- quarkus-vault for Vault integration (not standalone driver)
- Existing SecretManager/CredentialResolver SPIs in platform-api — don't rebuild
- ~3,600 lines genuinely new code across all 10 domains

## Immediate Next Step

**ARC42STORIES audit + epic planning.** Two goals:

### Goal 1: Audit ARC42STORIES.MD for Accuracy

Thorough section-by-section audit against the actual codebase using IntelliJ:
- Every chapter (C1-C5): are the descriptions accurate? Do the key files still exist at those paths? Have APIs changed?
- Every layer (L1-L6): do the patterns and anchors match current code?
- Section §8 (Crosscutting): is the SPI Quad pattern documentation current?
- Section §12 (Risks/Debt): are the listed debts still open? Have new ones appeared?
- The design review found mischaracterisations (provisioner dispatch, type hierarchies) — check if ARC42STORIES has similar staleness.

### Goal 2: CaseHub Component Utilisation Audit

For every existing chapter AND every proposed new chapter, verify:
- Are we using every CaseHub component we could be? (Blocks, Ganglion, Eidos, Qhorus, Ledger, Neocortex)
- Is every operational concept expressible as a YAML graph node for the diagramming tools?
- See §11 of the infrastructure taxonomy research doc for the full component mapping.

### Goal 3: Plan New Journeys + Chapters as GitHub Epics

Map the 10 domains to ARC42STORIES chapters and GitHub epics:

| Journey | Chapters | Domains |
|---------|----------|---------|
| Infrastructure maturity | C6: Canonical Topologies → C7: Host Configuration → C8: Data Management | D1, D2, D10 |
| Operational intelligence | C9: Secret & Identity → C10: Periodic Operations → C11: Observability | D7, D4, D8 |
| Lifecycle orchestration | C12: GOAP Actions → C13: Environment Lifecycle → C14: Governance Extensions | D5, D9, D6 |

Dependency graph: D1 → D2 → D7 → D4 → D10 → D8 → D3 → D5 → D9 → D6

Each chapter's epic should include a CaseHub component mapping (which engine capabilities does it exercise).

## References

- Research (topology): `docs/research/2026-08-29-canonical-deployment-topologies.md`
- Research (taxonomy): `docs/research/2026-08-29-infrastructure-operations-problem-taxonomy.md`
- Design spec: `docs/specs/2026-08-29-canonical-deployment-topologies-design.md`
- Implementation plan: workspace `plans/2026-08-29-canonical-deployment-topologies.md`
- ARC42STORIES: `ARC42STORIES.MD` (project root)
