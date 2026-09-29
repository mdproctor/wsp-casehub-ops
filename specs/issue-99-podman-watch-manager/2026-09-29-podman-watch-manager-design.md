# PodmanWatchManager — Active Podman Event Stream Subscription

**Issue:** casehubio/casehub-ops#99
**Branch:** issue-99-podman-watch-manager
**Date:** 2026-09-29

## Overview

Adds active Podman event stream subscription to the app module, following the K8sWatchManager pattern. Subscribes to Podman's `/events` NDJSON endpoint, translates container lifecycle events to StateEvents, and feeds them into `ContainerEventSource.emit()`. Also adds the `PodmanClient.events()` streaming method to the container module.

## Architecture

### PodmanClient.events()

New streaming method on the existing `PodmanClient` in the container module:

```java
public Multi<JsonObject> events(String type)
```

Connects to the Podman REST API endpoint `/events?stream=true&type={type}` via the existing Vert.x WebClient over the Unix domain socket. Uses `BodyCodec.jsonStream(JsonParser.newParser().objectValueMode())` to parse the newline-delimited JSON stream. Returns a reactive `Multi<JsonObject>` — one per event.

The connection is long-lived. Reconnection on disconnect is the caller's responsibility (PodmanWatchManager uses Mutiny retry with backoff).

### PodmanWatchManager

`@ApplicationScoped` CDI bean in `io.casehub.ops.app.container`. Follows the K8sWatchManager pattern but simplified for Podman's single-socket model.

**Lifecycle:**
- `@Observes StartupEvent` → subscribes to `podmanClient.events("container")`
- `@PreDestroy` → cancels the Multi subscription via `Cancellable`

**Event translation:**

| Podman Action | NodeStatus | Detail |
|---|---|---|
| `die` | DRIFTED | `container died: exit code {N}` |
| `stop` | SUSPENDED | `container stopped` |
| `remove` | ABSENT | `container removed` |
| `start` | PRESENT | `container started` |
| `health_status: unhealthy` | DRIFTED | `health check failed` |
| `health_status: healthy` | PRESENT | `health check passed` |

Unknown actions are silently filtered (translateEvent returns null, filtered by `Objects::nonNull`).

**NodeId mapping:** `NodeId.of(containerName)` from the event's `Actor.Attributes.name` field. Consistent with `PodmanActualStateAdapter.extractContainerName()`.

**Reconnection:** `onFailure().retry().withBackOff(Duration.ofSeconds(1), Duration.ofSeconds(30)).indefinitely()` — exponential backoff, never gives up. Handles Podman restarts, socket permission changes, and transient connection failures.

**Emission:** `containerEventSource.emit(stateEvent)` — feeds the passive hot emitter created in #98.

### Component Diagram

```
Podman REST API (/events?stream=true&type=container)
    │
    ▼ (NDJSON over Unix socket)
PodmanClient.events("container") → Multi<JsonObject>
    │
    ▼ (translate + filter)
PodmanWatchManager
    │ .map(translateEvent).filter(nonNull)
    │ .onFailure().retry().withBackOff(1s, 30s)
    ▼
ContainerEventSource.emit(StateEvent)
    │
    ▼
Multi<StateEvent> stream() → ReconciliationLoop
```

## Files

| File | Module | Purpose |
|---|---|---|
| `PodmanClient.java` (edit) | container | Add `events(String type)` streaming method |
| `PodmanWatchManager.java` (create) | app | Active Podman event stream subscriber |
| `PodmanClientTest.java` (edit) | container (test) | Add events() test |
| `ContainerReconciliationIntegrationTest.java` (edit) | container (test) | Add events()/emitEvent() to SimulatingPodmanClient |
| `PodmanWatchManagerTest.java` (create) | app (test) | Event translation + lifecycle tests |

## Test Plan

### PodmanClient — events() method

- Verify `SimulatingPodmanClient.events()` returns a Multi that emits JsonObjects pushed via `emitEvent()`

### PodmanWatchManagerTest

- **Event translation:** For each Podman action (die, stop, remove, start, health_status), verify the correct StateEvent is emitted to ContainerEventSource
- **Container name → NodeId:** Verify `NodeId.of(containerName)` mapping
- **Unknown action filtering:** Verify unrecognized actions (create, attach) are silently filtered
- **Health status events:** Verify unhealthy → DRIFTED, healthy → PRESENT
- **Die with exit code:** Verify exit code is included in detail string

### Not in scope

- Label-based filtering (`managed-by=casehub-ops`) — identified in #98 decision review as a future improvement, not blocking
- Integration with PodmanNodeProvisioner for label setting — separate concern

## Constraints

- PodmanWatchManager lives in the app module — not the container domain module
- `PodmanClient.events()` lives in the container module — it's a client method, not an active subscription
- No label filtering for now — all container events on the system are translated (unknown NodeIds ignored by reconciliation loop)

## References

- `K8sWatchManager.java` — active event stream pattern reference
- `ContainerEventSource.java` (#98) — passive hot emitter that receives events
- `PodmanClient.java` — existing REST client, extended with streaming
- `PodmanActualStateAdapter.java:86-92` — container name → NodeId mapping
- Decision review R1-02 (#98) — domain/app boundary finding that created this issue
- casehubio/casehub-ops#99 — issue specification
- casehubio/casehub-ops#98 — parent issue (ContainerEventSource + ContainerFaultPolicy)
