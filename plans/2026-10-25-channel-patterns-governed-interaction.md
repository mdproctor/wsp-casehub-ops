# Channel Patterns & Governed Interaction Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #187 — feat: channel pattern support in topology YAML — protocols + policyOverrides on channel nodes
**Issue group:** #187, #188

**Goal:** Extend ChannelNodeSpec with protocols, protocolParticipants, and policyOverrides, wire them through provisioning and drift detection, and add a Level 15 integration test demonstrating governed interaction via NEGOTIATION protocol.

**Architecture:** Three fields added to ChannelNodeSpec (api module), ChannelProvisionHandler switches to ChannelCreateRequest.builder() and wires new fields through ChannelOperations (deployment module), ChannelDriftChecker gains list/map comparison for drift detection, and a Level 15 topology YAML + integration test proves end-to-end protocol enforcement (topology-tests module).

**Tech Stack:** Java 21, Quarkus, Jackson YAML, qhorus Channel API, langchain4j-jlama (test-scoped)

## Global Constraints

- All new ChannelNodeSpec fields are optional (nullable) — existing YAML is unaffected
- PolicyOverrides use replace semantics: YAML = desired full set, extras on live channel = drift
- `@JsonIgnoreProperties(ignoreUnknown = true)` already on ChannelNodeSpec handles forward/backward compat
- Tests use `mvn --batch-mode -o test -pl <module>` (offline avoids GitHub Packages metadata hang)
- Every commit references an issue: `Refs #187` or `Refs #188`

---

## Batch 1: ChannelNodeSpec + Provisioner + Drift Detection

### Task 1: Add protocols, protocolParticipants, policyOverrides to ChannelNodeSpec

**Files:**
- Modify: `api/src/main/java/io/casehub/ops/api/deployment/ChannelNodeSpec.java`
- Modify: `api/src/test/java/io/casehub/ops/api/deployment/ChannelNodeSpecTest.java` (if exists, otherwise inline assertions below)

**Interfaces:**
- Consumes: nothing new
- Produces: `ChannelNodeSpec.protocols()` → `List<String>`, `ChannelNodeSpec.protocolParticipants()` → `List<String>`, `ChannelNodeSpec.policyOverrides()` → `Map<String, String>`

- [ ] **Step 1: Write failing test for new fields**

In the deployment test module (`deployment/src/test/java/io/casehub/ops/deployment/handler/ChannelProvisionHandlerTest.java`), add a test that constructs a `ChannelNodeSpec` with protocols, protocolParticipants, and policyOverrides:

```java
@Test
void provisionCreatesChannelWithProtocolsAndPolicyOverrides() {
    var spec = new ChannelNodeSpec(
            "design-review",
            "Design review channel",
            ChannelSemantic.APPEND,
            null, null,       // allowedTypes, deniedTypes
            null, null, null, // allowedWriters, adminInstances, barrierContributors
            null, null,       // rateLimits
            null, null, null, null, // connector binding
            List.of("NEGOTIATION"),                        // protocols
            List.of("coordinator", "reviewer-1"),          // protocolParticipants
            Map.of("negotiation.acceptance", "majority",   // policyOverrides
                   "negotiation.max-rounds", "3")
    );

    assertThat(spec.protocols()).containsExactly("NEGOTIATION");
    assertThat(spec.protocolParticipants()).containsExactly("coordinator", "reviewer-1");
    assertThat(spec.policyOverrides()).containsEntry("negotiation.acceptance", "majority");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode -o test -pl deployment -Dtest="ChannelProvisionHandlerTest#provisionCreatesChannelWithProtocolsAndPolicyOverrides"`
Expected: compilation failure — ChannelNodeSpec constructor doesn't accept 17 args

- [ ] **Step 3: Add three fields to ChannelNodeSpec record**

Use `ide_replace_member` to update the `ChannelNodeSpec` record in `api/src/main/java/io/casehub/ops/api/deployment/ChannelNodeSpec.java`:

```java
package io.casehub.ops.api.deployment;

import java.util.List;
import java.util.Map;
import java.util.Set;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
import io.casehub.desiredstate.api.NodeType;
import io.casehub.qhorus.api.channel.ChannelSemantic;
import io.casehub.qhorus.api.message.MessageType;

@JsonIgnoreProperties(ignoreUnknown = true)
public record ChannelNodeSpec(
        String name,
        String description,
        ChannelSemantic semantic,
        Set<MessageType> allowedTypes,
        Set<MessageType> deniedTypes,
        String allowedWriters,
        String adminInstances,
        String barrierContributors,
        Integer rateLimitPerChannel,
        Integer rateLimitPerInstance,
        String inboundConnectorId,
        String externalKey,
        String outboundConnectorId,
        String outboundDestination,
        List<String> protocols,
        List<String> protocolParticipants,
        Map<String, String> policyOverrides
) implements DeploymentNodeSpec {

    public ChannelNodeSpec {
        if (name == null || name.isBlank()) {
            throw new IllegalArgumentException("channel name is required");
        }
        if (semantic == null) {
            throw new IllegalArgumentException("channel semantic is required");
        }
        allowedTypes = allowedTypes != null ? Set.copyOf(allowedTypes) : null;
        deniedTypes = deniedTypes != null ? Set.copyOf(deniedTypes) : null;
        protocols = protocols != null ? List.copyOf(protocols) : null;
        protocolParticipants = protocolParticipants != null ? List.copyOf(protocolParticipants) : null;
        policyOverrides = policyOverrides != null ? Map.copyOf(policyOverrides) : null;
    }

    @Override
    public String nodeId() {
        return name;
    }

    @Override
    public NodeType nodeType() {
        return NodeType.of("channel");
    }
}
```

- [ ] **Step 4: Fix all existing call sites using the 14-arg constructor**

All existing tests that construct `ChannelNodeSpec` with 14 args need three additional `null` arguments. Use `ide_find_references` on the `ChannelNodeSpec` constructor to find all call sites:

- `ChannelProvisionHandlerTest.java` — each `new ChannelNodeSpec(...)` call gains `, null, null, null` at the end
- `ChannelDriftCheckerTest.java` — same fix
- `StubChannelOperations.java` — no constructor calls (uses Channel.builder)
- Any other test or production references found

- [ ] **Step 5: Run full api + deployment tests**

Run: `mvn --batch-mode -o test -pl api,deployment`
Expected: all tests PASS including the new test

- [ ] **Step 6: Commit**

```bash
git add api/src/main/java/io/casehub/ops/api/deployment/ChannelNodeSpec.java
git add deployment/src/test/
git commit -m "feat(#187): add protocols, protocolParticipants, policyOverrides to ChannelNodeSpec

Add three optional fields to ChannelNodeSpec for channel pattern support:
- protocols (List<String>) — e.g. [NEGOTIATION]
- protocolParticipants (List<String>) — agent instances in protocol
- policyOverrides (Map<String,String>) — protocol config overrides

Refs #187"
```

### Task 2: Wire ChannelProvisionHandler to use new fields

**Files:**
- Modify: `deployment/src/main/java/io/casehub/ops/deployment/handler/ChannelProvisionHandler.java`
- Modify: `deployment/src/test/java/io/casehub/ops/deployment/handler/ChannelProvisionHandlerTest.java`
- Modify: `deployment/src/test/java/io/casehub/ops/deployment/StubChannelOperations.java`

**Interfaces:**
- Consumes: `ChannelNodeSpec.protocols()`, `ChannelNodeSpec.protocolParticipants()`, `ChannelNodeSpec.policyOverrides()` from Task 1
- Produces: `ChannelOperations.setProtocols(UUID, List<String>)`, `ChannelOperations.setProtocolParticipants(UUID, List<String>)`, `ChannelOperations.setPolicyOverrides(UUID, Map<String,String>)` — used by drift tests and production wiring

- [ ] **Step 1: Write failing test for protocols on create**

Add to `ChannelProvisionHandlerTest`:

```java
@Test
void provisionCreatesChannelWithProtocols() {
    var spec = new ChannelNodeSpec(
            "negotiation-ch",
            "Negotiation channel",
            ChannelSemantic.APPEND,
            null, null, null, null, null, null, null,
            null, null, null, null,
            List.of("NEGOTIATION"),
            List.of("agent-a", "agent-b"),
            Map.of("negotiation.acceptance", "majority")
    );

    handler.provision(spec, new ProvisionContext("test-tenancy", emptyGraph));

    Channel created = stub.findByName("negotiation-ch").orElseThrow();
    assertThat(created.protocols()).containsExactly("NEGOTIATION");
    assertThat(created.protocolParticipants()).containsExactly("agent-a", "agent-b");
    assertThat(created.policyOverrides()).containsEntry("negotiation.acceptance", "majority");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode -o test -pl deployment -Dtest="ChannelProvisionHandlerTest#provisionCreatesChannelWithProtocols"`
Expected: FAIL — protocols not passed through, stub doesn't build protocols into Channel

- [ ] **Step 3: Add three methods to ChannelOperations interface**

In `ChannelProvisionHandler.java`, add to the `ChannelOperations` interface:

```java
Channel setProtocols(UUID channelId, List<String> protocols);
Channel setProtocolParticipants(UUID channelId, List<String> protocolParticipants);
Channel setPolicyOverrides(UUID channelId, Map<String, String> policyOverrides);
```

- [ ] **Step 4: Switch creation to builder, wire protocols**

Replace the `new ChannelCreateRequest(...)` call in `provision()` with:

```java
var request = ChannelCreateRequest.builder(spec.name())
        .description(spec.description())
        .semantic(spec.semantic())
        .barrierContributors(toList(spec.barrierContributors()))
        .allowedWriters(toList(spec.allowedWriters()))
        .adminInstances(toList(spec.adminInstances()))
        .rateLimitPerChannel(spec.rateLimitPerChannel())
        .rateLimitPerInstance(spec.rateLimitPerInstance())
        .allowedTypes(spec.allowedTypes())
        .deniedTypes(spec.deniedTypes())
        .protocols(spec.protocols())
        .protocolParticipants(spec.protocolParticipants())
        .inboundConnectorId(spec.inboundConnectorId())
        .externalKey(spec.externalKey())
        .outboundConnectorId(spec.outboundConnectorId())
        .outboundDestination(spec.outboundDestination())
        .build();
```

- [ ] **Step 5: Add policyOverrides to updateMutableFields with replace semantics**

Add to `updateMutableFields()`:

```java
ops.setProtocols(channelId, spec.protocols());
ops.setProtocolParticipants(channelId, spec.protocolParticipants());
if (spec.policyOverrides() != null) {
    ops.setPolicyOverrides(channelId, spec.policyOverrides());
} else {
    ops.setPolicyOverrides(channelId, null);
}
```

- [ ] **Step 6: Update CDI constructor to wire new ChannelService methods**

In the `@Inject` constructor, add the three new delegating methods:

```java
@Override public Channel setProtocols(UUID id, List<String> p) { return channelService.setProtocols(id, p); }
@Override public Channel setProtocolParticipants(UUID id, List<String> p) { return channelService.setProtocolParticipants(id, p); }
@Override public Channel setPolicyOverrides(UUID id, Map<String, String> p) { return channelService.setPolicyOverrides(id, p); }
```

- [ ] **Step 7: Update StubChannelOps in ChannelProvisionHandlerTest**

Add to the inner `StubChannelOps` class:
- `create()` must pass `protocols` and `protocolParticipants` through to the Channel builder
- Implement `setProtocols`, `setProtocolParticipants`, `setPolicyOverrides`

```java
@Override
public Channel create(ChannelCreateRequest req) {
    Channel ch = Channel.builder(req.name())
            .id(UUID.randomUUID())
            .description(req.description())
            .semantic(req.semantic())
            .barrierContributors(req.barrierContributors())
            .allowedWriters(req.allowedWriters())
            .adminInstances(req.adminInstances())
            .rateLimitPerChannel(req.rateLimitPerChannel())
            .rateLimitPerInstance(req.rateLimitPerInstance())
            .allowedTypes(req.allowedTypes())
            .deniedTypes(req.deniedTypes())
            .protocols(req.protocols())
            .protocolParticipants(req.protocolParticipants())
            .build();
    channels.put(ch.name(), ch);
    return ch;
}

@Override
public Channel setProtocols(UUID channelId, List<String> protocols) {
    Channel ch = findById(channelId);
    Channel updated = ch.toBuilder().protocols(protocols).build();
    channels.put(updated.name(), updated);
    return updated;
}

@Override
public Channel setProtocolParticipants(UUID channelId, List<String> protocolParticipants) {
    Channel ch = findById(channelId);
    Channel updated = ch.toBuilder().protocolParticipants(protocolParticipants).build();
    channels.put(updated.name(), updated);
    return updated;
}

@Override
public Channel setPolicyOverrides(UUID channelId, Map<String, String> policyOverrides) {
    Channel ch = findById(channelId);
    Channel updated = ch.toBuilder().policyOverrides(policyOverrides).build();
    channels.put(updated.name(), updated);
    return updated;
}
```

- [ ] **Step 8: Update StubChannelOperations (deployment test)**

Same pattern as Step 7 but in `deployment/src/test/java/io/casehub/ops/deployment/StubChannelOperations.java`. Add `setProtocols`, `setProtocolParticipants`, `setPolicyOverrides`, and wire `protocols`/`protocolParticipants` through in `create()`.

- [ ] **Step 9: Run all deployment tests**

Run: `mvn --batch-mode -o test -pl deployment`
Expected: all tests PASS

- [ ] **Step 10: Commit**

```bash
git add deployment/
git commit -m "feat(#187): wire protocols and policyOverrides through ChannelProvisionHandler

Switch ChannelCreateRequest construction to builder pattern.
Add setProtocols, setProtocolParticipants, setPolicyOverrides to
ChannelOperations interface. PolicyOverrides use replace semantics.

Refs #187"
```

### Task 3: Add drift detection for protocols, protocolParticipants, policyOverrides

**Files:**
- Modify: `deployment/src/main/java/io/casehub/ops/deployment/drift/ChannelDriftChecker.java`
- Modify: `deployment/src/test/java/io/casehub/ops/deployment/drift/ChannelDriftCheckerTest.java`

**Interfaces:**
- Consumes: `ChannelNodeSpec.protocols()`, `ChannelNodeSpec.protocolParticipants()`, `ChannelNodeSpec.policyOverrides()` from Task 1
- Produces: `ChannelDriftChecker.check()` returns `NodeStatus.DRIFTED` when protocols, protocolParticipants, or policyOverrides differ from actual

- [ ] **Step 1: Write failing test for protocols drift**

Add to `ChannelDriftCheckerTest`:

```java
@Test
void protocolsDrifted() {
    UUID id = UUID.randomUUID();
    Channel ch = Channel.builder("proto-ch")
            .id(id).semantic(ChannelSemantic.APPEND)
            .protocols(List.of("VOTING"))
            .build();
    channels.put("proto-ch:" + TENANCY_ID, ch);

    var spec = new ChannelNodeSpec("proto-ch", null, ChannelSemantic.APPEND,
            null, null, null, null, null, null, null, null, null, null, null,
            List.of("NEGOTIATION"), null, null);

    assertEquals(NodeStatus.DRIFTED, checker.check(spec, TENANCY_ID));
}

@Test
void protocolsMatch() {
    UUID id = UUID.randomUUID();
    Channel ch = Channel.builder("proto-ch")
            .id(id).semantic(ChannelSemantic.APPEND)
            .protocols(List.of("NEGOTIATION"))
            .build();
    channels.put("proto-ch:" + TENANCY_ID, ch);

    var spec = new ChannelNodeSpec("proto-ch", null, ChannelSemantic.APPEND,
            null, null, null, null, null, null, null, null, null, null, null,
            List.of("NEGOTIATION"), null, null);

    assertEquals(NodeStatus.PRESENT, checker.check(spec, TENANCY_ID));
}
```

- [ ] **Step 2: Write failing test for policyOverrides drift**

```java
@Test
void policyOverridesDrifted() {
    UUID id = UUID.randomUUID();
    Channel ch = Channel.builder("policy-ch")
            .id(id).semantic(ChannelSemantic.APPEND)
            .policyOverrides(Map.of("negotiation.acceptance", "unanimous"))
            .build();
    channels.put("policy-ch:" + TENANCY_ID, ch);

    var spec = new ChannelNodeSpec("policy-ch", null, ChannelSemantic.APPEND,
            null, null, null, null, null, null, null, null, null, null, null,
            null, null,
            Map.of("negotiation.acceptance", "majority"));

    assertEquals(NodeStatus.DRIFTED, checker.check(spec, TENANCY_ID));
}

@Test
void policyOverridesExtraKeysAreDrift() {
    UUID id = UUID.randomUUID();
    Channel ch = Channel.builder("policy-ch")
            .id(id).semantic(ChannelSemantic.APPEND)
            .policyOverrides(Map.of("negotiation.acceptance", "majority", "extra.key", "val"))
            .build();
    channels.put("policy-ch:" + TENANCY_ID, ch);

    var spec = new ChannelNodeSpec("policy-ch", null, ChannelSemantic.APPEND,
            null, null, null, null, null, null, null, null, null, null, null,
            null, null,
            Map.of("negotiation.acceptance", "majority"));

    assertEquals(NodeStatus.DRIFTED, checker.check(spec, TENANCY_ID));
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn --batch-mode -o test -pl deployment -Dtest="ChannelDriftCheckerTest"`
Expected: FAIL — drift checker doesn't check protocols or policyOverrides

- [ ] **Step 4: Add listMatch and mapMatch helpers to ChannelDriftChecker**

```java
private boolean listMatch(String fieldName, String channelName,
                           List<String> desired, List<String> actual) {
    Set<String> desiredSet = desired != null ? new TreeSet<>(desired) : Set.of();
    Set<String> actualSet = actual != null ? new TreeSet<>(actual) : Set.of();
    if (desiredSet.equals(actualSet)) {
        return true;
    }
    LOG.debugf("channel %s: %s drifted [%s → %s]", channelName, fieldName, desiredSet, actualSet);
    return false;
}

private boolean mapMatch(String fieldName, String channelName,
                          Map<String, String> desired, Map<String, String> actual) {
    Map<String, String> desiredMap = desired != null ? desired : Map.of();
    Map<String, String> actualMap = actual != null ? actual : Map.of();
    if (desiredMap.equals(actualMap)) {
        return true;
    }
    LOG.debugf("channel %s: %s drifted [%s → %s]", channelName, fieldName, desiredMap, actualMap);
    return false;
}
```

- [ ] **Step 5: Add three drift checks to check() method**

After the existing `barrierContributors` check, add:

```java
if (!listMatch("protocols", channelSpec.name(), channelSpec.protocols(), ch.protocols())) {
    drifted = true;
}
if (!listMatch("protocolParticipants", channelSpec.name(),
               channelSpec.protocolParticipants(), ch.protocolParticipants())) {
    drifted = true;
}
if (!mapMatch("policyOverrides", channelSpec.name(),
              channelSpec.policyOverrides(), ch.policyOverrides())) {
    drifted = true;
}
```

- [ ] **Step 6: Run all drift checker tests**

Run: `mvn --batch-mode -o test -pl deployment -Dtest="ChannelDriftCheckerTest"`
Expected: all tests PASS

- [ ] **Step 7: Run full deployment test suite**

Run: `mvn --batch-mode -o test -pl deployment`
Expected: all tests PASS

- [ ] **Step 8: Commit**

```bash
git add deployment/
git commit -m "feat(#187): add drift detection for protocols and policyOverrides

Add listMatch and mapMatch helpers to ChannelDriftChecker.
Detect drift on protocols, protocolParticipants, and policyOverrides.
Extra keys in actual policyOverrides are treated as drift (replace semantics).

Refs #187"
```

### Task 4: YAML deserialization verification

**Files:**
- Modify: `deployment/src/test/resources/test-deployment/topology.yaml` (add protocols to existing channel)
- Modify: `deployment/src/test/java/io/casehub/ops/deployment/DeploymentGoalCompilerTest.java` (or equivalent loader test)

**Interfaces:**
- Consumes: `ChannelNodeSpec` with new fields from Task 1
- Produces: verification that Jackson YAML correctly deserializes protocols, protocol-participants, policy-overrides from YAML into the Java record

- [ ] **Step 1: Write failing test for YAML deserialization**

Add a test that loads a YAML string with the new channel fields and asserts they deserialize:

```java
@Test
void channelYamlDeserializesProtocolsAndPolicyOverrides() throws Exception {
    String yaml = """
            agents: []
            channels:
              - spec:
                  name: test/review
                  semantic: APPEND
                  protocols: [NEGOTIATION]
                  protocol-participants: [agent-a, agent-b]
                  policy-overrides:
                    negotiation.acceptance: majority
                    negotiation.max-rounds: "3"
            caseTypes: []
            trust: []
            endpoints: []
            detections: []
            pools: []
            applications: []
            """;
    DeploymentGoals goals = new DeploymentGoalLoader().load(
            new java.io.ByteArrayInputStream(yaml.getBytes()));

    assertThat(goals.channels()).hasSize(1);
    var spec = goals.channels().get(0).spec();
    assertThat(spec.protocols()).containsExactly("NEGOTIATION");
    assertThat(spec.protocolParticipants()).containsExactly("agent-a", "agent-b");
    assertThat(spec.policyOverrides()).containsEntry("negotiation.acceptance", "majority");
    assertThat(spec.policyOverrides()).containsEntry("negotiation.max-rounds", "3");
}
```

- [ ] **Step 2: Run test**

Run: `mvn --batch-mode -o test -pl deployment -Dtest="*GoalCompiler*#channelYamlDeserializesProtocolsAndPolicyOverrides"`
Expected: PASS (Jackson kebab-case mapping should handle this automatically via `@JsonIgnoreProperties`)

If FAIL: investigate whether `PropertyNamingStrategies.KEBAB_CASE` is configured on the ObjectMapper used by `DeploymentGoalLoader`. The test will reveal the mapping path.

- [ ] **Step 3: Commit**

```bash
git add deployment/
git commit -m "test(#187): verify YAML deserialization of channel protocols and policyOverrides

Refs #187"
```

## Batch 2: Level 15 Governed Interaction Tutorial

### Task 5: Level 15 topology YAML and compilation test

**Files:**
- Create: `topology-tests/src/test/resources/topologies/deployment/15-governed-interaction/15-governed-interaction.yaml`
- Modify: `topology-tests/src/test/java/io/casehub/ops/topology/reconciliation/DeploymentReconciliationTest.java`

**Interfaces:**
- Consumes: `ChannelNodeSpec` with protocols/policyOverrides from Batch 1
- Produces: topology YAML that compiles and reconciles correctly, test methods asserting node ordering and provisioning success

- [ ] **Step 1: Create Level 15 topology YAML**

Create `topology-tests/src/test/resources/topologies/deployment/15-governed-interaction/15-governed-interaction.yaml`:

```yaml
# Level 15: Governed Interaction
# Methodology step: GOVERN — Negotiation Patterns
# Prerequisites: Level 01
# See README.md for walkthrough

desiredState:
  namespace: casehub
  name: governed-interaction

nodes:
  coordinator:
    type: agent
    spec:
      agentId: coordinator
      name: Design Coordinator
      slot: worker

  reviewer-1:
    type: agent
    dependsOn: [coordinator]
    spec:
      agentId: reviewer-1
      name: Reviewer Alpha
      slot: worker

  reviewer-2:
    type: agent
    dependsOn: [coordinator]
    spec:
      agentId: reviewer-2
      name: Reviewer Beta
      slot: worker

  design-review:
    type: channel
    dependsOn: [coordinator, reviewer-1, reviewer-2]
    spec:
      name: design-review
      semantic: APPEND
      protocols: [NEGOTIATION]
      protocol-participants: [coordinator, reviewer-1, reviewer-2]
      policy-overrides:
        negotiation.acceptance: majority
        negotiation.max-rounds: "3"
```

- [ ] **Step 2: Write test for topology compilation**

Add to `DeploymentReconciliationTest`:

```java
@Test
void governedInteraction_compilesWithChannelProtocols() throws IOException {
    DesiredStateGraph graph = compileSingleGraph(
            "topologies/deployment/15-governed-interaction/15-governed-interaction.yaml",
            DEPLOYMENT_REGISTRY);
    assertThat(graph.nodes()).hasSize(4); // 3 agents + 1 channel
    assertThat(graph.nodes().get(NodeId.of("design-review")).spec())
            .isInstanceOf(ChannelNodeSpec.class);
    ChannelNodeSpec channelSpec = (ChannelNodeSpec) graph.nodes()
            .get(NodeId.of("design-review")).spec();
    assertThat(channelSpec.protocols()).containsExactly("NEGOTIATION");
    assertThat(channelSpec.policyOverrides())
            .containsEntry("negotiation.acceptance", "majority");
}
```

- [ ] **Step 3: Write test for dependency ordering**

```java
@Test
void governedInteraction_channelDependsOnAgents() throws IOException {
    DesiredStateGraph graph = compileSingleGraph(
            "topologies/deployment/15-governed-interaction/15-governed-interaction.yaml",
            DEPLOYMENT_REGISTRY);
    TransitionPlan plan = planFromEmpty(graph);
    assertOrderedBefore(plan, "coordinator", "design-review");
    assertOrderedBefore(plan, "reviewer-1", "design-review");
    assertOrderedBefore(plan, "reviewer-2", "design-review");
}
```

- [ ] **Step 4: Write test for execution success**

```java
@Test
void governedInteraction_executesSuccessfully() throws IOException {
    DesiredStateGraph graph = compileSingleGraph(
            "topologies/deployment/15-governed-interaction/15-governed-interaction.yaml",
            DEPLOYMENT_REGISTRY);
    TransitionPlan plan = planFromEmpty(graph);
    TransitionResult result = executeTransition(plan, "test-tenant");
    assertThat(result.outcomes().values())
            .allMatch(o -> o instanceof StepOutcome.Succeeded);
}
```

- [ ] **Step 5: Run topology tests**

Run: `mvn --batch-mode test -pl topology-tests -Dtest="DeploymentReconciliationTest#governedInteraction*"`
Expected: all 3 tests PASS

- [ ] **Step 6: Commit**

```bash
git add topology-tests/
git commit -m "feat(#188): add Level 15 governed interaction topology YAML and compilation tests

Three agents (coordinator, two reviewers) communicate via a channel
with NEGOTIATION protocol. Tests verify compilation, dependency ordering,
and provisioning execution.

Refs #188"
```

### Task 6: Full build verification

**Files:** none (verification only)

**Interfaces:**
- Consumes: all changes from Tasks 1-5
- Produces: green build across all modules

- [ ] **Step 1: Run full project build**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS across all modules

- [ ] **Step 2: Verify no regressions**

Check that no pre-existing tests broke. Pay attention to:
- `api/` tests (ChannelNodeSpec constructor changes)
- `deployment/` tests (provisioner, drift checker)
- `topology-tests/` (all reconciliation tests)
- `service/` tests (if any reference ChannelNodeSpec)

- [ ] **Step 3: Commit any fixes if needed**

If any tests fail, fix and commit with: `fix(#187): resolve <issue>`

## References

- [2026-10-25-channel-patterns-governed-interaction-design.md] — design spec this plan implements
- `api/src/main/java/io/casehub/ops/api/deployment/ChannelNodeSpec.java:11-48` — current spec record
- `deployment/src/main/java/io/casehub/ops/deployment/handler/ChannelProvisionHandler.java:22-90` — current provisioner
- `deployment/src/main/java/io/casehub/ops/deployment/drift/ChannelDriftChecker.java:24-164` — current drift checker
- `deployment/src/test/java/io/casehub/ops/deployment/handler/ChannelProvisionHandlerTest.java` — existing provisioner tests
- `deployment/src/test/java/io/casehub/ops/deployment/drift/ChannelDriftCheckerTest.java` — existing drift tests
- `topology-tests/src/test/java/io/casehub/ops/topology/reconciliation/DeploymentReconciliationTest.java` — existing topology tests
- `qhorus/api/src/main/java/io/casehub/qhorus/api/channel/ChannelCreateRequest.java` — builder with protocols
- `qhorus/runtime-core/src/main/java/io/casehub/qhorus/runtime/channel/ChannelService.java:162-236` — setProtocols, setPolicyOverrides
- GitHub #186 — governed interaction patterns epic
- GitHub #187 — channel pattern support
- GitHub #188 — governed interaction tutorial level
