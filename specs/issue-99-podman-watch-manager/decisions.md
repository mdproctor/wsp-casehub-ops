# Decisions — #99 PodmanWatchManager

## D1: Startup mechanism

**Choice:** Auto-start at CDI startup via @Observes StartupEvent
**Alternatives:**
- Explicit start call — more control but unnecessary ceremony for a single local socket
**Rationale:** Podman is local-only, no multi-cluster configuration needed. Subscribe immediately. @PreDestroy cancels the subscription.
**Trade-offs:** Cannot defer connection — will attempt to connect even if Podman isn't running (handled by retry backoff)
**Sources:** K8sWatchManager.java (explicit start pattern for comparison), Quarkus CDI lifecycle
**Exploration:** quick
**Status:** captured

## D2: Event translation scope

**Choice:** Full translation — all lifecycle actions including positive signals
**Alternatives:**
- Problem-only events — simpler but reconciliation loop can't confirm recovery via events
**Rationale:** Positive events (start, healthy) let the reconciliation loop confirm recovery after faults and update last-seen timestamps without waiting for the next poll.
**Trade-offs:** More events emitted, but the reconciliation loop already filters by NodeId
**Sources:** #98 event mapping table, K8sWatchManager.driftWatcher() (K8s only emits on MODIFIED/DELETED)
**Exploration:** quick
**Status:** captured
