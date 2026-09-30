# PodmanWatchManager Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #99 — PodmanWatchManager — active Podman event stream subscription in app module
**Issue group:** #99

**Goal:** Add active Podman event stream subscription to the app module, translating container lifecycle events to StateEvents and feeding them into ContainerEventSource.

**Architecture:** PodmanClient.events() provides a reactive Multi<JsonObject> from Podman's /events NDJSON endpoint. PodmanWatchManager in the app module subscribes at CDI startup, translates events, and feeds ContainerEventSource.emit(). Follows K8sWatchManager pattern.

**Tech Stack:** Java 22, Quarkus (CDI), Smallrye Mutiny (Multi), Vert.x WebClient + JsonParser (NDJSON streaming), JUnit 5, AssertJ

## Global Constraints

- PodmanWatchManager lives in `app/` module — NOT the container domain module
- `PodmanClient.events()` lives in `container/` module — client method, not active subscription
- Container module is a Jandex library — no lifecycle management
- App module has Quarkus runtime — @Observes StartupEvent, @PreDestroy available
- Test scope for container module: `casehub-desiredstate` and `casehub-desiredstate-testing`

---

## Batch 1: PodmanClient streaming + PodmanWatchManager

### Task 1: PodmanClient.events() + SimulatingPodmanClient extension

**Files:**
- Modify: `container/src/main/java/io/casehub/ops/container/podman/PodmanClient.java`
- Modify: `container/src/test/java/io/casehub/ops/container/ContainerReconciliationIntegrationTest.java` (SimulatingPodmanClient)
- Test: `container/src/test/java/io/casehub/ops/container/podman/PodmanClientTest.java`

**Interfaces:**
- Consumes: Vert.x `WebClient`, `SocketAddress`, `JsonParser`, `BackPressureStrategy` (existing in PodmanClient)
- Produces: `Multi<JsonObject> events(String type)` — used by Task 2 (PodmanWatchManager). `SimulatingPodmanClient.emitEvent(JsonObject)` — used by Task 2 tests.

- [ ] **Step 1: Write the failing test**

Add to `PodmanClientTest.java`:

```java
@Test
void events_returnsMultiOfJsonObjects() {
    var sim = new ContainerReconciliationIntegrationTest.SimulatingPodmanClient();
    var event = new JsonObject()
        .put("Type", "container")
        .put("Action", "start")
        .put("Actor", new JsonObject()
            .put("Attributes", new JsonObject().put("name", "myapp")));

    var future = sim.events("container")
        .select().first(1)
        .collect().asList()
        .subscribeAsCompletionStage();

    sim.emitEvent(event);

    assertThat(future).succeedsWithin(Duration.ofSeconds(2))
        .asList().hasSize(1);
    assertThat(((JsonObject) future.join().get(0)).getString("Action")).isEqualTo("start");
}
```

Add imports:

```java
import io.vertx.core.json.JsonObject;
import java.time.Duration;
import static org.assertj.core.api.Assertions.assertThat;
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode -o test -pl container -Dtest=PodmanClientTest#events_returnsMultiOfJsonObjects`
Expected: FAIL — `events()` method and `emitEvent()` method do not exist

- [ ] **Step 3: Add events() to PodmanClient**

Add to `PodmanClient.java` after the existing `pullImage` method:

```java
public Multi<JsonObject> events(String type) {
    return Multi.createFrom().<JsonObject>emitter(emitter -> {
        JsonParser parser = JsonParser.newParser()
            .objectValueMode()
            .handler(event -> {
                if (event.type() == JsonEventType.VALUE) {
                    emitter.emit(event.objectValue());
                }
            })
            .exceptionHandler(emitter::fail);

        webClient.request(HttpMethod.GET, socketAddress, 80, "localhost",
                API_VERSION + "/events")
            .addQueryParam("stream", "true")
            .addQueryParam("type", type)
            .as(BodyCodec.jsonStream(parser))
            .send()
            .onFailure(emitter::fail);
    }, BackPressureStrategy.BUFFER);
}
```

Add imports to `PodmanClient.java`:

```java
import io.smallrye.mutiny.Multi;
import io.smallrye.mutiny.subscription.BackPressureStrategy;
import io.vertx.core.parsetools.JsonEventType;
import io.vertx.core.parsetools.JsonParser;
import io.vertx.ext.web.codec.BodyCodec;
```

- [ ] **Step 4: Add events() override and emitEvent() to SimulatingPodmanClient**

In `ContainerReconciliationIntegrationTest.SimulatingPodmanClient`, add:

```java
private volatile MultiEmitter<? super JsonObject> eventEmitter;
private final Multi<JsonObject> eventStream = Multi.createFrom()
    .<JsonObject>emitter(e -> this.eventEmitter = e, BackPressureStrategy.BUFFER)
    .broadcast().toAllSubscribers();

@Override
public Multi<JsonObject> events(String type) {
    return eventStream;
}

void emitEvent(JsonObject event) {
    var e = this.eventEmitter;
    if (e != null) e.emit(event);
}
```

Add imports to the test file:

```java
import io.smallrye.mutiny.Multi;
import io.smallrye.mutiny.subscription.BackPressureStrategy;
import io.smallrye.mutiny.subscription.MultiEmitter;
import io.vertx.core.json.JsonObject;
```

- [ ] **Step 5: Run test to verify it passes**

Run: `mvn --batch-mode -o test -pl container -Dtest=PodmanClientTest#events_returnsMultiOfJsonObjects`
Expected: PASS

- [ ] **Step 6: Run full container module tests**

Run: `mvn --batch-mode -o test -pl container`
Expected: All existing tests pass (SimulatingPodmanClient changes are backwards-compatible)

- [ ] **Step 7: Commit**

```bash
git add container/src/main/java/io/casehub/ops/container/podman/PodmanClient.java container/src/test/java/io/casehub/ops/container/ContainerReconciliationIntegrationTest.java container/src/test/java/io/casehub/ops/container/podman/PodmanClientTest.java
git commit -m "feat(#99): PodmanClient.events() — NDJSON streaming from Podman /events"
```

### Task 2: PodmanWatchManager

**Files:**
- Create: `app/src/main/java/io/casehub/ops/app/container/PodmanWatchManager.java`
- Test: `app/src/test/java/io/casehub/ops/app/container/PodmanWatchManagerTest.java`

**Interfaces:**
- Consumes: `PodmanClient.events(String type)` (Task 1), `ContainerEventSource.emit(StateEvent)` (#98), `NodeId`, `NodeStatus`, `StateEvent`
- Produces: `PodmanWatchManager` — CDI bean, auto-starts on Quarkus startup, feeds ContainerEventSource

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.ops.app.container;

import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.api.NodeStatus;
import io.casehub.desiredstate.api.StateEvent;
import io.casehub.ops.container.ContainerEventSource;
import io.casehub.ops.container.ContainerReconciliationIntegrationTest.SimulatingPodmanClient;
import io.vertx.core.json.JsonObject;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Duration;

import static org.assertj.core.api.Assertions.assertThat;

class PodmanWatchManagerTest {

    private SimulatingPodmanClient podmanClient;
    private ContainerEventSource eventSource;
    private PodmanWatchManager watchManager;

    @BeforeEach
    void setUp() {
        podmanClient = new SimulatingPodmanClient();
        eventSource = new ContainerEventSource();
        watchManager = new PodmanWatchManager(podmanClient, eventSource);
        watchManager.start(null);
    }

    @Test
    void die_event_emits_drifted() {
        var future = eventSource.stream()
            .select().first(1)
            .collect().asList()
            .subscribeAsCompletionStage();

        podmanClient.emitEvent(podmanEvent("die", "myapp",
            new JsonObject().put("exitCode", "137")));

        assertThat(future).succeedsWithin(Duration.ofSeconds(2))
            .asList().singleElement().satisfies(e -> {
                var event = (StateEvent) e;
                assertThat(event.node()).isEqualTo(NodeId.of("myapp"));
                assertThat(event.newStatus()).isEqualTo(NodeStatus.DRIFTED);
                assertThat(event.detail()).contains("exit code 137");
            });
    }

    @Test
    void stop_event_emits_suspended() {
        var future = eventSource.stream()
            .select().first(1)
            .collect().asList()
            .subscribeAsCompletionStage();

        podmanClient.emitEvent(podmanEvent("stop", "myapp"));

        assertThat(future).succeedsWithin(Duration.ofSeconds(2))
            .asList().singleElement().satisfies(e -> {
                var event = (StateEvent) e;
                assertThat(event.newStatus()).isEqualTo(NodeStatus.SUSPENDED);
            });
    }

    @Test
    void remove_event_emits_absent() {
        var future = eventSource.stream()
            .select().first(1)
            .collect().asList()
            .subscribeAsCompletionStage();

        podmanClient.emitEvent(podmanEvent("remove", "myapp"));

        assertThat(future).succeedsWithin(Duration.ofSeconds(2))
            .asList().singleElement().satisfies(e -> {
                var event = (StateEvent) e;
                assertThat(event.newStatus()).isEqualTo(NodeStatus.ABSENT);
            });
    }

    @Test
    void start_event_emits_present() {
        var future = eventSource.stream()
            .select().first(1)
            .collect().asList()
            .subscribeAsCompletionStage();

        podmanClient.emitEvent(podmanEvent("start", "myapp"));

        assertThat(future).succeedsWithin(Duration.ofSeconds(2))
            .asList().singleElement().satisfies(e -> {
                var event = (StateEvent) e;
                assertThat(event.newStatus()).isEqualTo(NodeStatus.PRESENT);
            });
    }

    @Test
    void health_status_unhealthy_emits_drifted() {
        var future = eventSource.stream()
            .select().first(1)
            .collect().asList()
            .subscribeAsCompletionStage();

        podmanClient.emitEvent(podmanEvent("health_status", "myapp",
            new JsonObject().put("healthstatus", "unhealthy")));

        assertThat(future).succeedsWithin(Duration.ofSeconds(2))
            .asList().singleElement().satisfies(e -> {
                var event = (StateEvent) e;
                assertThat(event.newStatus()).isEqualTo(NodeStatus.DRIFTED);
                assertThat(event.detail()).isEqualTo("health check failed");
            });
    }

    @Test
    void health_status_healthy_emits_present() {
        var future = eventSource.stream()
            .select().first(1)
            .collect().asList()
            .subscribeAsCompletionStage();

        podmanClient.emitEvent(podmanEvent("health_status", "myapp",
            new JsonObject().put("healthstatus", "healthy")));

        assertThat(future).succeedsWithin(Duration.ofSeconds(2))
            .asList().singleElement().satisfies(e -> {
                var event = (StateEvent) e;
                assertThat(event.newStatus()).isEqualTo(NodeStatus.PRESENT);
                assertThat(event.detail()).isEqualTo("health check passed");
            });
    }

    @Test
    void unknown_action_is_filtered() {
        var future = eventSource.stream()
            .select().first(1)
            .collect().asList()
            .subscribeAsCompletionStage();

        podmanClient.emitEvent(podmanEvent("create", "myapp"));
        podmanClient.emitEvent(podmanEvent("start", "myapp"));

        assertThat(future).succeedsWithin(Duration.ofSeconds(2))
            .asList().singleElement().satisfies(e -> {
                var event = (StateEvent) e;
                assertThat(event.newStatus()).isEqualTo(NodeStatus.PRESENT);
            });
    }

    @Test
    void shutdown_cancels_subscription() {
        watchManager.shutdown();
        assertThat(watchManager.isRunning()).isFalse();
    }

    private static JsonObject podmanEvent(String action, String containerName) {
        return podmanEvent(action, containerName, new JsonObject());
    }

    private static JsonObject podmanEvent(String action, String containerName, JsonObject extraAttrs) {
        var attrs = new JsonObject().put("name", containerName).mergeIn(extraAttrs);
        return new JsonObject()
            .put("Type", "container")
            .put("Action", action)
            .put("Actor", new JsonObject()
                .put("Attributes", attrs));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode -o test -pl app -Dtest=PodmanWatchManagerTest`
Expected: FAIL — `PodmanWatchManager` class does not exist

- [ ] **Step 3: Write PodmanWatchManager implementation**

```java
package io.casehub.ops.app.container;

import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.api.NodeStatus;
import io.casehub.desiredstate.api.StateEvent;
import io.casehub.ops.container.ContainerEventSource;
import io.casehub.ops.container.podman.PodmanClient;
import io.quarkus.runtime.StartupEvent;
import io.smallrye.mutiny.subscription.Cancellable;
import io.vertx.core.json.JsonObject;
import jakarta.annotation.PreDestroy;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.inject.Inject;

import java.time.Duration;
import java.util.Objects;
import java.util.logging.Logger;

@ApplicationScoped
public class PodmanWatchManager {

    private static final Logger LOG = Logger.getLogger(PodmanWatchManager.class.getName());

    private final PodmanClient podmanClient;
    private final ContainerEventSource eventSource;
    private volatile Cancellable subscription;

    @Inject
    public PodmanWatchManager(PodmanClient podmanClient, ContainerEventSource eventSource) {
        this.podmanClient = podmanClient;
        this.eventSource = eventSource;
    }

    void start(@Observes StartupEvent event) {
        subscription = podmanClient.events("container")
            .map(PodmanWatchManager::translateEvent)
            .filter(Objects::nonNull)
            .onFailure().invoke(t -> LOG.warning("Podman event stream disconnected: " + t.getMessage()))
            .onFailure().retry()
                .withBackOff(Duration.ofSeconds(1), Duration.ofSeconds(30))
                .indefinitely()
            .subscribe().with(
                stateEvent -> eventSource.emit(stateEvent),
                t -> LOG.severe("Podman event stream failed permanently: " + t.getMessage())
            );
        LOG.info("Podman event stream subscription started");
    }

    @PreDestroy
    void shutdown() {
        var s = this.subscription;
        if (s != null) {
            s.cancel();
            this.subscription = null;
            LOG.info("Podman event stream subscription cancelled");
        }
    }

    public boolean isRunning() {
        return subscription != null;
    }

    static StateEvent translateEvent(JsonObject event) {
        String action = event.getString("Action");
        JsonObject actor = event.getJsonObject("Actor");
        if (actor == null) return null;
        JsonObject attrs = actor.getJsonObject("Attributes");
        if (attrs == null) return null;
        String name = attrs.getString("name");
        if (name == null) return null;

        NodeId nodeId = NodeId.of(name);

        return switch (action) {
            case "die" -> {
                String exitCode = attrs.getString("exitCode", "unknown");
                yield new StateEvent(nodeId, NodeStatus.DRIFTED,
                    "container died: exit code " + exitCode);
            }
            case "stop" -> new StateEvent(nodeId, NodeStatus.SUSPENDED, "container stopped");
            case "remove" -> new StateEvent(nodeId, NodeStatus.ABSENT, "container removed");
            case "start" -> new StateEvent(nodeId, NodeStatus.PRESENT, "container started");
            case "health_status" -> {
                String healthStatus = attrs.getString("healthstatus", "unknown");
                if ("unhealthy".equals(healthStatus)) {
                    yield new StateEvent(nodeId, NodeStatus.DRIFTED, "health check failed");
                } else if ("healthy".equals(healthStatus)) {
                    yield new StateEvent(nodeId, NodeStatus.PRESENT, "health check passed");
                }
                yield null;
            }
            default -> null;
        };
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode -o test -pl app -Dtest=PodmanWatchManagerTest`
Expected: PASS (8 tests)

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/io/casehub/ops/app/container/PodmanWatchManager.java app/src/test/java/io/casehub/ops/app/container/PodmanWatchManagerTest.java
git commit -m "feat(#99): PodmanWatchManager — active Podman event stream subscription in app module"
```

## References

- [2026-09-29-podman-watch-manager-design.md] — design spec this plan implements
- [K8sWatchManager.java] — active event stream pattern reference
- [KubernetesEventSource.java] — K8s passive emitter (app module)
- [ContainerEventSource.java] (#98) — passive hot emitter that receives events
- [PodmanClient.java] — existing REST client, extended with streaming
- [PodmanActualStateAdapter.java:86-92] — container name → NodeId mapping
- [GitHub #99] — PodmanWatchManager issue
- [GitHub #98] — parent issue (ContainerEventSource + ContainerFaultPolicy)
