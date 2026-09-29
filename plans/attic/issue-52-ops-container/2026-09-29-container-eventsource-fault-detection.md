# Container EventSource + Fault Detection Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #98 — container EventSource + health-driven fault detection
**Issue group:** #95, #96, #97, #98 (branch issue-52-ops-container)

**Goal:** Add passive hot emitter EventSource, standard ThresholdFaultPolicy, and ContainerReviewSpec to the container module — completing the SPI quad.

**Architecture:** Follows the established domain module pattern exactly. ContainerEventSource is a passive hot emitter (same as DeploymentEventSource, InfraEventSource). ContainerFaultPolicy delegates to ThresholdFaultPolicy with PROVISION_FAILED, tier(3) — same as all other domain fault policies. ContainerReviewSpec in the API module for review nodes.

**Tech Stack:** Java 22, Quarkus (CDI), Smallrye Mutiny (Multi), JUnit 5, AssertJ

## Global Constraints

- All production classes are `@ApplicationScoped` CDI beans
- Container module is a Jandex library — no external connections, no lifecycle management
- Test scope uses `casehub-desiredstate` and `casehub-desiredstate-testing` (never compile/runtime)
- API module is pure Java — no CDI, no Quarkus dependencies

---

## Batch 1: ContainerEventSource + ContainerFaultPolicy + ContainerReviewSpec

### Task 1: ContainerReviewSpec + ContainerEventSource

**Files:**
- Create: `api/src/main/java/io/casehub/ops/api/container/ContainerReviewSpec.java`
- Create: `container/src/main/java/io/casehub/ops/container/ContainerEventSource.java`
- Test: `api/src/test/java/io/casehub/ops/api/container/ContainerReviewSpecTest.java`
- Test: `container/src/test/java/io/casehub/ops/container/ContainerEventSourceTest.java`

**Interfaces:**
- Consumes: `NodeSpec`, `NodeType`, `NodeId`, `EventSource`, `StateEvent`, `NodeStatus` (all from desiredstate-api)
- Produces: `ContainerReviewSpec(NodeId faultedNode, String reason)` — used by Task 2. `ContainerEventSource` — CDI bean with `emit(StateEvent)`, `emitDrift(NodeId)`, `Multi<StateEvent> stream()`

- [ ] **Step 1: Write ContainerReviewSpec test**

```java
package io.casehub.ops.api.container;

import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.api.NodeType;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class ContainerReviewSpecTest {

    @Test
    void nodeType_returnsContainerReview() {
        var spec = new ContainerReviewSpec(NodeId.of("app-1"), "OOM killed");
        assertThat(spec.nodeType()).isEqualTo(NodeType.of("container-review"));
    }

    @Test
    void rejectsNullFaultedNode() {
        assertThatThrownBy(() -> new ContainerReviewSpec(null, "reason"))
            .isInstanceOf(NullPointerException.class);
    }

    @Test
    void rejectsNullReason() {
        assertThatThrownBy(() -> new ContainerReviewSpec(NodeId.of("app-1"), null))
            .isInstanceOf(NullPointerException.class);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode -o test -pl api -Dtest=ContainerReviewSpecTest`
Expected: FAIL — `ContainerReviewSpec` class does not exist

- [ ] **Step 3: Write ContainerReviewSpec implementation**

```java
package io.casehub.ops.api.container;

import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.api.NodeSpec;
import io.casehub.desiredstate.api.NodeType;

import java.util.Objects;

public record ContainerReviewSpec(NodeId faultedNode, String reason) implements NodeSpec {
    public ContainerReviewSpec {
        Objects.requireNonNull(faultedNode, "faultedNode");
        Objects.requireNonNull(reason, "reason");
    }

    @Override
    public NodeType nodeType() {
        return NodeType.of("container-review");
    }
}
```

- [ ] **Step 4: Run ContainerReviewSpec test to verify it passes**

Run: `mvn --batch-mode -o test -pl api -Dtest=ContainerReviewSpecTest`
Expected: PASS (3 tests)

- [ ] **Step 5: Write ContainerEventSource test**

```java
package io.casehub.ops.container;

import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.api.NodeStatus;
import io.casehub.desiredstate.api.StateEvent;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Duration;

import static org.assertj.core.api.Assertions.assertThat;

class ContainerEventSourceTest {

    private ContainerEventSource eventSource;

    @BeforeEach
    void setUp() {
        eventSource = new ContainerEventSource();
    }

    @Test
    void emit_pushes_to_stream() {
        var event = new StateEvent(NodeId.of("myapp"), NodeStatus.DRIFTED, "test event");
        eventSource.emit(event);

        var received = eventSource.stream()
            .select().first(1)
            .collect().asList()
            .await().atMost(Duration.ofSeconds(2));

        assertThat(received).containsExactly(event);
    }

    @Test
    void emitDrift_emits_drifted_status() {
        eventSource.emitDrift(NodeId.of("my-container"));

        var received = eventSource.stream()
            .select().first(1)
            .collect().asList()
            .await().atMost(Duration.ofSeconds(2));

        assertThat(received).hasSize(1);
        assertThat(received.get(0).node()).isEqualTo(NodeId.of("my-container"));
        assertThat(received.get(0).newStatus()).isEqualTo(NodeStatus.DRIFTED);
        assertThat(received.get(0).detail()).isEqualTo("drift detected");
    }

    @Test
    void stream_broadcasts_to_all_subscribers() {
        var sub1 = eventSource.stream()
            .select().first(1)
            .collect().asList()
            .subscribeAsCompletionStage();
        var sub2 = eventSource.stream()
            .select().first(1)
            .collect().asList()
            .subscribeAsCompletionStage();

        var event = new StateEvent(NodeId.of("myapp"), NodeStatus.PRESENT, "started");
        eventSource.emit(event);

        assertThat(sub1).succeedsWithin(Duration.ofSeconds(2))
            .asList().containsExactly(event);
        assertThat(sub2).succeedsWithin(Duration.ofSeconds(2))
            .asList().containsExactly(event);
    }
}
```

- [ ] **Step 6: Run test to verify it fails**

Run: `mvn --batch-mode -o test -pl container -Dtest=ContainerEventSourceTest`
Expected: FAIL — `ContainerEventSource` class does not exist

- [ ] **Step 7: Write ContainerEventSource implementation**

```java
package io.casehub.ops.container;

import io.casehub.desiredstate.api.EventSource;
import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.api.NodeStatus;
import io.casehub.desiredstate.api.StateEvent;
import io.smallrye.mutiny.Multi;
import io.smallrye.mutiny.subscription.BackPressureStrategy;
import io.smallrye.mutiny.subscription.MultiEmitter;
import jakarta.enterprise.context.ApplicationScoped;

@ApplicationScoped
public class ContainerEventSource implements EventSource {

    private final Multi<StateEvent> stream;
    private volatile MultiEmitter<? super StateEvent> emitter;

    public ContainerEventSource() {
        this.stream = Multi.createFrom()
            .<StateEvent>emitter(e -> this.emitter = e, BackPressureStrategy.BUFFER)
            .broadcast().toAllSubscribers();
    }

    @Override
    public Multi<StateEvent> stream() {
        return stream;
    }

    public void emit(StateEvent event) {
        var e = this.emitter;
        if (e != null) {
            e.emit(event);
        }
    }

    public void emitDrift(NodeId nodeId) {
        emit(new StateEvent(nodeId, NodeStatus.DRIFTED, "drift detected"));
    }
}
```

- [ ] **Step 8: Run ContainerEventSource test to verify it passes**

Run: `mvn --batch-mode -o test -pl container -Dtest=ContainerEventSourceTest`
Expected: PASS (3 tests)

- [ ] **Step 9: Commit**

```bash
git add api/src/main/java/io/casehub/ops/api/container/ContainerReviewSpec.java api/src/test/java/io/casehub/ops/api/container/ContainerReviewSpecTest.java container/src/main/java/io/casehub/ops/container/ContainerEventSource.java container/src/test/java/io/casehub/ops/container/ContainerEventSourceTest.java
git commit -m "feat(#98): ContainerReviewSpec + ContainerEventSource — review spec and passive hot emitter"
```

### Task 2: ContainerFaultPolicy

**Files:**
- Create: `container/src/main/java/io/casehub/ops/container/ContainerFaultPolicy.java`
- Modify: `container/pom.xml` (add casehub-ops-api dependency)
- Test: `container/src/test/java/io/casehub/ops/container/ContainerFaultPolicyTest.java`

**Interfaces:**
- Consumes: `FaultPolicy`, `ThresholdFaultPolicy`, `FaultPolicy.addReviewNode()`, `ContainerReviewSpec` (Task 1), `ContainerNodeTypes`, `FaultEvent`, `FaultType`
- Produces: `ContainerFaultPolicy` — CDI bean, `onFault()` returns graph mutations

- [ ] **Step 1: Add casehub-ops-api dependency to container/pom.xml**

Add after the `casehub-desiredstate-api` dependency:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-ops-api</artifactId>
</dependency>
```

- [ ] **Step 2: Write the failing test**

```java
package io.casehub.ops.container;

import io.casehub.desiredstate.api.ActualState;
import io.casehub.desiredstate.api.DesiredNode;
import io.casehub.desiredstate.api.DesiredStateGraph;
import io.casehub.desiredstate.api.FaultEvent;
import io.casehub.desiredstate.api.FaultType;
import io.casehub.desiredstate.api.HumanGating;
import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.api.NodeStatus;
import io.casehub.desiredstate.runtime.DefaultDesiredStateGraphFactory;
import io.casehub.ops.api.container.ContainerReviewSpec;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class ContainerFaultPolicyTest {

    private static final String TENANCY = "tenant-1";

    private ContainerFaultPolicy policy;
    private DefaultDesiredStateGraphFactory graphFactory;

    @BeforeEach
    void setUp() {
        policy = new ContainerFaultPolicy();
        graphFactory = new DefaultDesiredStateGraphFactory();
    }

    @Test
    void provision_failed_no_escalation_below_threshold() {
        var appNode = appContainer("myapp");
        var graph = graphFactory.of(List.of(appNode), List.of());
        var actual = new ActualState(Map.of(NodeId.of("myapp"), NodeStatus.ABSENT));
        var fault = new FaultEvent(NodeId.of("myapp"), FaultType.PROVISION_FAILED, "connection refused");

        var mutations = policy.onFault(TENANCY, fault, graph, actual);
        assertThat(mutations).isEmpty();
    }

    @Test
    void provision_failed_escalates_at_threshold_three() {
        var appNode = appContainer("myapp");
        var graph = graphFactory.of(List.of(appNode), List.of());
        var actual = new ActualState(Map.of(NodeId.of("myapp"), NodeStatus.ABSENT));
        var fault = new FaultEvent(NodeId.of("myapp"), FaultType.PROVISION_FAILED, "image pull failed");

        policy.onFault(TENANCY, fault, graph, actual);
        policy.onFault(TENANCY, fault, graph, actual);
        var mutations = policy.onFault(TENANCY, fault, graph, actual);

        assertThat(mutations).isNotEmpty();
        assertThat(mutations).anySatisfy(m -> {
            var node = m.node();
            assertThat(node.spec()).isInstanceOf(ContainerReviewSpec.class);
            var spec = (ContainerReviewSpec) node.spec();
            assertThat(spec.faultedNode()).isEqualTo(NodeId.of("myapp"));
            assertThat(spec.reason()).isEqualTo("image pull failed");
        });
    }

    @Test
    void database_container_faults_are_handled() {
        var dbNode = new DesiredNode(NodeId.of("mydb"),
            new DatabaseContainerSpec("mydb", "postgres:16", "local"),
            HumanGating.NONE);
        var graph = graphFactory.of(List.of(dbNode), List.of());
        var actual = new ActualState(Map.of(NodeId.of("mydb"), NodeStatus.ABSENT));
        var fault = new FaultEvent(NodeId.of("mydb"), FaultType.PROVISION_FAILED, "fail");

        policy.onFault(TENANCY, fault, graph, actual);
        policy.onFault(TENANCY, fault, graph, actual);
        var mutations = policy.onFault(TENANCY, fault, graph, actual);

        assertThat(mutations).isNotEmpty();
    }

    @Test
    void ignores_review_node_faults() {
        var reviewNode = new DesiredNode(NodeId.of("container-review-myapp"),
            new ContainerReviewSpec(NodeId.of("myapp"), "test"),
            HumanGating.ALL);
        var graph = graphFactory.of(List.of(reviewNode), List.of());
        var actual = new ActualState(Map.of());
        var fault = new FaultEvent(NodeId.of("container-review-myapp"),
            FaultType.PROVISION_FAILED, "fail");

        var mutations = policy.onFault(TENANCY, fault, graph, actual);
        assertThat(mutations).isEmpty();
    }

    @Test
    void unhandled_fault_type_returns_empty() {
        var appNode = appContainer("myapp");
        var graph = graphFactory.of(List.of(appNode), List.of());
        var actual = new ActualState(Map.of());
        var fault = new FaultEvent(NodeId.of("myapp"), FaultType.DEPROVISION_FAILED, "fail");

        var mutations = policy.onFault(TENANCY, fault, graph, actual);
        assertThat(mutations).isEmpty();
    }

    private static DesiredNode appContainer(String name) {
        return new DesiredNode(NodeId.of(name),
            new AppContainerSpec(name, "img:latest", Map.of(), Map.of()),
            HumanGating.NONE);
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `mvn --batch-mode -o test -pl container -Dtest=ContainerFaultPolicyTest`
Expected: FAIL — `ContainerFaultPolicy` class does not exist

- [ ] **Step 4: Write ContainerFaultPolicy implementation**

```java
package io.casehub.ops.container;

import io.casehub.desiredstate.api.ActualState;
import io.casehub.desiredstate.api.DesiredNode;
import io.casehub.desiredstate.api.DesiredStateGraph;
import io.casehub.desiredstate.api.FaultEvent;
import io.casehub.desiredstate.api.FaultPolicy;
import io.casehub.desiredstate.api.FaultType;
import io.casehub.desiredstate.api.GraphMutation;
import io.casehub.desiredstate.api.NodeType;
import io.casehub.desiredstate.api.ThresholdFaultPolicy;
import io.casehub.ops.api.container.ContainerReviewSpec;
import jakarta.enterprise.context.ApplicationScoped;

import java.util.List;
import java.util.Set;

@ApplicationScoped
public class ContainerFaultPolicy implements FaultPolicy {

    private static final NodeType CONTAINER_REVIEW = NodeType.of("container-review");

    private final ThresholdFaultPolicy delegate = ThresholdFaultPolicy.builder()
        .faultTypes(Set.of(FaultType.PROVISION_FAILED))
        .nodeTypes(Set.of(ContainerNodeTypes.APP_CONTAINER, ContainerNodeTypes.DATABASE))
        .ignoreTypes(Set.of(CONTAINER_REVIEW))
        .tier(3, FaultPolicy.addReviewNode(
            (event, current) -> new ContainerReviewSpec(event.node(), event.detail())))
        .build();

    @Override
    public List<GraphMutation<DesiredNode>> onFault(String tenancyId, FaultEvent event,
                                        DesiredStateGraph current, ActualState actual) {
        return delegate.onFault(tenancyId, event, current, actual);
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `mvn --batch-mode -o test -pl container -Dtest=ContainerFaultPolicyTest`
Expected: PASS (5 tests)

- [ ] **Step 6: Run full build**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS — all modules compile and tests pass

- [ ] **Step 7: Commit**

```bash
git add container/pom.xml container/src/main/java/io/casehub/ops/container/ContainerFaultPolicy.java container/src/test/java/io/casehub/ops/container/ContainerFaultPolicyTest.java
git commit -m "feat(#98): ContainerFaultPolicy — standard tier(3) escalation with ContainerReviewSpec"
```

## References

- [2026-09-29-container-eventsource-fault-detection-design.md] — design spec (revised post decision review)
- [DeploymentEventSource.java] — hot emitter pattern reference
- [InfraEventSource.java] — second hot emitter reference
- [DeploymentFaultPolicy.java] — ThresholdFaultPolicy delegate pattern
- [InfraFaultPolicy.java] — second delegate reference
- [ThresholdFaultPolicy.java] — tier-based escalation API
- [DeploymentReviewSpec.java] — review spec record pattern
- [Decision review R1-02, R1-09, R1-10] — boundary and consistency findings
- [GitHub #98] — container EventSource + fault detection
