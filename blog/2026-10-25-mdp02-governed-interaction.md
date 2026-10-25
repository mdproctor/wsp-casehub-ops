---
layout: post
title: "One YAML Stanza, Four Layers Deep"
date: 2026-10-25
entry_type: note
subtype: diary
projects: [casehubio/casehub-ops]
tags: [channels, protocols, negotiation, desired-state, governed-interaction]
series: issue-187-channel-patterns-topology
---

# One YAML Stanza, Four Layers Deep

Most infrastructure tools stop at deployment. You declare what you want running, and the system makes it so. CaseHub's desired-state engine does that too — agents, channels, trust policies, applications — but deployment is just the first layer. The interesting question is what happens after everything is running.

Consider three agents collaborating on a design decision. A coordinator proposes. Two reviewers endorse or counter-propose. The coordinator can only close the discussion after a majority agrees. This is a negotiation protocol — a pattern of interaction where the channel itself enforces the rules, not the agents.

The YAML to declare this looks unremarkable:

```yaml
design-review:
  type: channel
  spec:
    name: design-review
    semantic: APPEND
    protocols: [NEGOTIATION]
    protocolParticipants: [coordinator, reviewer-1, reviewer-2]
    policyOverrides:
      negotiation.acceptance: majority
      negotiation.max-rounds: 3
```

Three new fields on a channel spec. But what happens when the reconciliation loop provisions this channel is where it gets interesting.

## Four layers activate from one stanza

The ops module reads `protocols: [NEGOTIATION]` and passes it through to qhorus at creation time. Qhorus registers the protocol with its enforcement pipeline. The agency module's `NegotiationProtocol` — discovered via CDI — provides the protocol logic: what PROPOSE means, how RESPONSE votes are tracked, when DONE is allowed. And the platform routes agent inference through manifest to whichever LLM provider is configured — in our integration tests, that's Jlama via langchain4j, running pure Java in-process.

One YAML stanza activates:
- **Ops** — parses the YAML, compiles the desired-state graph, provisions the channel
- **Qhorus** — enforcement pipeline evaluates the protocol on every message
- **Agency** — `NegotiationProtocol` provides the interaction semantics
- **Platform** — manifest routes agent requests through the configured provider

The agent that sends DONE before majority endorsement gets blocked — not by its own code, not by application logic, but by the infrastructure. The channel won't accept the message. This is what "governed interaction" means: the topology declares the rules, and the platform enforces them.

## Replace semantics — a design decision that matters

`policyOverrides` controls protocol behaviour. `negotiation.acceptance: majority` tells the protocol what "enough agreement" means. The question is: when the YAML declares overrides, does that mean "ensure these keys exist" (merge) or "this is the complete set" (replace)?

I chose replace semantics. The YAML is the source of truth. If the live channel has an extra override key that isn't in the YAML, that's drift — the reconciliation loop should fix it.

The catch: qhorus's `setPolicyOverrides` API uses merge semantics internally. You pass a map; it merges with existing keys; null values signal removal. To get replace semantics through a merge API, the provisioner computes a synthetic diff — desired entries plus null-valued entries for every key present on the live channel but absent from the spec. Claude caught this during code review. I'd wired the update path to pass the spec's overrides straight through, which would have silently preserved stale keys on existing channels. The drift checker would correctly flag the mismatch, but the provisioner couldn't fix it.

The fix is a `computePolicyOverrideDiff` helper that builds the diff map before calling the merge API. Simple once you see it, invisible if you don't.

## What this opens up

Negotiation is the first governed interaction pattern, but the architecture supports any protocol that qhorus can enforce. Voting, approval chains, structured debate, sequential review — each is a protocol implementation that a single YAML field activates. The agent doesn't need to know about the protocol's internals. It sends messages; the channel decides whether to accept them.

The Level 15 tutorial topology is deliberately minimal — three agents, one channel, one protocol. The point isn't complexity. The point is that the four-layer composition works end-to-end, and the person writing the topology YAML doesn't need to know it's four layers. They write `protocols: [NEGOTIATION]` and the infrastructure makes it true.
