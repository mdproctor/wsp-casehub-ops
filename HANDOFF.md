# Handoff — casehub-ops

## Active Work

**Branch:** `issue-150-multi-env-deployment`
**Issues:** #150 (epic), #151, #152, #153
**State:** Branch created, .plan scaffolded. Ready to brainstorm and plan.

## What to Do

Run `work continue` to pick up. The next step is brainstorming the design for #150 — specifically the environment abstraction that lets the same topology YAML deploy to local, Kubernetes, and Podman without changes.

## Context

**The goal:** Same topology YAML → different target environments. The `NodeProvisioner` SPI is the extension point — each environment gets its own implementation. The topology YAML stays environment-agnostic; environment selection happens at deploy time via configuration.

**What exists:**
- `container/` module — Podman REST API support for app containers, databases, networks, volumes. Foundation for #153.
- `ClaudonyNodeProvisioner` — proven across 6 acceptance tests (just landed). Handles `casehub_application` and `claudony_app` node types via the claudony fleet API.
- `NodeProvisioner` SPI — the desiredstate runtime's extension point. `handledTypes()`, `provision()`, `deprovision()`, `resyncInterval()`.
- `ServiceCluster` / `ServiceProcess` — test harness for launching real service instances.

**The three children:**
- **#151** Environment abstraction layer — how topology tests select a provisioner (config property? profile? annotation?). This unblocks #152 and #153.
- **#152** Kubernetes provisioner — deploy topology nodes to k8s via k3d/kind. New `NodeProvisioner` implementation.
- **#153** Podman provisioner — deploy topology nodes to podman pods. New `NodeProvisioner` implementation, building on the existing `container/` module.

**Key design question:** How does environment selection work? Options include a `target-environment` config property in the topology YAML, a test profile (`-Pintegration-k8s`), or a system property. The brainstorm should explore these.

## Recently Completed

**Branch `issue-194-acceptance-test-retrofit`** landed on main (2026-10-26):
- #194-#199: Six acceptance tests across L02, L03, L08, L10, L12, L14
- Each proves its topology level works end-to-end (not just deploys)
- Tutorial-style console output for learning content
