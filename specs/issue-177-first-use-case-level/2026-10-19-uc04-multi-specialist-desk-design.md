# UC04 Multi-Specialist Desk — Design Spec

**Issue:** casehubio/casehub-ops#177
**Topology companion:** L04 Fleet Scaling (`deployment/04-llm-fleet/`)
**Branch:** issue-177-first-use-case-level

## Goal

Build the first use case level as the template for the use case track. UC04 Multi-Specialist
Desk maps to L04 Fleet Scaling — same platform capability (forEach expansion), different
narrative (customer support fleet with distinct specialists).

This level establishes the pattern that UC01–UC08 will follow: YAML-only, domain-specific
data, README as scenario narrative, cross-reference to topology companion.

## Deliverables

1. **`UC04-multi-specialist-desk.yaml`** — use case topology YAML
2. **`README.md`** — expanded scenario narrative (replaces the existing stub)
3. No Java test classes — use case YAMLs are validated by the existing compilation infrastructure

## YAML Design

### Data Block

Replace L04's abstract roles with a customer support scenario:

```
name:STRING,specialty:STRING,greeting:STRING,escalation_threshold:STRING,knowledge_base:STRING
billing,Billing & Payments,How can I help with your billing?,high,billing-kb
technical,Technical Support,What technical issue are you experiencing?,medium,tech-kb
returns,Returns & Refunds,I can help process your return.,low,returns-kb
```

Three specialists with domain-specific attributes:
- **billing** — handles payment issues, high escalation threshold (most queries resolved in-agent)
- **technical** — handles product/technical issues, medium threshold (some need engineering)
- **returns** — handles return/refund processing, low threshold (often needs supervisor approval)

### Node Structure

Same infrastructure nodes as L04 (manifest, eidos, engine). The `support-agent` node
replaces `my-llm`, using forEach over the support team data:

```yaml
support-agent:
  type: casehub_application
  dependsOn: [engine]
  forEach:
    as: agent
    in: support-team
  spec:
    applicationId: support-${each.agent.name}
    image: casehubio/support-agent:latest
    deploymentTarget: claudony
    replicas: 1
    config:
      agent.id: support-${each.agent.name}
      agent.name: ${each.agent.name} Specialist
      agent.specialty: ${each.agent.specialty}
      agent.greeting: ${each.agent.greeting}
      agent.escalation-threshold: ${each.agent.escalation_threshold}
      agent.knowledge-base: ${each.agent.knowledge_base}
      agent.identity-source: eidos
```

### Key Differences from L04

| Aspect | L04 (topology) | UC04 (use case) |
|--------|----------------|-----------------|
| desiredState name | `llm-fleet` | `multi-specialist-desk` |
| Data block name | `llm-fleet` | `support-team` |
| Data columns | role, capability, archetype, adjective, domain | specialty, greeting, escalation_threshold, knowledge_base |
| Node name | `my-llm` | `support-agent` |
| Image | `casehubio/my-llm-app:latest` | `casehubio/support-agent:latest` |
| Config keys | `llm.*` | `agent.*` |
| Expanded IDs | my-llm-analyst, my-llm-coder, my-llm-assistant | support-billing, support-technical, support-returns |

Same platform mechanism, different domain vocabulary.

## README Structure (Template Pattern)

The README establishes the structure all UC levels will follow:

```markdown
# UC04: Multi-Specialist Desk

**Topology companion:** [L04 Fleet Scaling](link)
**Prerequisites:** UC03

## Scenario
[2-3 paragraph business narrative — who, what problem, why this matters]

## The Fleet
[Table or description of the domain-specific agents and their attributes]

## How It Works
[3-5 step walkthrough linking to topology for mechanism details:
 "The platform uses forEach expansion (see L04) to create one instance per specialist"]

## What To Look For
[Verification checklist — domain-specific assertions:
 "billing agent's greeting is 'How can I help with your billing?'"]

## Why This Matters
[1-2 paragraphs connecting the pattern to real-world value]
```

Sections stay the same across UC levels; content changes per scenario. "How It Works"
always links to the topology companion for mechanism details, keeping the UC README
focused on the business story.

## Validation

The YAML compiles and validates through the existing `DeploymentCompilationTest` infrastructure.
No new Java test classes needed — the compilation tests load all YAML files under `topologies/`
and verify they produce valid `DesiredStateGraph` instances with correct node counts,
dependencies, and config values.

Verify:
- YAML compiles without errors
- 6 nodes produced (3 infrastructure + 3 support agents)
- Expanded node IDs: `support-agent.billing`, `support-agent.technical`, `support-agent.returns`
- Each agent's config reflects its data row

## References

- `topology-tests/.../deployment/04-llm-fleet/04-llm-fleet.yaml` — topology companion YAML
- `topology-tests/.../deployment/04-llm-fleet/README.md` — topology companion README
- `topology-tests/.../use-cases/INDEX.md` — dual-track structure definition
- `topology-tests/.../use-cases/UC04-multi-specialist-desk/README.md` — existing stub
- `DeploymentCompilationTest.java` — compilation test infrastructure
- `casehubio/casehub-ops#177` — parent issue
- `casehubio/casehub-ops#165` — parent epic
