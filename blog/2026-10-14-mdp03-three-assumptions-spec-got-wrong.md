---
layout: post
title: "Three Assumptions the Spec Got Wrong"
date: 2026-10-14
entry_type: note
subtype: diary
projects: [casehubio/casehub-ops]
tags: [integration-testing, quarkus, desired-state, deployability, testcontainers]
series: issue-139-real-instance-test-infra
---

# Three Assumptions the Spec Got Wrong

I set out to build integration test infrastructure that starts real CaseHub services as separate JVM processes — the foundation for every topology test level above the current WireMock stubs. The idea was straightforward: start claudony, eidos, and engine as separate processes, point the deployment target at real claudony, exercise the registration API.

Three assumptions collapsed during design review.

## Eidos and engine aren't apps

The spec assumed three deployable services. Only one exists. Claudony has an `app/` submodule with `quarkus-maven-plugin` and a Dockerfile. Eidos and engine are libraries — Quarkus extensions with no standalone entry point. They're embedded in `casehub-ops-app`, not deployed independently.

This changes what "deploy real instances" means for the proof-of-concept. Instead of three separate JVM processes (claudony, eidos, engine), the realistic deployment is two: claudony as the fleet manager, and ops-app as the Quarkus shell that embeds everything else. The topology YAML declares eidos and engine as separate nodes, but the nodes describe what claudony *manages*, not what runs as independent processes.

The fix for later levels: add `app/` submodules to eidos and engine, following claudony's pattern. Not scaffold — scaffold is a leaf in the dependency graph. It depends on engine. Using it to wrap engine creates a circular build dependency. App folders keep the DAG clean and the packaging collocated with the library.

## Dev Services don't activate in pre-built jars

Claudony needs PostgreSQL. In `@QuarkusTest` or `quarkus:dev`, Dev Services auto-start a container. I expected the same when starting a pre-built jar with `-Dquarkus.profile=test`.

It doesn't work. Dev Services are part of Quarkus augmentation — the build-time step. When you run a pre-built jar, augmentation already happened. The runtime has no mechanism to start containers; it just reads the datasource config and finds nothing listening.

The workaround: Testcontainers PostgreSQL started in the test JVM, connection URL passed to claudony via system properties. Docker is back in the picture, but only for the database — the services themselves are plain JVM processes.

## registerApp is metadata-only

The existing `DeploymentTargetIntegrationTest` uses WireMock to verify that `ClaudonyDeploymentTarget` makes the right HTTP calls. The proof-of-concept replaces WireMock with a real claudony instance — but the underlying operation, `registerApp`, stores metadata in claudony's registry. It doesn't start a process or deploy an application.

The proof-of-concept still validates real things the WireMock test can't: the full REST call path through a real Quarkus HTTP server, actual database persistence of the registration, and health-check polling against a real `/q/health/ready` endpoint. But end-to-end deployment of running services waits for the eidos/engine app modules.

## What got built

The infrastructure works. `ServiceProcess` wraps `ProcessBuilder` for Quarkus fast-jars — jar location by convention, dynamic port allocation via `ServerSocket(0)`, health polling, log capture, graceful shutdown. `ServiceClusterExtension` orchestrates multiple processes with Testcontainers PostgreSQL provisioning and `@ServiceEndpoint` field injection. `@LlmAvailable` gates tests by API key presence, following the `@KubernetesAvailable` pattern.

The interesting design choice: the `@ServiceCluster` annotation declares services declaratively, and the extension handles everything — database provisioning, port allocation, auth bypass, process lifecycle. The test class stays clean. Later, when eidos and engine become deployable, the same infrastructure extends without changing the test API.
