# Design Journal — issue-118-extract-ops-service-module

## 2026-10-04 — #116 landed, #118 designed

Delivered #116 (PoolNodeSpec — pool as 7th deployment node type). Full cycle:
brainstorm → plan → implement → code review → branch audit → squash → merge.
226 deployment tests green. Also fixed pre-existing test breakage (stale
AgentDescriptor constructor, TransitionPlan API migration).

Architecture discussion led to three new issues:
- **#117** — deployment profiles and bootstrap model. Ops replaces Terraform +
  Ansible + Helm. Three profiles (local-dev / local-distributed / enterprise)
  using different provisioner backends. Scaffold + ops is the control plane.
  Case zero: ops manages its own lifecycle after bootstrap.
- **#118** — extract ops/service library module. ops/app is a Quarkus application
  that can't be consumed as a dependency. Extract everything into ops/service
  (plain JAR) so Scaffold can embed ops. This is the critical path blocker for
  scaffold#53 and the deployment profile model.
- **#119** — enable GraphQL generation for ops @McpDomain APIs. Depends on #118.

Key insight from the session: the library/standalone split applies to all
platform services (ops, qhorus, claudony). Each needs an embeddable library
and a thin standalone wrapper. Scaffold embeds the libraries for local-dev;
ops provisions the containers for distributed deployment.

Qhorus mesh stays in the qhorus repo. Its job: make mesh/ production-ready
as a container image. Ops handles deployment orchestration. The distributed
mesh clustering/partitioning is a qhorus engineering problem, not a deployment
problem.
