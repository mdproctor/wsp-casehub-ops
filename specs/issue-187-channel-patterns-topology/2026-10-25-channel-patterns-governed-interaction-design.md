# Channel Patterns & Governed Interaction — Design Spec

**Issues:** casehubio/casehub-ops#187, casehubio/casehub-ops#188
**Branch:** issue-187-channel-patterns-topology
**Date:** 2026-10-25

## Summary

Extend the topology YAML channel node type to support `protocols`, `protocol-participants`, and `policy-overrides` stanzas, mapping to qhorus `Channel.protocols`, `Channel.protocolParticipants`, and `Channel.policyOverrides` at provisioning time. Add a Level 15 tutorial demonstrating governed interaction via the NEGOTIATION protocol.

## Scope

### In scope

- `ChannelNodeSpec` gains three fields: `protocols`, `protocolParticipants`, `policyOverrides`
- `ChannelProvisionHandler` switches to `ChannelCreateRequest.builder()`, wires protocols at creation, policyOverrides post-creation
- `ChannelDriftChecker` gains drift detection for protocols, protocolParticipants, policyOverrides
- `ChannelOperations` interface gains `setProtocols`, `setProtocolParticipants`, `setPolicyOverrides`
- Level 15 topology YAML and integration test in `topology-tests`
- Jlama-backed LLM via manifest → langchain4j for integration tests

### Out of scope

- Other unexposed `ChannelCreateRequest` fields (reviewerInstances, enforcementMode, trackDelivery, etc.)
- New platform provider modules — Jlama is accessed through existing langchain4j bridge

## Prerequisites

Both external blockers are resolved (CLOSED):
- casehubio/qhorus#496 — ChannelProjector API facade
- casehubio/agency#354 — NegotiationProtocol as qhorus ChannelProtocol

## Architecture

### Layer 1 — API (`api/` module)

`ChannelNodeSpec` record gains three optional fields:

```java
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
        List<String> protocols,                // NEW
        List<String> protocolParticipants,      // NEW
        Map<String, String> policyOverrides     // NEW
) implements DeploymentNodeSpec { ... }
```

Compact constructor applies defensive copies:
- `protocols = protocols != null ? List.copyOf(protocols) : null;`
- `protocolParticipants = protocolParticipants != null ? List.copyOf(protocolParticipants) : null;`
- `policyOverrides = policyOverrides != null ? Map.copyOf(policyOverrides) : null;`

### Layer 2 — Deployment (`deployment/` module)

#### ChannelProvisionHandler

**Creation path:** Switches from the legacy 14-arg `ChannelCreateRequest` constructor to the builder pattern:

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

`policyOverrides` is NOT on `ChannelCreateRequest` — set post-creation.

**Update path:** `updateMutableFields()` adds:

```java
ops.setProtocols(channelId, spec.protocols());
ops.setProtocolParticipants(channelId, spec.protocolParticipants());
ops.setPolicyOverrides(channelId, spec.policyOverrides());
```

**Replace semantics for policyOverrides:** Since qhorus `setPolicyOverrides()` merges (null-value = remove key), the provisioner computes a synthetic diff to achieve replace semantics:

1. Start with the desired map from the spec (or empty if null)
2. For each key present on the actual channel but absent from the desired map, add an entry with null value (signals removal)
3. Pass the combined map to `setPolicyOverrides()`

If the spec's policyOverrides is null, pass null to `setPolicyOverrides()` (clears all overrides). This gives declarative replace semantics through the merge API.

**ChannelOperations interface** gains three methods:

```java
Channel setProtocols(UUID channelId, List<String> protocols);
Channel setProtocolParticipants(UUID channelId, List<String> protocolParticipants);
Channel setPolicyOverrides(UUID channelId, Map<String, String> policyOverrides);
```

#### ChannelDriftChecker

Three new drift checks:

**Protocols** — order-independent list comparison via `TreeSet`:

```java
if (!listMatch("protocols", channelSpec.name(),
               channelSpec.protocols(), ch.protocols())) {
    drifted = true;
}
```

**Protocol participants** — same list comparison pattern.

**Policy overrides** — map comparison with replace semantics. Null/empty on both sides is a match. Any difference in keys or values is drift.

Helper methods:
- `listMatch(fieldName, channelName, List<String> desired, List<String> actual)` — converts both to `TreeSet` (null → empty set), logs drift at DEBUG
- `mapMatch(fieldName, channelName, Map<String,String> desired, Map<String,String> actual)` — compares maps (null → empty map), logs drift at DEBUG

### Layer 3 — Level 15 Integration Test (`topology-tests/` module)

#### Topology YAML

`topology-tests/src/test/resources/topologies/deployment/15-governed-interaction/15-governed-interaction.yaml`:

```yaml
# Level 15: Governed Interaction
# Methodology step: GOVERN — Negotiation Patterns
# Prerequisites: Level 01

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
        negotiation.max-rounds: 3
```

#### Operational Assertions

The integration test verifies the 6 operational assertions:

1. Coordinator sends PROPOSE → commitment opens
2. Reviewer-1 sends RESPONSE (endorse) → negotiation state tracks 1/2
3. Coordinator attempts DONE → BLOCKED (majority not yet reached)
4. Reviewer-2 sends RESPONSE (endorse) → negotiation state tracks 2/2
5. Coordinator sends DONE → ALLOWED (majority reached)
6. Ledger records full negotiation chain with protocol advisories

#### LLM Configuration

The test exercises the full platform routing chain:

**eidos → manifest → AgentProvider → langchain4j → JlamaChatModel**

- Manifest is configured with a langchain4j-backed provider using JlamaChatModel (pure Java in-process GGUF inference)
- Agents receive structured system prompts defining roles and message format
- Tests need structured input/output at good speed, not deep reasoning — a small quantized model suffices
- Manifest fallback (simulated responses) ensures tests pass without a model file present

**Test dependency:** `topology-tests/pom.xml` gains `langchain4j-jlama` as a test-scoped dependency.

## YAML Surface

Existing channel YAML is unaffected — all new fields are optional.

**Flat deployment goals format:**
```yaml
channels:
  - spec:
      name: design-review
      semantic: APPEND
      protocols: [NEGOTIATION]
      protocol-participants: [coordinator, reviewer-1, reviewer-2]
      policy-overrides:
        negotiation.acceptance: majority
        negotiation.max-rounds: 3
```

**Graph-based topology format:**
```yaml
nodes:
  design-review:
    type: channel
    spec:
      name: design-review
      semantic: APPEND
      protocols: [NEGOTIATION]
      protocol-participants: [coordinator, reviewer-1, reviewer-2]
      policy-overrides:
        negotiation.acceptance: majority
        negotiation.max-rounds: 3
```

Jackson handles `kebab-case` → `camelCase` mapping. `@JsonIgnoreProperties(ignoreUnknown = true)` is already present on the record.

## What This Proves

The Level 15 tutorial demonstrates four-layer composition end-to-end:

- **Platform** — CDI discovers agency beans, manifest routes to Jlama-backed provider
- **Qhorus** — enforcement pipeline evaluates NegotiationProtocol, projections maintain state
- **Agency** — NegotiationProtocol + NegotiationProjection provide the interaction logic
- **Ops** — one YAML stanza activates all of the above

## Design Decisions

See [decisions.md](decisions.md) for the full decision log. Key choices:

- **D1:** PolicyOverrides use replace semantics (YAML = desired full set)
- **D2:** Three fields: protocols, protocolParticipants, policyOverrides
- **D3:** Tutorial is Level 15
- **D4:** Live integration tests (deploy + run, not stubs)
- **D5:** Jlama via manifest → langchain4j routing chain
- **D6:** Incremental extension of existing patterns

## References

- `api/src/main/java/io/casehub/ops/api/deployment/ChannelNodeSpec.java` — current spec record
- `deployment/src/main/java/io/casehub/ops/deployment/handler/ChannelProvisionHandler.java` — current provisioner
- `deployment/src/main/java/io/casehub/ops/deployment/drift/ChannelDriftChecker.java` — current drift checker
- `qhorus/api/src/main/java/io/casehub/qhorus/api/channel/Channel.java` — platform Channel with protocols, policyOverrides
- `qhorus/api/src/main/java/io/casehub/qhorus/api/channel/ChannelCreateRequest.java` — builder with protocols support
- `qhorus/runtime-core/src/main/java/io/casehub/qhorus/runtime/channel/ChannelService.java:162-236` — setProtocols, setPolicyOverrides methods
- `platform/agent-langchain4j-core` — ChatModelAgentProvider bridge
- casehubio/qhorus#496 — ChannelProjector (CLOSED)
- casehubio/agency#354 — NegotiationProtocol (CLOSED)
- casehubio/casehub-ops#186 — governed interaction patterns epic
