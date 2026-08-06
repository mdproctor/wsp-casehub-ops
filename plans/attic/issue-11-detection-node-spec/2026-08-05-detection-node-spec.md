# DetectionNodeSpec Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #11 — design: DetectionNodeSpec — RAS situation registration as a deployment node type
**Issue group:** #11

**Goal:** Add DetectionNodeSpec as the 6th deployment node type, enabling declarative RAS situation definitions in YAML topology goals.

**Architecture:** DetectionNodeSpec is a record in `ops-api/deployment` that mirrors the serializable surface of `SituationDefinition`. It flows through the existing deployment SPI quad: GoalCompiler compiles it, ActualStateAdapter checks drift, NodeProvisioner provisions/deprovisions via a new `SituationRegistrar` interface extracted to `casehub-ras-api`.

**Tech Stack:** Java 21 records, Jackson polymorphic serialization, Quarkus CDI, casehub-ras-api, casehub-desiredstate-api

## Global Constraints

- `ops-api` gains a compile dependency on `casehub-ras-api`
- `SituationRegistrar` interface must be added to `casehub-ras-api` first (cross-repo)
- `SituationDefinitionRegistry` in `casehub-ras` must implement `SituationRegistrar` (cross-repo)
- All existing `DeploymentGoals` constructor call sites gain a 7th positional arg
- `DeploymentNodeSpec` sealed permits list must include `DetectionNodeSpec`
- `PlanStoreMapper` mixins must include `DetectionNodeSpec` for JSON serialization

---

### Task 1: Cross-repo — SituationRegistrar interface in casehub-ras

**Files:**
- Create: `casehub-ras/api/src/main/java/io/casehub/ras/api/SituationRegistrar.java`
- Modify: `casehub-ras/runtime/src/main/java/io/casehub/ras/runtime/SituationDefinitionRegistry.java`
- Test: `casehub-ras/runtime/src/test/java/io/casehub/ras/runtime/SituationRegistrarTest.java`

**Interfaces:**
- Produces: `SituationRegistrar` with `register(SituationRegistration)`, `deregister(String)`, `exists(String)` — consumed by Tasks 4 and 5

- [ ] **Step 1: Write the SituationRegistrar interface**

```java
package io.casehub.ras.api;

public interface SituationRegistrar {
    void register(SituationRegistration registration);
    void deregister(String situationId);
    boolean exists(String situationId);
}
```

Create at `casehub-ras/api/src/main/java/io/casehub/ras/api/SituationRegistrar.java`.

- [ ] **Step 2: Make SituationDefinitionRegistry implement SituationRegistrar**

Modify the class declaration:
```java
public class SituationDefinitionRegistry implements SituationRegistrar {
```

Add the `exists()` method:
```java
public boolean exists(String situationId) {
    return snapshot.situationIds().contains(situationId);
}
```

Add `@Override` annotations to existing `register()` and `deregister()` methods.

- [ ] **Step 3: Write the failing test**

```java
package io.casehub.ras.runtime;

import io.casehub.ras.api.*;
import org.junit.jupiter.api.Test;
import java.time.Duration;
import java.util.*;
import static org.assertj.core.api.Assertions.*;

class SituationRegistrarTest {

    @Test
    void registryImplementsSituationRegistrar() {
        SituationRegistrar registrar = createRegistrar();
        assertThat(registrar).isInstanceOf(SituationDefinitionRegistry.class);
    }

    @Test
    void registerMakesSituationExist() {
        SituationRegistrar registrar = createRegistrar();
        assertThat(registrar.exists("test.situation")).isFalse();

        registrar.register(testRegistration("test.situation"));
        assertThat(registrar.exists("test.situation")).isTrue();
    }

    @Test
    void deregisterRemovesSituation() {
        SituationRegistrar registrar = createRegistrar();
        registrar.register(testRegistration("test.situation"));
        assertThat(registrar.exists("test.situation")).isTrue();

        registrar.deregister("test.situation");
        assertThat(registrar.exists("test.situation")).isFalse();
    }

    @Test
    void deregisterNonExistentIsNoOp() {
        SituationRegistrar registrar = createRegistrar();
        assertThatNoException().isThrownBy(() ->
            registrar.deregister("nonexistent"));
    }

    @Test
    void registerDuplicateThrows() {
        SituationRegistrar registrar = createRegistrar();
        registrar.register(testRegistration("test.situation"));
        assertThatThrownBy(() ->
            registrar.register(testRegistration("test.situation")))
            .isInstanceOf(IllegalStateException.class)
            .hasMessageContaining("Duplicate");
    }

    private SituationRegistrar createRegistrar() {
        var ganglion = io.casehub.ras.testing.MockGanglion.withId("test-ganglion");
        return SituationDefinitionRegistry.forTesting(List.of(), List.of(ganglion));
    }

    private SituationRegistration testRegistration(String situationId) {
        return new SituationRegistration(
            new SituationDefinition(
                situationId,
                Set.of("test.event"),
                Duration.ofMinutes(5),
                null,
                new ChainMode.Streak("test-ganglion", 3),
                new TriggerAction.NotifyOnly(),
                new TriggerMode.FireOnce()),
            null);
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl api,runtime -f casehub-ras/pom.xml`
Expected: ALL PASS

- [ ] **Step 5: Install casehub-ras locally**

Run: `mvn --batch-mode install -DskipTests -f casehub-ras/pom.xml`

This publishes the updated `casehub-ras-api` JAR with `SituationRegistrar` to the local `.m2`, making it available for ops module compilation.

- [ ] **Step 6: Commit**

```bash
git -C casehub-ras add api/src/main/java/io/casehub/ras/api/SituationRegistrar.java
git -C casehub-ras add runtime/src/main/java/io/casehub/ras/runtime/SituationDefinitionRegistry.java
git -C casehub-ras add runtime/src/test/java/io/casehub/ras/runtime/SituationRegistrarTest.java
git -C casehub-ras commit -m "feat(#11): extract SituationRegistrar interface to ras-api

Refs casehubio/casehub-ops#11"
```

---

### Task 2: DetectionNodeSpec record in ops-api

**Files:**
- Create: `api/src/main/java/io/casehub/ops/api/deployment/DetectionNodeSpec.java`
- Modify: `api/src/main/java/io/casehub/ops/api/deployment/DeploymentNodeSpec.java` (sealed permits)
- Modify: `api/src/main/java/io/casehub/ops/api/deployment/DeploymentGoals.java` (add detections field)
- Modify: `api/src/main/java/io/casehub/ops/api/approval/PlanStoreMapper.java` (add mixins)
- Modify: `api/pom.xml` (add casehub-ras-api dependency)
- Test: `api/src/test/java/io/casehub/ops/api/deployment/DetectionNodeSpecTest.java`

**Interfaces:**
- Consumes: `ChainMode`, `TriggerAction`, `TriggerMode`, `SituationDefinition`, `SituationRegistration` from `casehub-ras-api`
- Produces: `DetectionNodeSpec` record — consumed by Tasks 3, 4, 5

- [ ] **Step 1: Add casehub-ras-api dependency to api/pom.xml**

Add to `<dependencies>` section:
```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-ras-api</artifactId>
</dependency>
```

Version is managed by parent POM.

- [ ] **Step 2: Write the failing test**

```java
package io.casehub.ops.api.deployment;

import io.casehub.ras.api.*;
import org.junit.jupiter.api.Test;
import java.time.Duration;
import java.util.Map;
import java.util.Set;
import static org.assertj.core.api.Assertions.*;

class DetectionNodeSpecTest {

    @Test
    void nodeIdReturnsSituationId() {
        var spec = testSpec("app.failure-detection");
        assertThat(spec.nodeId()).isEqualTo("app.failure-detection");
    }

    @Test
    void nodeTypeReturnsDetection() {
        var spec = testSpec("app.failure-detection");
        assertThat(spec.nodeType()).isEqualTo("detection");
    }

    @Test
    void toRegistrationPreservesAllFields() {
        var spec = testSpec("app.failure-detection");
        SituationRegistration reg = spec.toRegistration();

        SituationDefinition def = reg.definition();
        assertThat(def.situationId()).isEqualTo("app.failure-detection");
        assertThat(def.eventTypes()).containsExactlyInAnyOrder(
            "desiredstate.node.faulted", "desiredstate.node.recovered");
        assertThat(def.correlationWindow()).isEqualTo(Duration.ofMinutes(10));
        assertThat(def.chainMode()).isInstanceOf(ChainMode.Streak.class);
        assertThat(((ChainMode.Streak) def.chainMode()).ganglionId())
            .isEqualTo("node-fault");
        assertThat(((ChainMode.Streak) def.chainMode()).requiredCount())
            .isEqualTo(3);
        assertThat(def.triggerAction()).isInstanceOf(TriggerAction.CreateCase.class);
    }

    @Test
    void nullSituationIdThrows() {
        assertThatThrownBy(() -> new DetectionNodeSpec(
            null,
            Set.of("test.event"),
            Duration.ofMinutes(5),
            null,
            new ChainMode.Streak("g1", 3),
            new TriggerAction.NotifyOnly(),
            null
        )).isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void blankSituationIdThrows() {
        assertThatThrownBy(() -> new DetectionNodeSpec(
            "  ",
            Set.of("test.event"),
            Duration.ofMinutes(5),
            null,
            new ChainMode.Streak("g1", 3),
            new TriggerAction.NotifyOnly(),
            null
        )).isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void emptyEventTypesThrows() {
        assertThatThrownBy(() -> new DetectionNodeSpec(
            "test",
            Set.of(),
            Duration.ofMinutes(5),
            null,
            new ChainMode.Streak("g1", 3),
            new TriggerAction.NotifyOnly(),
            null
        )).isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void nullChainModeThrows() {
        assertThatThrownBy(() -> new DetectionNodeSpec(
            "test",
            Set.of("test.event"),
            Duration.ofMinutes(5),
            null,
            null,
            new TriggerAction.NotifyOnly(),
            null
        )).isInstanceOf(NullPointerException.class);
    }

    @Test
    void nullTriggerActionThrows() {
        assertThatThrownBy(() -> new DetectionNodeSpec(
            "test",
            Set.of("test.event"),
            Duration.ofMinutes(5),
            null,
            new ChainMode.Streak("g1", 3),
            null,
            null
        )).isInstanceOf(NullPointerException.class);
    }

    @Test
    void nullTriggerModeDefaultsToFireOnce() {
        var spec = new DetectionNodeSpec(
            "test",
            Set.of("test.event"),
            Duration.ofMinutes(5),
            null,
            new ChainMode.Streak("g1", 3),
            new TriggerAction.NotifyOnly(),
            null
        );
        SituationRegistration reg = spec.toRegistration();
        assertThat(reg.definition().triggerMode())
            .isInstanceOf(TriggerMode.FireOnce.class);
    }

    @Test
    void jsonRoundTrip() throws Exception {
        var mapper = new com.fasterxml.jackson.databind.ObjectMapper();
        mapper.findAndRegisterModules();
        var spec = testSpec("app.failure-detection");
        String json = mapper.writeValueAsString(spec);
        DetectionNodeSpec deserialized = mapper.readValue(json, DetectionNodeSpec.class);
        assertThat(deserialized).isEqualTo(spec);
    }

    @Test
    void valueSemantics() {
        var spec1 = testSpec("app.failure-detection");
        var spec2 = testSpec("app.failure-detection");
        assertThat(spec1).isEqualTo(spec2);
        assertThat(spec1.hashCode()).isEqualTo(spec2.hashCode());
    }

    static DetectionNodeSpec testSpec(String situationId) {
        return new DetectionNodeSpec(
            situationId,
            Set.of("desiredstate.node.faulted", "desiredstate.node.recovered"),
            Duration.ofMinutes(10),
            null,
            new ChainMode.Streak("node-fault", 3),
            new TriggerAction.CreateCase(
                new CaseTriggerConfig("ops", "incident-response", "1.0", Map.of())),
            new TriggerMode.FireOnce()
        );
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `mvn --batch-mode -o test -pl api`
Expected: FAIL — `DetectionNodeSpec` class does not exist

- [ ] **Step 4: Create DetectionNodeSpec**

```java
package io.casehub.ops.api.deployment;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
import io.casehub.ras.api.*;

import java.time.Duration;
import java.util.Objects;
import java.util.Set;

@JsonIgnoreProperties(ignoreUnknown = true)
public record DetectionNodeSpec(
        String situationId,
        Set<String> eventTypes,
        Duration correlationWindow,
        Duration eventBufferDelay,
        ChainMode chainMode,
        TriggerAction triggerAction,
        TriggerMode triggerMode
) implements DeploymentNodeSpec {

    public DetectionNodeSpec {
        if (situationId == null || situationId.isBlank()) {
            throw new IllegalArgumentException("situationId is required");
        }
        if (eventTypes == null || eventTypes.isEmpty()) {
            throw new IllegalArgumentException("eventTypes must not be empty");
        }
        eventTypes = Set.copyOf(eventTypes);
        Objects.requireNonNull(chainMode, "chainMode is required");
        Objects.requireNonNull(triggerAction, "triggerAction is required");
    }

    @Override
    public String nodeId() {
        return situationId;
    }

    @Override
    public String nodeType() {
        return "detection";
    }

    public SituationRegistration toRegistration() {
        return new SituationRegistration(
                new SituationDefinition(situationId, eventTypes,
                        correlationWindow, eventBufferDelay,
                        chainMode, triggerAction, triggerMode),
                null);
    }
}
```

- [ ] **Step 5: Update DeploymentNodeSpec sealed permits**

Change:
```java
public sealed interface DeploymentNodeSpec extends NodeSpec permits
                                                    AgentNodeSpec, ChannelNodeSpec, CaseTypeNodeSpec, TrustPolicyNodeSpec, EndpointNodeSpec {
```
To:
```java
public sealed interface DeploymentNodeSpec extends NodeSpec permits
                                                    AgentNodeSpec, ChannelNodeSpec, CaseTypeNodeSpec, TrustPolicyNodeSpec, EndpointNodeSpec, DetectionNodeSpec {
```

- [ ] **Step 6: Update DeploymentGoals**

Add `detections` field between `endpoints` and `adaptations`:
```java
public record DeploymentGoals(
        List<GoalEntry<AgentNodeSpec>> agents,
        List<GoalEntry<ChannelNodeSpec>> channels,
        List<GoalEntry<CaseTypeNodeSpec>> caseTypes,
        List<GoalEntry<TrustPolicyNodeSpec>> trust,
        List<GoalEntry<EndpointNodeSpec>> endpoints,
        List<GoalEntry<DetectionNodeSpec>> detections,
        List<AdaptationRuleSpec> adaptations
) {
    public DeploymentGoals {
        agents = agents != null ? List.copyOf(agents) : List.of();
        channels = channels != null ? List.copyOf(channels) : List.of();
        caseTypes = caseTypes != null ? List.copyOf(caseTypes) : List.of();
        trust = trust != null ? List.copyOf(trust) : List.of();
        endpoints = endpoints != null ? List.copyOf(endpoints) : List.of();
        detections = detections != null ? List.copyOf(detections) : List.of();
        adaptations = adaptations != null ? List.copyOf(adaptations) : List.of();
    }
}
```

- [ ] **Step 7: Update PlanStoreMapper mixins**

Add to `NodeSpecMixin` (after endpoint line):
```java
@JsonSubTypes.Type(value = DetectionNodeSpec.class, name = "deployment-detection"),
```

Add to `DeploymentNodeSpecMixin` (after endpoint line):
```java
@JsonSubTypes.Type(value = DetectionNodeSpec.class, name = "detection"),
```

Add import: `import io.casehub.ops.api.deployment.DetectionNodeSpec;`

- [ ] **Step 8: Fix all DeploymentGoals constructor call sites**

Every existing call to `new DeploymentGoals(agents, channels, caseTypes, trust, endpoints, adaptations)` must gain a `List.of()` for `detections` as the 6th positional arg (before `adaptations`).

Search all files:
```
ide_search_text query="new DeploymentGoals(" filePattern="*.java"
```

Fix each call site by inserting `List.of(),` before the final `adaptations` arg.

- [ ] **Step 9: Run tests to verify they pass**

Run: `mvn --batch-mode -o test -pl api`
Expected: ALL PASS

- [ ] **Step 10: Commit**

```bash
git add api/pom.xml
git add api/src/main/java/io/casehub/ops/api/deployment/DetectionNodeSpec.java
git add api/src/main/java/io/casehub/ops/api/deployment/DeploymentNodeSpec.java
git add api/src/main/java/io/casehub/ops/api/deployment/DeploymentGoals.java
git add api/src/main/java/io/casehub/ops/api/approval/PlanStoreMapper.java
git add api/src/test/java/io/casehub/ops/api/deployment/DetectionNodeSpecTest.java
git commit -m "feat(#11): add DetectionNodeSpec — 6th deployment node type

Refs #11"
```

---

### Task 3: DeploymentGoalCompiler wiring

**Files:**
- Modify: `deployment/src/main/java/io/casehub/ops/deployment/DeploymentGoalCompiler.java`
- Modify: `deployment/src/test/java/io/casehub/ops/deployment/DeploymentGoalCompilerTest.java`

**Interfaces:**
- Consumes: `DetectionNodeSpec` from Task 2, `GoalEntry` from ops-api
- Produces: `DesiredNode` with `NodeType.of("detection")` in the compiled graph

- [ ] **Step 1: Write the failing test**

Add to `DeploymentGoalCompilerTest`:
```java
@Test
void compilesDetectionNode() {
    var detection = DetectionNodeSpecTest.testSpec("app.repeated-failure");
    var goals = new DeploymentGoals(
            List.of(), List.of(), List.of(), List.of(), List.of(),
            List.of(new GoalEntry<>(detection, List.of())),
            List.of());

    DesiredStateGraph graph = ((CompilationResult.SingleGraph)
            compiler.compile(goals, factory)).graph();

    assertThat(graph.nodes()).hasSize(1);
    DesiredNode node = graph.nodes().get(NodeId.of("app.repeated-failure"));
    assertThat(node.id()).isEqualTo(NodeId.of("app.repeated-failure"));
    assertThat(node.type().value()).isEqualTo("detection");
    assertThat(node.spec()).isInstanceOf(DetectionNodeSpec.class);
}
```

Add import: `import io.casehub.ops.api.deployment.DetectionNodeSpec;`
Add import: `import io.casehub.ops.api.deployment.DetectionNodeSpecTest;` (test helper from Task 2)

Note: If `DetectionNodeSpecTest` is not visible from this test (different test classpath), inline the `testSpec()` helper or use `io.casehub.ops.testing` fixtures.

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode -o test -pl deployment -Dtest=DeploymentGoalCompilerTest#compilesDetectionNode`
Expected: FAIL — no `compileEntries` call for detections

- [ ] **Step 3: Add the compiler line**

In `DeploymentGoalCompiler.compile()`, add after the endpoints line:
```java
compileEntries(goals.detections(), nodes, dependencies);
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn --batch-mode -o test -pl deployment -Dtest=DeploymentGoalCompilerTest`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git add deployment/src/main/java/io/casehub/ops/deployment/DeploymentGoalCompiler.java
git add deployment/src/test/java/io/casehub/ops/deployment/DeploymentGoalCompilerTest.java
git commit -m "feat(#11): wire DetectionNodeSpec in DeploymentGoalCompiler

Refs #11"
```

---

### Task 4: DetectionProvisionHandler and provisioner integration

**Files:**
- Create: `deployment/src/main/java/io/casehub/ops/deployment/handler/DetectionProvisionHandler.java`
- Modify: `deployment/src/main/java/io/casehub/ops/deployment/DeploymentNodeProvisioner.java`
- Test: `deployment/src/test/java/io/casehub/ops/deployment/handler/DetectionProvisionHandlerTest.java`
- Modify: `deployment/src/test/java/io/casehub/ops/deployment/DeploymentNodeProvisionerTest.java`

**Interfaces:**
- Consumes: `SituationRegistrar` from Task 1, `DetectionNodeSpec` from Task 2
- Produces: provision/deprovision of detection nodes through the deployment provisioner

- [ ] **Step 1: Write the failing handler test**

```java
package io.casehub.ops.deployment.handler;

import io.casehub.desiredstate.api.*;
import io.casehub.ops.api.deployment.DetectionNodeSpec;
import io.casehub.ras.api.*;
import org.junit.jupiter.api.Test;
import java.time.Duration;
import java.util.*;
import static org.assertj.core.api.Assertions.*;

class DetectionProvisionHandlerTest {

    @Test
    void provisionRegistersWithRegistrar() {
        var registrar = new StubSituationRegistrar();
        var handler = new DetectionProvisionHandler(registrar);
        var spec = testSpec("app.detect-failure");
        var graph = new io.casehub.desiredstate.runtime.DefaultDesiredStateGraphFactory()
                .of(List.of(), List.of());

        var result = handler.provision(spec,
            new ProvisionContext("tenant-1", graph));

        assertThat(result).isInstanceOf(ProvisionResult.Success.class);
        assertThat(registrar.registered).containsKey("app.detect-failure");
    }

    @Test
    void deprovisionDeregistersFromRegistrar() {
        var registrar = new StubSituationRegistrar();
        var handler = new DetectionProvisionHandler(registrar);
        var spec = testSpec("app.detect-failure");
        var graph = new io.casehub.desiredstate.runtime.DefaultDesiredStateGraphFactory()
                .of(List.of(), List.of());

        registrar.register(spec.toRegistration());
        handler.deprovision(spec, new DeprovisionContext("tenant-1", graph));

        assertThat(registrar.registered).doesNotContainKey("app.detect-failure");
    }

    private DetectionNodeSpec testSpec(String situationId) {
        return new DetectionNodeSpec(
            situationId,
            Set.of("test.event"),
            Duration.ofMinutes(5), null,
            new ChainMode.Streak("g1", 3),
            new TriggerAction.NotifyOnly(),
            new TriggerMode.FireOnce());
    }

    static class StubSituationRegistrar implements SituationRegistrar {
        final Map<String, SituationRegistration> registered = new HashMap<>();

        @Override
        public void register(SituationRegistration registration) {
            registered.put(registration.definition().situationId(), registration);
        }

        @Override
        public void deregister(String situationId) {
            registered.remove(situationId);
        }

        @Override
        public boolean exists(String situationId) {
            return registered.containsKey(situationId);
        }
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode -o test -pl deployment -Dtest=DetectionProvisionHandlerTest`
Expected: FAIL — `DetectionProvisionHandler` does not exist

- [ ] **Step 3: Create DetectionProvisionHandler**

```java
package io.casehub.ops.deployment.handler;

import io.casehub.desiredstate.api.*;
import io.casehub.ops.api.deployment.DetectionNodeSpec;
import io.casehub.ras.api.SituationRegistrar;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

@ApplicationScoped
public class DetectionProvisionHandler {

    private final SituationRegistrar registrar;

    @Inject
    public DetectionProvisionHandler(SituationRegistrar registrar) {
        this.registrar = registrar;
    }

    public ProvisionResult provision(DetectionNodeSpec spec, ProvisionContext context) {
        registrar.register(spec.toRegistration());
        return new ProvisionResult.Success();
    }

    public DeprovisionResult deprovision(DetectionNodeSpec spec, DeprovisionContext context) {
        registrar.deregister(spec.situationId());
        return new DeprovisionResult.Success();
    }
}
```

- [ ] **Step 4: Run handler test to verify it passes**

Run: `mvn --batch-mode -o test -pl deployment -Dtest=DetectionProvisionHandlerTest`
Expected: PASS

- [ ] **Step 5: Update DeploymentNodeProvisioner**

Add `DetectionProvisionHandler` field, constructor parameter, `handledTypes()` entry, and switch arms:

Constructor gains `DetectionProvisionHandler detectionHandler` parameter and `this.detectionHandler = detectionHandler;` assignment.

`handledTypes()` adds `NodeType.of("detection")`.

`doProvision()` switch gains: `case DetectionNodeSpec s -> detectionHandler.provision(s, context);`

`doDeprovision()` switch gains: `case DetectionNodeSpec s -> detectionHandler.deprovision(s, context);`

- [ ] **Step 6: Update DeploymentNodeProvisionerTest**

Add a test for detection provisioning. The existing test uses a builder or direct constructor — add `DetectionProvisionHandler` with a stub `SituationRegistrar` to the test setup, then add test methods for provision and deprovision of a `DetectionNodeSpec`.

- [ ] **Step 7: Run all deployment tests**

Run: `mvn --batch-mode -o test -pl deployment`
Expected: ALL PASS

- [ ] **Step 8: Commit**

```bash
git add deployment/src/main/java/io/casehub/ops/deployment/handler/DetectionProvisionHandler.java
git add deployment/src/main/java/io/casehub/ops/deployment/DeploymentNodeProvisioner.java
git add deployment/src/test/java/io/casehub/ops/deployment/handler/DetectionProvisionHandlerTest.java
git add deployment/src/test/java/io/casehub/ops/deployment/DeploymentNodeProvisionerTest.java
git commit -m "feat(#11): add DetectionProvisionHandler and wire into DeploymentNodeProvisioner

Refs #11"
```

---

### Task 5: DetectionDriftChecker and adapter integration

**Files:**
- Create: `deployment/src/main/java/io/casehub/ops/deployment/drift/DetectionDriftChecker.java`
- Modify: `deployment/src/main/java/io/casehub/ops/deployment/DeploymentActualStateAdapter.java` (handledTypes)
- Test: `deployment/src/test/java/io/casehub/ops/deployment/drift/DetectionDriftCheckerTest.java`
- Modify: `deployment/src/test/java/io/casehub/ops/deployment/DeploymentActualStateAdapterTest.java`

**Interfaces:**
- Consumes: `SituationRegistrar` from Task 1, `DetectionNodeSpec` from Task 2
- Produces: drift detection for detection nodes through the deployment adapter

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.ops.deployment.drift;

import io.casehub.desiredstate.api.NodeSpec;
import io.casehub.desiredstate.api.NodeStatus;
import io.casehub.ops.api.deployment.DetectionNodeSpec;
import io.casehub.ras.api.*;
import org.junit.jupiter.api.Test;
import java.time.Duration;
import java.util.*;
import static org.assertj.core.api.Assertions.*;

class DetectionDriftCheckerTest {

    @Test
    void nodeTypeIsDetection() {
        var checker = new DetectionDriftChecker(stubRegistrar(false));
        assertThat(checker.nodeType()).isEqualTo("detection");
    }

    @Test
    void returnsPresentWhenSituationExists() {
        var checker = new DetectionDriftChecker(stubRegistrar(true));
        var spec = testSpec("test.situation");
        assertThat(checker.check(spec, "tenant-1")).isEqualTo(NodeStatus.PRESENT);
    }

    @Test
    void returnsAbsentWhenSituationDoesNotExist() {
        var checker = new DetectionDriftChecker(stubRegistrar(false));
        var spec = testSpec("test.situation");
        assertThat(checker.check(spec, "tenant-1")).isEqualTo(NodeStatus.ABSENT);
    }

    @Test
    void returnsUnknownForNonDetectionSpec() {
        var checker = new DetectionDriftChecker(stubRegistrar(true));
        NodeSpec otherSpec = new io.casehub.ops.api.deployment.TrustPolicyNodeSpec(
            "policy", 0.5, 3, 0.1, 0.2, null, false);
        assertThat(checker.check(otherSpec, "tenant-1")).isEqualTo(NodeStatus.UNKNOWN);
    }

    private SituationRegistrar stubRegistrar(boolean exists) {
        return new SituationRegistrar() {
            @Override public void register(SituationRegistration r) {}
            @Override public void deregister(String id) {}
            @Override public boolean exists(String id) { return exists; }
        };
    }

    private DetectionNodeSpec testSpec(String situationId) {
        return new DetectionNodeSpec(
            situationId,
            Set.of("test.event"),
            Duration.ofMinutes(5), null,
            new ChainMode.Streak("g1", 3),
            new TriggerAction.NotifyOnly(),
            new TriggerMode.FireOnce());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode -o test -pl deployment -Dtest=DetectionDriftCheckerTest`
Expected: FAIL — `DetectionDriftChecker` does not exist

- [ ] **Step 3: Create DetectionDriftChecker**

```java
package io.casehub.ops.deployment.drift;

import io.casehub.desiredstate.api.NodeSpec;
import io.casehub.desiredstate.api.NodeStatus;
import io.casehub.ops.api.deployment.DetectionNodeSpec;
import io.casehub.ops.api.deployment.NodeDriftChecker;
import io.casehub.ras.api.SituationRegistrar;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

@ApplicationScoped
public class DetectionDriftChecker implements NodeDriftChecker {

    private final SituationRegistrar registrar;

    @Inject
    public DetectionDriftChecker(SituationRegistrar registrar) {
        this.registrar = registrar;
    }

    @Override
    public String nodeType() {
        return "detection";
    }

    @Override
    public NodeStatus check(NodeSpec spec, String tenancyId) {
        if (!(spec instanceof DetectionNodeSpec detection)) {
            return NodeStatus.UNKNOWN;
        }
        return registrar.exists(detection.situationId())
                ? NodeStatus.PRESENT
                : NodeStatus.ABSENT;
    }
}
```

- [ ] **Step 4: Update DeploymentActualStateAdapter.handledTypes()**

Add `NodeType.of("detection")` to the `handledTypes()` set.

- [ ] **Step 5: Run drift checker test**

Run: `mvn --batch-mode -o test -pl deployment -Dtest=DetectionDriftCheckerTest`
Expected: PASS

- [ ] **Step 6: Update DeploymentActualStateAdapterTest**

Add a test verifying that detection nodes flow through the adapter with the stub drift checker pattern already used for other types.

- [ ] **Step 7: Run all deployment tests**

Run: `mvn --batch-mode -o test -pl deployment`
Expected: ALL PASS

- [ ] **Step 8: Commit**

```bash
git add deployment/src/main/java/io/casehub/ops/deployment/drift/DetectionDriftChecker.java
git add deployment/src/main/java/io/casehub/ops/deployment/DeploymentActualStateAdapter.java
git add deployment/src/test/java/io/casehub/ops/deployment/drift/DetectionDriftCheckerTest.java
git add deployment/src/test/java/io/casehub/ops/deployment/DeploymentActualStateAdapterTest.java
git commit -m "feat(#11): add DetectionDriftChecker and wire into DeploymentActualStateAdapter

Refs #11"
```

---

### Task 6: Full build verification and dead dependency cleanup

**Files:**
- Modify: `app/pom.xml` (remove dead `casehub-engine-blackboard` dependency)

- [ ] **Step 1: Run full build**

Run: `mvn --batch-mode install`
Expected: ALL PASS across all modules

- [ ] **Step 2: Remove dead casehub-engine-blackboard dependency**

Remove from `app/pom.xml`:
```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-engine-blackboard</artifactId>
    <version>${project.version}</version>
</dependency>
```

- [ ] **Step 3: Run app tests**

Run: `mvn --batch-mode -o test -pl app`
Expected: ALL PASS (blackboard was already dead)

- [ ] **Step 4: Commit**

```bash
git add app/pom.xml
git commit -m "chore(#11): remove dead casehub-engine-blackboard dependency from app

Refs #11"
```
