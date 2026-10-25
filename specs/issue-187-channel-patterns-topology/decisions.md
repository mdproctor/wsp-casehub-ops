# Design Decisions — issue-187-channel-patterns-topology

## D1: PolicyOverrides semantics

**Choice:** Replace semantics — YAML declares the complete policyOverrides, any extra keys on the live channel are drift
**Alternatives:**
- Merge semantics — only ensure declared keys are present, allow external tooling to add extras without triggering drift
**Rationale:** Simpler mental model, matches how all other spec fields work in the desired-state system. The YAML is the source of truth.
**Trade-offs:** External tooling cannot add overrides without updating the YAML
**Sources:** qhorus ChannelService.setPolicyOverrides() merge behavior, ChannelDriftChecker patterns
**Exploration:** quick
**Status:** captured

## D2: Fields to expose in ChannelNodeSpec

**Choice:** protocols (List<String>), protocolParticipants (List<String>), policyOverrides (Map<String,String>)
**Alternatives:**
- Protocols only (strict #187 scope) — would need a follow-up for protocolParticipants
- All unexposed ChannelCreateRequest fields — larger scope, harder to review, different concerns
**Rationale:** protocolParticipants is trivial alongside protocols and avoids a follow-up. Other fields (reviewerInstances, enforcementMode, etc.) are separate concerns.
**Trade-offs:** Other ChannelCreateRequest fields remain unexposed until explicitly needed
**Sources:** qhorus Channel record (protocols, protocolParticipants, policyOverrides fields), ChannelCreateRequest builder
**Exploration:** quick
**Status:** captured

## D3: Tutorial level number

**Choice:** Level 15
**Alternatives:**
- Level 13 (slot before fault recovery) — adjacent to adaptation rules but breaks existing numbering
**Rationale:** Next sequential number after existing 14 levels. Governed interaction builds on the full progression. Renumbering is mechanical if reordering is needed later.
**Trade-offs:** None significant — level numbers are just naming convention
**Sources:** topology-tests/src/test/resources/topologies/deployment/ existing levels 01-14
**Exploration:** quick
**Status:** captured

## D4: Test approach for Level 15

**Choice:** Live integration test in topology-tests that deploys and runs the full stack
**Alternatives:**
- Unit test with stubs — would verify YAML compilation and provisioner wiring but not protocol enforcement
- Both unit + integration — more coverage but more work
**Rationale:** The point of topology tests is to prove the topology deploys AND runs. They also serve as tutorials for end users. Simulated fallback via manifest when LLM is unavailable.
**Trade-offs:** Requires the full platform stack (claudony, agency, qhorus) running
**Sources:** Existing topology-tests integration test patterns
**Exploration:** quick
**Status:** captured

## D5: LLM provider for integration tests

**Choice:** Jlama via the standard platform routing: eidos → manifest → AgentProvider → langchain4j → JlamaChatModel. Pure Java in-process inference, no external dependencies.
**Alternatives:**
- Ollama with a small model — requires external process, not pure Java
- New agent-jlama platform module — unnecessary given existing langchain4j bridge
- Direct Jlama usage — bypasses the platform stack, wouldn't prove the routing works
**Rationale:** Tests must exercise the full platform routing chain. Manifest is configured with a langchain4j-backed provider using JlamaChatModel. The test topology declares agents normally; manifest routes to the Jlama-backed provider. Tests need structured input/output at good speed, not deep reasoning. Simulated fallback via manifest when no LLM is available.
**Trade-offs:** Slower inference than native/GPU-backed solutions, but adequate for structured test responses. Requires langchain4j-jlama available as a manifest-routable provider in the platform.
**Sources:** platform agent-langchain4j-core ChatModelAgentProvider, eidos agent identity, manifest provider config
**Exploration:** quick
**Depends on:** D4 (test approach)
**Status:** captured

## D6: Implementation approach

**Choice:** Incremental extension of existing patterns (Approach A)
**Alternatives:**
- Reflective field pass-through with Map<String, Object> extensions — breaks type-safe pattern
- ChannelNodeSpec as wrapper around ChannelCreateRequest — couples ops to qhorus API evolution
**Rationale:** Follows established pattern for how every other field was added. Type-safe, reviewable, consistent with existing code.
**Trade-offs:** Future channel fields still need explicit wiring
**Sources:** ChannelProvisionHandler, ChannelDriftChecker, AgentNodeSpec as reference patterns
**Exploration:** quick
**Status:** captured
