# UC04 Multi-Specialist Desk Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #177 — First use case level — UC04 Multi-specialist desk or UC05 Policy rollout
**Issue group:** #177

**Goal:** Build the first use case level (UC04 Multi-Specialist Desk) as the template for the use case track — YAML topology + README narrative.

**Architecture:** UC04 mirrors L04 Fleet Scaling's structure with domain-specific customer support data. Three deliverables: a use case YAML file, an expanded README, and compilation test methods added to the existing `DeploymentCompilationTest`. The YAML uses the same `forEach` expansion mechanism as L04 but with support-domain vocabulary (billing, technical, returns specialists).

**Tech Stack:** YAML (topology definition), Java (compilation test methods), Markdown (README)

## Global Constraints

- Use case YAMLs live under `topology-tests/src/test/resources/topologies/use-cases/`
- YAML filename matches directory: `UC04-multi-specialist-desk.yaml`
- Node type is `casehub_application` — same as all deployment topology levels
- Type registry uses `DEPLOYMENT_REGISTRY` (InfraNodeSpec + DeploymentNodeSpec)
- README leans on topology companion for mechanism details — no duplication

---

## Batch 1: UC04 Multi-Specialist Desk

### Task 1: Write compilation tests for UC04 YAML

**Files:**
- Modify: `topology-tests/src/test/java/io/casehub/ops/topology/compilation/DeploymentCompilationTest.java:209-218`

**Interfaces:**
- Consumes: `TopologyTestBase.compileSingleGraph(String, Map)`, `extractAppSpec(DesiredStateGraph, String)` — both already exist in the test class
- Produces: Four test methods that will fail until the YAML is created in Task 2

- [ ] **Step 1: Add UC04 compilation test methods**

Add a new section after the Level 04 tests (before `extractAppSpec`), following the same pattern as `llmFleet_*` tests. Insert these methods using `ide_insert_member` before the `extractAppSpec` method:

```java
// ── UC04: Multi-Specialist Desk ─────────────────────────────────────

@Test
void uc04MultiSpecialistDesk_hasSixNodes() throws IOException {
    DesiredStateGraph graph = compileSingleGraph(
            "topologies/use-cases/UC04-multi-specialist-desk/UC04-multi-specialist-desk.yaml", DEPLOYMENT_REGISTRY);
    assertThat(graph.nodes()).hasSize(6);
}

@Test
void uc04MultiSpecialistDesk_threeExpandedSupportAgents() throws IOException {
    DesiredStateGraph graph = compileSingleGraph(
            "topologies/use-cases/UC04-multi-specialist-desk/UC04-multi-specialist-desk.yaml", DEPLOYMENT_REGISTRY);
    assertThat(graph.nodes().keySet().stream()
                    .map(NodeId::value)
                    .filter(id -> id.startsWith("support-agent."))
                    .toList())
            .containsExactlyInAnyOrder("support-agent.billing", "support-agent.technical", "support-agent.returns");
}

@Test
void uc04MultiSpecialistDesk_allSupportAgentsDependOnEngine() throws IOException {
    DesiredStateGraph graph = compileSingleGraph(
            "topologies/use-cases/UC04-multi-specialist-desk/UC04-multi-specialist-desk.yaml", DEPLOYMENT_REGISTRY);
    for (String agentId : List.of("support-agent.billing", "support-agent.technical", "support-agent.returns")) {
        assertThat(graph.dependenciesOf(NodeId.of(agentId)))
                .as("dependencies of %s", agentId)
                .contains(NodeId.of("engine"));
    }
}

@Test
void uc04MultiSpecialistDesk_configValuesFlowPerAgent() throws IOException {
    DesiredStateGraph graph = compileSingleGraph(
            "topologies/use-cases/UC04-multi-specialist-desk/UC04-multi-specialist-desk.yaml", DEPLOYMENT_REGISTRY);

    var billing = extractAppSpec(graph, "support-agent.billing");
    assertThat(billing.config()).containsEntry("agent.specialty", "Billing & Payments");
    assertThat(billing.config()).containsEntry("agent.escalation-threshold", "high");

    var technical = extractAppSpec(graph, "support-agent.technical");
    assertThat(technical.config()).containsEntry("agent.specialty", "Technical Support");
    assertThat(technical.config()).containsEntry("agent.escalation-threshold", "medium");

    var returns = extractAppSpec(graph, "support-agent.returns");
    assertThat(returns.config()).containsEntry("agent.specialty", "Returns & Refunds");
    assertThat(returns.config()).containsEntry("agent.escalation-threshold", "low");
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn --batch-mode -o test -pl topology-tests -Dtest=DeploymentCompilationTest#uc04MultiSpecialistDesk* -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — YAML file not found

- [ ] **Step 3: Commit failing tests**

```bash
git add topology-tests/src/test/java/io/casehub/ops/topology/compilation/DeploymentCompilationTest.java
git commit -m "test(#177): add compilation tests for UC04 Multi-Specialist Desk

TDD red — YAML file not yet created.

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 2: Create UC04 YAML topology file

**Files:**
- Create: `topology-tests/src/test/resources/topologies/use-cases/UC04-multi-specialist-desk/UC04-multi-specialist-desk.yaml`

**Interfaces:**
- Consumes: nothing
- Produces: valid desiredstate YAML that compiles to 6 nodes (3 infrastructure + 3 support agents via forEach expansion)

- [ ] **Step 1: Create the UC04 YAML file**

Write the file at `topology-tests/src/test/resources/topologies/use-cases/UC04-multi-specialist-desk/UC04-multi-specialist-desk.yaml`:

```yaml
# UC04: Multi-Specialist Desk
# Use case companion for Level 04: Fleet Scaling
# Prerequisites: UC03
# See README.md for scenario walkthrough

desiredState:
  namespace: casehub
  name: multi-specialist-desk

data:
  support-team:
    inline: |
      name:STRING,specialty:STRING,greeting:STRING,escalation_threshold:STRING,knowledge_base:STRING
      billing,Billing & Payments,How can I help with your billing?,high,billing-kb
      technical,Technical Support,What technical issue are you experiencing?,medium,tech-kb
      returns,Returns & Refunds,I can help process your return.,low,returns-kb

nodes:
  manifest:
    type: casehub_application
    spec:
      applicationId: manifest
      image: casehubio/manifest:latest
      deploymentTarget: claudony
      replicas: 1

  eidos:
    type: casehub_application
    dependsOn: [manifest]
    spec:
      applicationId: eidos
      image: casehubio/eidos:latest
      deploymentTarget: claudony
      replicas: 1

  engine:
    type: casehub_application
    dependsOn: [eidos]
    spec:
      applicationId: engine
      image: casehubio/engine:latest
      deploymentTarget: claudony
      replicas: 1

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

- [ ] **Step 2: Run compilation tests to verify they pass**

Run: `mvn --batch-mode -o test -pl topology-tests -Dtest=DeploymentCompilationTest#uc04MultiSpecialistDesk*`
Expected: 4 tests PASS — 6 nodes compiled, 3 support agents expanded with correct IDs and config

- [ ] **Step 3: Run full test suite to check for regressions**

Run: `mvn --batch-mode -o test -pl topology-tests`
Expected: all existing tests still pass

- [ ] **Step 4: Commit**

```bash
git add topology-tests/src/test/resources/topologies/use-cases/UC04-multi-specialist-desk/UC04-multi-specialist-desk.yaml
git commit -m "feat(#177): UC04 Multi-Specialist Desk topology YAML

Customer support fleet with billing, technical, and returns specialists.
Uses forEach expansion over support-team data block — same mechanism as
L04 Fleet Scaling with domain-specific vocabulary.

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 3: Write UC04 README narrative

**Files:**
- Modify: `topology-tests/src/test/resources/topologies/use-cases/UC04-multi-specialist-desk/README.md`

**Interfaces:**
- Consumes: nothing
- Produces: scenario narrative README following the template pattern for all UC levels

- [ ] **Step 1: Replace README stub with full narrative**

Overwrite the existing stub at `topology-tests/src/test/resources/topologies/use-cases/UC04-multi-specialist-desk/README.md`:

```markdown
# UC04: Multi-Specialist Desk

**Topology companion:** [L04 Fleet Scaling](../../deployment/04-llm-fleet/)
**Prerequisites:** UC03

## Scenario

A growing customer support operation needs specialist agents — not a single
generalist that handles everything poorly, but dedicated specialists who each
excel at their domain. Billing queries go to an agent that understands payment
systems, invoicing rules, and refund policies. Technical issues go to an agent
trained on the product's architecture, troubleshooting guides, and known issues.
Returns go to an agent that knows the returns workflow, shipping logistics, and
exception policies.

Each specialist has its own greeting, escalation threshold, and knowledge base.
The billing specialist resolves most queries directly (high threshold) because
billing answers are usually definitive. The technical specialist escalates more
frequently (medium threshold) because some issues need engineering. The returns
specialist escalates most often (low threshold) because refund approvals often
require supervisor sign-off.

The fleet scales by adding specialties — not by cloning a generic agent. When
the business adds a "warranty claims" specialty next quarter, the topology
change is one CSV row.

## The Fleet

| Specialist | Greeting | Escalation | Knowledge Base |
|-----------|----------|------------|----------------|
| Billing | "How can I help with your billing?" | High | billing-kb |
| Technical | "What technical issue are you experiencing?" | Medium | tech-kb |
| Returns | "I can help process your return." | Low | returns-kb |

## How It Works

1. Infrastructure services deploy first — manifest, eidos, engine — in
   dependency order (same as every topology level)
2. The `support-agent` node uses **forEach expansion** over the `support-team`
   data block to create one instance per specialist
   ([see L04 for the mechanism](../../deployment/04-llm-fleet/))
3. Each instance gets a unique `applicationId` (`support-billing`,
   `support-technical`, `support-returns`) and domain-specific config
   resolved from its data row via `${each.agent.*}` variables
4. All three agents depend on `engine` — they deploy in parallel after
   infrastructure is ready
5. The result: 6 nodes total (3 infrastructure + 3 support agents), each
   agent configured for its specialty

## What To Look For

1. **6 total nodes** — 3 infrastructure (manifest, eidos, engine) + 3 support agents
2. **Distinct applicationIds** — `support-billing`, `support-technical`, `support-returns`
3. **Per-agent config** — billing has escalation "high", technical has "medium", returns has "low"
4. **Greeting differentiation** — each agent greets customers with domain-appropriate language
5. **Knowledge base routing** — each agent points to its own KB (billing-kb, tech-kb, returns-kb)

## Why This Matters

Real support operations don't run one agent that does everything. They run
specialists — and those specialists need different configurations, different
knowledge bases, and different escalation rules. The alternative is maintaining
three separate topology definitions that drift apart over time.

forEach expansion turns this into a data problem. The topology defines the
shape once; the data block defines the variants. Adding a fourth specialty
is a CSV row, not a copy-paste of infrastructure config. This is why fleet
scaling matters — not for running N identical copies, but for running N
differentiated instances from a single declaration.
```

- [ ] **Step 2: Commit**

```bash
git add topology-tests/src/test/resources/topologies/use-cases/UC04-multi-specialist-desk/README.md
git commit -m "feat(#177): UC04 Multi-Specialist Desk README narrative

Template for use case track — scenario, fleet table, how-it-works
linked to topology companion, verification checklist, business value.

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

## References

- [2026-10-19-uc04-multi-specialist-desk-design.md] — design spec this plan implements
- [topology-tests/.../deployment/04-llm-fleet/04-llm-fleet.yaml] — topology companion YAML
- [topology-tests/.../compilation/DeploymentCompilationTest.java:153-208] — existing L04 test methods (pattern to follow)
- [topology-tests/.../use-cases/INDEX.md] — dual-track structure definition
- [GitHub #177] — focal issue
- [GitHub #165] — parent epic
