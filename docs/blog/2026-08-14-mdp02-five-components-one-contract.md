---
layout: post
title: "Five Components, One Contract"
date: 2026-08-14
entry_type: note
subtype: diary
projects: [casehub-ops, blocks-ui]
tags: [lit, web-components, blocks-ui, component-architecture]
series: issue-38-stubbed-ui-screens
---

*Continues from [2026-08-14: When Stubs Become the Architecture](2026-08-14-mdp01-stubs-to-services.md).*

The backend wiring is done. Every ops console endpoint returns real data. Now the question: can five Lit components consume those APIs without each one reinventing the data fetching, event emission, and SSE lifecycle?

The answer turned out to be yes, and the reason is a contract that blocks-ui already had: the dual-data-mode pattern. Every component accepts either a `data` prop (for testing, showcases, and parent-controlled rendering) or an `endpoint` URL (for live data). The component never cares which mode it's in. `DataSourceMixin` handles the fetch lifecycle; `emitPagesEvent` standardises outbound events as `pages-event` CustomEvents with a topic and payload.

Service-card, cluster-panel, dimension-dashboard — these three are straightforward applications of the pattern. Data in, render, emit selection events. The interesting parts are the other two.

## SSE as a lifecycle concern

Reconciliation-status and topology-viewer both support live updates. Each creates an `SSEManager` instance and subscribes in `connectedCallback`, unsubscribes in `disconnectedCallback`. The SSE handler writes directly to the component's state, which triggers Lit's reactive render cycle.

What makes this clean is that SSE is additive. The component works identically with just the `data` prop — SSE is a progressive enhancement. A test harness sets `data` directly. A production dashboard sets `sseEndpoint` and the component updates itself. The same component serves both uses without conditional logic in the template.

## The graph canvas question

Topology-viewer converts a `TopologySnapshot` into a `GraphModel` — the same graph-core type that blocks-dag-viewer uses for HTN execution plans. Nodes carry service status; edges carry dependency labels. The conversion is a pure function: domain types in, graph types out.

The actual rendering uses a card-based layout with status-coloured borders and an edge list — functional but not the eventual form. The real rendering will come from `pages-graph-canvas` with React Flow and ELK layout, the same stack that powers the DAG viewer and the case diagram editor. I left a comment stub at the integration point — identical to the one in blocks-dag-viewer — because the graph canvas integration is a cross-cutting concern that belongs in the rendering layer, not in five separate components. When it lands, topology-viewer picks it up by swapping the template, not the data model.

The decision to defer is deliberate. The `toGraphModel` transform is the stable part — it maps service status to node decorations and dependency edges to graph edges. The rendering is the volatile part. Build the stable contract first; the rendering can vary without touching the component's API.
