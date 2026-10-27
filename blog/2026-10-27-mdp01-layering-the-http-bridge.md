---
layout: post
title: "Layering the HTTP Bridge — Why Testing Parity Forced a Platform Change"
date: 2026-10-27
entry_type: note
subtype: diary
projects: [casehubio/casehub-ops]
tags: [architecture, testing, multi-env, deployment, platform]
series: issue-207-multi-env-deploy-testing
---

# Layering the HTTP Bridge — Why Testing Parity Forced a Platform Change

After landing #206 — unified `serviceUrl()` resolution across LOCAL, K8S, and PODMAN — the three environments finally share the same deploy → health-wait → assert flow. Same structure. But not the same depth.

LOCAL gets the full `FleetLlmExerciser`: load agent descriptors from the classpath, construct a `RoutingAgentProvider`, send a real prompt through the LangChain4j bridge, assert the response is substantive and domain-relevant. K8S and PODMAN get `assertHttpOk(url + "/q/health/ready")`. The container is alive. Whether it does anything useful — nobody checks.

I started the design session for #207 thinking the fix was straightforward: write an HTTP client that POSTs to the deployed container and asserts the response. Build the exerciser, wire it in, done. But the question that unravelled this was simpler than that: what endpoint does it POST to?

## The gap nobody built for

The casehub platform has a well-designed Java-level agent interaction API. `AgentProvider.invoke()` accepts a system prompt and user prompt, returns a reactive stream of `AgentEvent` — text deltas, tool calls, an `InvocationComplete` terminal event carrying token counts, cost, and routing metadata. Clean, typed, streaming.

But this API exists only in-JVM. The only HTTP path to an agent today is claudony's terminal session API — send raw keystrokes to a tmux session, read raw text back. No structure. No metadata. The platform has `AgentProvider`, `AgentSession`, `AgentEvent` — a complete interaction model — and zero HTTP exposure for any of it.

Claude dug through three repos — eidos, platform, claudony — cataloguing every REST endpoint, every JAX-RS resource, every Vert.x route handler. The finding was definitive: structured HTTP agent invocation doesn't exist. The exerciser can't POST to an endpoint that nobody built.

## Where the bridge belongs

The obvious reaction is "add an endpoint." The interesting question is where.

I considered three options. A single endpoint in each app — duplicative, no standard contract. Two separate endpoints composed by the client — eidos for agent identity, platform for invocation, the exerciser stitching them together with two HTTP calls. Or a single endpoint with an SPI that lets eidos plug into platform's invocation mechanics.

The third option won because of where the data comes from. The exerciser needs the response text and routing metadata (from platform's `AgentProvider`) but also the agent's role and domain vocabulary (from eidos's `AgentDescriptor`). If the caller has to construct the system prompt, it needs the rendered prompt from eidos's `SystemPromptRenderer` — not just the raw `briefing` field, which is a subset of what the renderer produces.

Platform can't depend on eidos — wrong dependency direction. But platform can define an SPI. `AgentResolver`: given an agentId, return the rendered system prompt, slot, and domain. Eidos implements it. CDI discovers it. When eidos is on the classpath, the resolved mode works automatically. When it isn't, callers provide their own system prompt.

One POST. `{agentId: "my-llm", userPrompt: "..."}`. The app resolves the agent internally. The exerciser becomes trivially thin.

## The decision that surprised me

I went into this expecting the deployment path to be the contentious decision — should K8S and PODMAN deploy through the full `TransitionPlanner` → provisioner pipeline, or bypass it and call `target.deploy()` directly?

Claude argued for the direct path. The reconciliation test suite already validates the runtime. LOCAL validates the runtime → provisioner → target bridge. What multi-env tests uniquely prove is that the deployment target implementations work with real infrastructure — fabric8 actually creates K8s Deployments, Podman REST API actually creates containers, port mapping actually resolves. Running those containers through the TransitionPlanner adds six layers of indirection to the failure diagnosis without adding coverage that doesn't already exist elsewhere.

I found this convincing on first principles. The multi-env tests aren't testing the desiredstate runtime. They're testing the last mile — the infrastructure contact point.

The decision that actually mattered was the one I hadn't anticipated: the `ImageResolver` fallback chain. The original design was three tiers — local → ghcr.io → nginx:alpine — as a cascading fallback. Claude's review caught the problem: a fallback chain silently degrades correctness. Developer changes agent routing logic, forgets to rebuild the local image, registry has the last-published image without the change, test passes, developer concludes it works. It doesn't.

The fix is mutually exclusive tiers. Dev gets local-only — fail loud if the image is missing. CI gets registry-only — the image was just built in the prior pipeline stage. Smoke tests get nginx:alpine — explicitly opted in. Each context gets exactly one image source. No silent degradation.

## What this opens up

The platform `agent-rest` module isn't just test infrastructure. It's the first structured HTTP interface to the agent interaction model. External integrations can invoke agents by ID. API-first access becomes possible. The exerciser motivated it, but the surface area is broader.

The more interesting implication is what happens when the production networking tier (#211) lands. Right now, `serviceUrl()` returns a bare URL string — `http://localhost:30123`. That works for NodePort and container port mapping. But Ingress returns `https://hostname/path` — TLS, path prefixes, certificate trust. The abstraction that works for test-tier networking won't survive contact with production networking. `serviceUrl()` needs to evolve from "give me a URL" to "give me a configured client for this service." That's a future design session, but the shape of the problem is already visible.
