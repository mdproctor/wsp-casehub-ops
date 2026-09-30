# Design Journal — issue-99-podman-watch-manager

## 2026-09-29 — PodmanClient.events() streaming + deferred PodmanWatchManager

Added `Multi<JsonObject> events(String type)` to PodmanClient — NDJSON streaming
from Podman's `/events` endpoint using Vert.x `JsonParser` in `objectValueMode`.
Extended `SimulatingPodmanClient` with `events()`/`emitEvent()` for test feeding.

PodmanWatchManager (app module) deferred — the app module has pre-existing
compilation errors (`ApplicationEntity.findById` doesn't exist; it's a plain
JPA entity, not Panache). Filed #100 to fix the app module compilation.
Once #100 is resolved, the plan's Task 2 has the full PodmanWatchManager
implementation ready to execute.

Design validated by #98's decision review (R1-02): active stream subscriptions
belong in the app module, not domain modules. K8sWatchManager is the pattern reference.
