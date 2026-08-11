---
layout: post
title: "Filling the Topology Gap — Endpoints in the Deployment Module"
date: 2026-06-26
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ops]
tags: [deployment, endpoints, desired-state]
series: issue-6-deployment-goal-compiler
---

The deployment module started as four node types: agents, channels, case types, trust policies. I'd always known this was incomplete — a deployment that can declare *who does the work* but not *what infrastructure it connects to* is missing half the picture. Issue #6 was filed before any of the deployment module existed, and its original spec described sub-compilers, streams, detection, connectors — a different architecture entirely. By the time I got to it, the deployment module had already been built across #2 and #7 with a deliberately simpler design. The question was whether #6 still had real work in it.

It did, but not where the original issue expected.

The platform's `EndpointRegistry` has been sitting there since platform#73 — seven protocols (KAFKA, AMQP, CAMEL, GRPC, HTTP, MCP, QHORUS), typed capabilities (SEND, RECEIVE, QUERY, DISPATCH), Path-based addressing, credential references. The stream modules (`streams-kafka`, `streams-camel`, `streams-poll`) already discover endpoints from the registry and set up ingestion pipelines. `streams-camel` even reacts dynamically to `EndpointRegistered` CDI events, adding Camel routes when new CAMEL endpoints appear. The machinery was connected everywhere except the deployment module — the one place that's supposed to declare what a deployment's topology looks like.

The design turned out to be straightforward once I stopped trying to fit the original #6 spec. One `EndpointNodeSpec` record, protocol-agnostic, mirroring `EndpointDescriptor` minus the `tenancyId` (which comes from `ProvisionContext` at provision time). Protocol-specific validation catches the cross-module required properties — `EndpointPropertyKeys.TOPIC` for KAFKA, `EndpointPropertyKeys.URL` for HTTP/GRPC — without trying to encode every protocol's internal configuration. A `toDescriptor(tenancyId)` method serves as the single conversion point: the provisioner calls `register(spec.toDescriptor(context.tenancyId()))`, the drift checker calls `resolved.equals(spec.toDescriptor(tenancyId))`. Since `EndpointDescriptor` is a record, `equals()` handles drift comparison structurally — no field-by-field fragility.

The implementation landed in three tasks. We extended the sealed hierarchy from four to five permits, which broke the exhaustive switches in the provisioner — exactly the right kind of breakage, forcing every dispatch site to be explicit about the new type. The handler is eight lines of real logic. The drift checker is five. The compiler gained one line: `compileEntries(goals.endpoints(), nodes, dependencies)`. Cross-type dependencies work naturally — an agent with `dependsOn: [tools/github]` gets a graph edge to that endpoint, and the `TransitionPlanner` provisions endpoints first.

The review caught a real issue: `readActual(DesiredStateGraph)` had been updated to `readActual(DesiredStateGraph, String tenancyId)` in `casehub-desiredstate-api`, and the deployment module's adapter was updated during this branch — but infra, IoT, and compliance still had the old one-arg override. The full multi-module build surfaced it. SPI signature changes in dependency JARs that don't fail until the cross-module build are the kind of thing that makes you grateful for a mandatory `mvn install` before close.

The gap that remains is detection. `casehub-ras` exists and observes CloudEvents from the stream modules, routing to pluggable Ganglion strategies — but it has no declarative configuration API. When RAS gains a registration surface, a `DetectionNodeSpec` becomes natural (#11). The full pipeline — endpoints → streams → CloudEvent → RAS → startCase() — would then be entirely under desired-state management.
