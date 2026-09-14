# Summarisation-Enhanced RAS Detection Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** casehubio/casehub-ops#84 — feat: summarisation-enhanced RAS detection for deployment topologies
**Issue group:** #84

**Goal:** Wire a YAML-declared summarisation pipeline between desiredstate ReconciliationLoop CloudEvents and RAS, giving RAS altitude — detecting deployment phase transitions (DEGRADED, RECOVERING, HEALTHY) instead of counting individual NODE_FAULTED events.

**Architecture:** A single YAML pipeline consumes L1 desiredstate CloudEvents (`node.faulted`, `node.drifted`, `node.recovered`) via `type-prefix` matching. L2 `threshold-classify` classifies each event by fault type (PROVISION_FAILURE, DEPENDENCY_FAULT, etc.) or marks it as STABILISING (drift/recovery signals). L3 `phase-detect` tracks deployment health phase transitions. RAS situation definitions consume L3 phase events via `ExpressionRules` ganglia and `ChainMode` to detect sustained degradation.

**Tech Stack:** casehub-blocks-summarisation-yaml (YAML pipeline surface), casehub-ras-api (situation definitions), MVEL3 (expression evaluation), JUnit 5 + AssertJ (testing)

## Global Constraints

- No code changes to casehub-desiredstate (design spec §8)
- All new code in casehub-ops (app/ and api/ modules)
- YAML pipeline follows the `logistics-hub.yaml` reference pattern from blocks
- RAS situation definitions follow the `OpsMonitoringSituationDefinitionProvider` pattern
- Pipeline YAML discovered at `META-INF/summarisation/*.yaml` classpath convention
- Expression language: MVEL3 (via `casehub-platform-expression`, already on classpath)
- `threshold-classify` rules must be non-overlapping (all matching rules produce output)
- `phase-detect` emits only on state transitions (not on steady-state batches)
- `NodeFaultedData` has `faultType` (non-null); `NodeDriftedData`/`NodeRecoveredData` do not — this is the L2 discriminator

---

## Batch 1: Summarisation Pipeline Foundation

### Task 1: YAML Pipeline + Dependencies + CloudEvent Types

**Files:**
- Modify: `app/pom.xml` — add `casehub-blocks-summarisation-yaml` dependency
- Create: `api/src/main/java/io/casehub/ops/api/lifecycle/DeploymentSummarisationEventTypes.java`
- Create: `app/src/main/resources/META-INF/summarisation/deployment-monitoring.yaml`
- Test: `app/src/test/java/io/casehub/ops/app/lifecycle/summarisation/DeploymentMonitoringPipelineTest.java`

**Interfaces:**
- Consumes: `DesiredStateEventTypes` (L1 CloudEvent type URIs), `NodeFaultedData`/`NodeDriftedData`/`NodeRecoveredData` (payload schemas), `FaultType` (enum values for classification rules)
- Produces: L2 anomaly events (`io.casehub.ops.deployment.anomaly`) with `category` + `severity` fields; L3 phase events (`io.casehub.ops.deployment.phase`) with `from` + `to` + `phase` fields

- [ ] **Step 1: Write the pipeline validation test**

```java
package io.casehub.ops.app.lifecycle.summarisation;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import io.casehub.blocks.summarisation.yaml.*;
import io.casehub.blocks.summarisation.yaml.builtin.*;
import io.casehub.platform.api.expression.ExpressionEngine;
import io.casehub.platform.expression.MvelExpressionEngine;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.io.IOException;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class DeploymentMonitoringPipelineTest {

    static final ObjectMapper MAPPER = new ObjectMapper(new YAMLFactory());
    static final ExpressionEngine EXPR = new MvelExpressionEngine();

    PipelineDefinition definition;
    SummariserRegistry registry;

    @BeforeEach
    @SuppressWarnings("unchecked")
    void setUp() throws IOException {
        var yaml = getClass().getResourceAsStream(
                "/META-INF/summarisation/deployment-monitoring.yaml");
        definition = MAPPER.readValue(yaml, PipelineWrapper.class).pipeline();

        registry = new SummariserRegistry();
        registry.register("threshold-classify", (SummariserFactory)
                config -> ThresholdClassifySummariser.create(config, EXPR));
        registry.register("phase-detect", (SummariserFactory) config -> {
            var aggregateFields = definition.levels().stream()
                    .filter(l -> l.summariser().type().equals("phase-detect"))
                    .findFirst()
                    .map(LevelDefinition::aggregateFields)
                    .orElse(List.of());
            return PhaseDetectSummariser.create(config, aggregateFields);
        });
    }

    @Test
    void yamlParsesCorrectly() {
        assertThat(definition.name()).isEqualTo("deployment-monitoring");
        assertThat(definition.levels()).hasSize(2);
        assertThat(definition.levels().get(0).name()).isEqualTo("anomalies");
        assertThat(definition.levels().get(1).name()).isEqualTo("phases");
    }

    @Test
    void validationPasses() {
        var errors = new PipelineValidator().validate(definition, registry);
        var realErrors = errors.stream()
                .filter(e -> e.level() == PipelineValidator.ValidationError.Level.ERROR)
                .toList();
        assertThat(realErrors).isEmpty();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode -o test -pl app -Dtest=DeploymentMonitoringPipelineTest -DrunQuarkusTests`

Expected: Compilation error — missing dependency `casehub-blocks-summarisation-yaml`, missing YAML file

- [ ] **Step 3: Add summarisation-yaml dependency to app/pom.xml**

Add to `app/pom.xml` in the CaseHub foundation dependencies section (after `casehub-platform-expression`):

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-blocks-summarisation-yaml</artifactId>
</dependency>
```

- [ ] **Step 4: Create DeploymentSummarisationEventTypes**

```java
package io.casehub.ops.api.lifecycle;

public final class DeploymentSummarisationEventTypes {
    private DeploymentSummarisationEventTypes() {}

    public static final String ANOMALY = "io.casehub.ops.deployment.anomaly";
    public static final String PHASE = "io.casehub.ops.deployment.phase";
}
```

- [ ] **Step 5: Create deployment-monitoring.yaml pipeline**

Create `app/src/main/resources/META-INF/summarisation/deployment-monitoring.yaml`:

```yaml
pipeline:
  name: deployment-monitoring
  source:
    type-prefix: io.casehub.desiredstate.node.

  levels:
    - name: anomalies
      grouping:
        type: windowed
        count: 10
      summariser:
        type: threshold-classify
        rules:
          - name: provision-failure
            when: "faultType == 'PROVISION_FAILED'"
            category: PROVISION_FAILURE
            severity: CRITICAL
          - name: dependency-fault
            when: "faultType == 'DEPENDENCY_UNAVAILABLE'"
            category: DEPENDENCY_FAULT
            severity: CRITICAL
          - name: node-destroyed
            when: "faultType == 'NODE_DESTROYED'"
            category: NODE_DESTROYED
            severity: CRITICAL
          - name: deprovision-failure
            when: "faultType == 'DEPROVISION_FAILED'"
            category: DEPROVISION_FAILURE
            severity: HIGH
          - name: degradation
            when: "faultType == 'NODE_DEGRADED'"
            category: DEGRADATION
            severity: HIGH
          - name: human-timeout
            when: "faultType == 'HUMAN_NODE_TIMEOUT'"
            category: HUMAN_TIMEOUT
            severity: MEDIUM
          - name: approval-rejected
            when: "faultType == 'APPROVAL_REJECTED'"
            category: APPROVAL_REJECTED
            severity: MEDIUM
          - name: stabilising
            when: "faultType == null"
            category: STABILISING
            severity: LOW
      emit:
        cloud-event-type: io.casehub.ops.deployment.anomaly

    - name: phases
      grouping:
        type: windowed
        age: 60000
        count: 20
      aggregate-fields:
        - severity
        - category
      summariser:
        type: phase-detect
        initial: HEALTHY
        states:
          - HEALTHY
          - DEGRADED
          - RECOVERING
        transitions:
          - from: HEALTHY
            to: DEGRADED
            count-field: severity
            count-value: CRITICAL
            op: ">="
            threshold: 2
          - from: DEGRADED
            to: RECOVERING
            count-field: severity
            count-value: CRITICAL
            op: "=="
            threshold: 0
          - from: RECOVERING
            to: HEALTHY
            count-field: category
            count-value: STABILISING
            op: ">="
            threshold: 3
      emit:
        cloud-event-type: io.casehub.ops.deployment.phase
```

- [ ] **Step 6: Run validation test to verify it passes**

Run: `mvn --batch-mode -o test -pl app -Dtest=DeploymentMonitoringPipelineTest -DrunQuarkusTests`

Expected: PASS — both `yamlParsesCorrectly` and `validationPasses`

- [ ] **Step 7: Add L2 classification behaviour test**

Add to `DeploymentMonitoringPipelineTest`:

```java
@Test
void classifiesProvisionFailure() {
    var pipeline = new PipelineCompiler().compile(definition, registry, null, EXPR);

    var anomalies = new java.util.ArrayList<io.casehub.blocks.summarisation.LevelEvent<?>>();
    pipeline.<Object>outputBus("anomalies").subscribe(e -> true, anomalies::add);

    publishFaultEvent(pipeline, "node-1", "agent", "PROVISION_FAILED", "timeout", "tenant-1");
    publishFaultEvent(pipeline, "node-2", "channel", "DEPENDENCY_UNAVAILABLE", "db down", "tenant-1");
    // Fill window to 10
    for (int i = 0; i < 8; i++) {
        publishRecoveryEvent(pipeline, "node-" + (i + 3), "agent", "tenant-1");
    }
    pipeline.tick(1000L).toCompletableFuture().join();

    var faultAnomalies = anomalies.stream()
            .filter(e -> e.payload() instanceof java.util.Map<?,?> m && "PROVISION_FAILURE".equals(m.get("category")))
            .toList();
    assertThat(faultAnomalies).hasSize(1);

    var depAnomalies = anomalies.stream()
            .filter(e -> e.payload() instanceof java.util.Map<?,?> m && "DEPENDENCY_FAULT".equals(m.get("category")))
            .toList();
    assertThat(depAnomalies).hasSize(1);

    var stabilising = anomalies.stream()
            .filter(e -> e.payload() instanceof java.util.Map<?,?> m && "STABILISING".equals(m.get("category")))
            .toList();
    assertThat(stabilising).hasSize(8);
}

@Test
void phaseTransitionsToDegraded() {
    var pipeline = new PipelineCompiler().compile(definition, registry, null, EXPR);

    var phases = new java.util.ArrayList<io.casehub.blocks.summarisation.LevelEvent<?>>();
    pipeline.<Object>outputBus("phases").subscribe(e -> true, phases::add);

    // Publish 10 critical faults to fill anomaly window
    for (int i = 0; i < 10; i++) {
        publishFaultEvent(pipeline, "node-" + i, "agent", "PROVISION_FAILED", "timeout", null);
    }
    pipeline.tick(1000L).toCompletableFuture().join();

    // Tick past the phase window age to drain
    pipeline.tick(70_000L).toCompletableFuture().join();

    assertThat(phases).hasSizeGreaterThanOrEqualTo(1);
    assertThat(phases.get(0).payload()).isInstanceOfSatisfying(java.util.Map.class, m -> {
        assertThat(m).containsEntry("from", "HEALTHY");
        assertThat(m).containsEntry("to", "DEGRADED");
    });
}

@SuppressWarnings("unchecked")
private void publishFaultEvent(CompiledPipeline<?> pipeline,
                                String nodeId, String nodeType, String faultType,
                                String reason, String tenancyId) {
    var bus = (io.casehub.blocks.summarisation.EventStreamBus<Object>) (Object) pipeline.inputBus();
    bus.publish(new io.casehub.blocks.summarisation.LevelEvent<>(
            java.util.Map.<String, Object>of(
                    "tenancyId", nodeId.contains("tenant") ? tenancyId : tenancyId != null ? tenancyId : "",
                    "nodeId", nodeId,
                    "nodeType", nodeType,
                    "faultType", faultType,
                    "reason", reason,
                    "graphVersion", 1L,
                    "parentNodeId", ""),
            System.currentTimeMillis(),
            new io.casehub.blocks.summarisation.EventLevel("input", 0),
            tenancyId));
}

@SuppressWarnings("unchecked")
private void publishRecoveryEvent(CompiledPipeline<?> pipeline,
                                   String nodeId, String nodeType, String tenancyId) {
    var bus = (io.casehub.blocks.summarisation.EventStreamBus<Object>) (Object) pipeline.inputBus();
    bus.publish(new io.casehub.blocks.summarisation.LevelEvent<>(
            java.util.Map.<String, Object>of(
                    "tenancyId", tenancyId != null ? tenancyId : "",
                    "nodeId", nodeId,
                    "nodeType", nodeType,
                    "graphVersion", 1L,
                    "parentNodeId", ""),
            System.currentTimeMillis(),
            new io.casehub.blocks.summarisation.EventLevel("input", 0),
            tenancyId));
}
```

- [ ] **Step 8: Run all tests to verify**

Run: `mvn --batch-mode -o test -pl app -Dtest=DeploymentMonitoringPipelineTest -DrunQuarkusTests`

Expected: PASS — all 4 tests

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/170/ops add \
  app/pom.xml \
  api/src/main/java/io/casehub/ops/api/lifecycle/DeploymentSummarisationEventTypes.java \
  app/src/main/resources/META-INF/summarisation/deployment-monitoring.yaml \
  app/src/test/java/io/casehub/ops/app/lifecycle/summarisation/DeploymentMonitoringPipelineTest.java
git -C /Users/mdproctor/claude/casehub/slots/170/ops commit -m "feat(#84): add deployment monitoring summarisation pipeline

YAML pipeline consuming desiredstate L1 CloudEvents (node.faulted,
node.drifted, node.recovered) via type-prefix matching. L2 classifies
faults by FaultType with severity tiers. L3 phase-detect tracks
HEALTHY/DEGRADED/RECOVERING transitions. Refs casehubio/casehub-ops#84"
```

---

## Batch 2: RAS Situation Definitions + Integration Test

### Task 2: Deployment Topology RAS Situation Definitions

**Files:**
- Create: `app/src/main/java/io/casehub/ops/app/lifecycle/ras/DeploymentTopologySituationDefinitionProvider.java`
- Test: `app/src/test/java/io/casehub/ops/app/lifecycle/ras/DeploymentTopologySituationDefinitionProviderTest.java`

**Interfaces:**
- Consumes: `DeploymentSummarisationEventTypes.PHASE` (L3 CloudEvent type), `GanglionDescriptor.ExpressionRules` (RAS API), `SituationRegistration` + `SituationDefinition` + `ChainMode` (RAS API), `LambdaExpression` (platform API)
- Produces: Ganglia IDs `deployment-degraded`, `deployment-recovering`, `deployment-healthy` — used by situation registrations and available for downstream `situation-watcher` composition

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.ops.app.lifecycle.ras;

import io.casehub.ops.api.lifecycle.DeploymentSummarisationEventTypes;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class DeploymentTopologySituationDefinitionProviderTest {

    private DeploymentTopologySituationDefinitionProvider provider;

    @BeforeEach
    void setUp() {
        provider = new DeploymentTopologySituationDefinitionProvider();
    }

    @Test
    void declaresThreeGanglia() {
        var ganglia = provider.ganglionDescriptors();
        assertThat(ganglia).hasSize(3);
        assertThat(ganglia).extracting("id")
                .containsExactlyInAnyOrder(
                        "deployment-degraded",
                        "deployment-recovering",
                        "deployment-healthy");
    }

    @Test
    void allGangliaConsumePhaseEvents() {
        var ganglia = provider.ganglionDescriptors();
        for (var g : ganglia) {
            assertThat(g).isInstanceOfSatisfying(
                    io.casehub.ras.api.GanglionDescriptor.ExpressionRules.class,
                    er -> assertThat(er.eventTypes())
                            .contains(DeploymentSummarisationEventTypes.PHASE));
        }
    }

    @Test
    void declaresSituationRegistration() {
        var registrations = provider.registrations();
        assertThat(registrations).hasSize(1);
        assertThat(registrations.get(0).definition().id())
                .isEqualTo("deployment.sustained-degradation");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode -o test -pl app -Dtest=DeploymentTopologySituationDefinitionProviderTest -DrunQuarkusTests`

Expected: FAIL — class not found

- [ ] **Step 3: Implement DeploymentTopologySituationDefinitionProvider**

```java
package io.casehub.ops.app.lifecycle.ras;

import io.casehub.ops.api.lifecycle.DeploymentSummarisationEventTypes;
import io.casehub.platform.api.expression.LambdaExpression;
import io.casehub.ras.api.*;
import jakarta.enterprise.context.ApplicationScoped;

import java.time.Duration;
import java.util.*;

@ApplicationScoped
public class DeploymentTopologySituationDefinitionProvider implements SituationDefinitionProvider {

    public static final String DEGRADED_ID = "deployment-degraded";
    public static final String RECOVERING_ID = "deployment-recovering";
    public static final String HEALTHY_ID = "deployment-healthy";

    @Override
    public List<GanglionDescriptor> ganglionDescriptors() {
        return List.of(
                ganglion(DEGRADED_ID, ctx -> {
                    Map<String, Object> data = data(ctx);
                    return data != null && "DEGRADED".equals(data.get("to"));
                }, 0.95),
                ganglion(RECOVERING_ID, ctx -> {
                    Map<String, Object> data = data(ctx);
                    return data != null && "RECOVERING".equals(data.get("to"));
                }, 0.9),
                ganglion(HEALTHY_ID, ctx -> {
                    Map<String, Object> data = data(ctx);
                    return data != null && "HEALTHY".equals(data.get("to"));
                }, 0.95));
    }

    @Override
    public List<SituationRegistration> registrations() {
        return List.of(new SituationRegistration(
                new SituationDefinition(
                        "deployment.sustained-degradation",
                        Set.of(DeploymentSummarisationEventTypes.PHASE),
                        Duration.ofMinutes(15),
                        null,
                        new ChainMode.Streak(DEGRADED_ID, 2),
                        new TriggerAction.CreateCase(
                                new CaseTriggerConfig("ops-deployment", "escalate", "1.0", Map.of())),
                        new TriggerMode.FireOnce()),
                null));
    }

    @SuppressWarnings("unchecked")
    private static Map<String, Object> data(Map ctx) {
        Object d = ctx.get("data");
        return d instanceof Map ? (Map<String, Object>) d : null;
    }

    @SuppressWarnings("unchecked")
    private static GanglionDescriptor ganglion(String id,
                                                java.util.function.Function<Map, Boolean> condition,
                                                double confidence) {
        return new GanglionDescriptor.ExpressionRules(
                id,
                Set.of(DeploymentSummarisationEventTypes.PHASE),
                List.of(new GanglionDescriptor.ExpressionRules.Rule(
                        new LambdaExpression<>(condition),
                        DetectionSignal.DETECTED,
                        confidence,
                        null,
                        Map.of())),
                Map.of());
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode -o test -pl app -Dtest=DeploymentTopologySituationDefinitionProviderTest -DrunQuarkusTests`

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/170/ops add \
  app/src/main/java/io/casehub/ops/app/lifecycle/ras/DeploymentTopologySituationDefinitionProvider.java \
  app/src/test/java/io/casehub/ops/app/lifecycle/ras/DeploymentTopologySituationDefinitionProviderTest.java
git -C /Users/mdproctor/claude/casehub/slots/170/ops commit -m "feat(#84): add deployment topology RAS situation definitions

Three ganglia (deployment-degraded/recovering/healthy) consuming L3
phase events. Sustained-degradation situation uses Streak(2) chain
mode to detect repeated DEGRADED transitions before escalating.
Refs casehubio/casehub-ops#84"
```

### Task 3: End-to-End Integration Test

**Files:**
- Create: `app/src/test/java/io/casehub/ops/app/lifecycle/summarisation/DeploymentSummarisationIntegrationTest.java`

**Interfaces:**
- Consumes: `DeploymentMonitoringPipelineTest` (pipeline helpers from Task 1), `DeploymentTopologySituationDefinitionProvider` (Task 2), `PipelineCompiler`, `SummariserRegistry`, `GanglionDescriptor.ExpressionRules`
- Produces: nothing — validates the full L1→L2→L3→RAS chain in a single test

- [ ] **Step 1: Write the integration test**

```java
package io.casehub.ops.app.lifecycle.summarisation;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import io.casehub.blocks.summarisation.EventLevel;
import io.casehub.blocks.summarisation.EventStreamBus;
import io.casehub.blocks.summarisation.LevelEvent;
import io.casehub.blocks.summarisation.yaml.*;
import io.casehub.blocks.summarisation.yaml.builtin.*;
import io.casehub.ops.app.lifecycle.ras.DeploymentTopologySituationDefinitionProvider;
import io.casehub.platform.api.expression.ExpressionEngine;
import io.casehub.platform.expression.MvelExpressionEngine;
import io.casehub.ras.api.DetectionSignal;
import io.casehub.ras.api.GanglionDescriptor;
import io.cloudevents.CloudEvent;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.io.IOException;
import java.util.*;

import static org.assertj.core.api.Assertions.assertThat;

class DeploymentSummarisationIntegrationTest {

    static final ObjectMapper MAPPER = new ObjectMapper(new YAMLFactory());
    static final ExpressionEngine EXPR = new MvelExpressionEngine();
    static final EventLevel INPUT = new EventLevel("input", 0);

    CompiledPipeline<Object> pipeline;
    List<CloudEvent> emittedCloudEvents;
    DeploymentTopologySituationDefinitionProvider situationProvider;

    @BeforeEach
    @SuppressWarnings("unchecked")
    void setUp() throws IOException {
        var yaml = getClass().getResourceAsStream(
                "/META-INF/summarisation/deployment-monitoring.yaml");
        var definition = MAPPER.readValue(yaml, PipelineWrapper.class).pipeline();

        var registry = new SummariserRegistry();
        registry.register("threshold-classify", (SummariserFactory)
                config -> ThresholdClassifySummariser.create(config, EXPR));
        registry.register("phase-detect", (SummariserFactory) config -> {
            var aggregateFields = definition.levels().stream()
                    .filter(l -> l.summariser().type().equals("phase-detect"))
                    .findFirst()
                    .map(LevelDefinition::aggregateFields)
                    .orElse(List.of());
            return PhaseDetectSummariser.create(config, aggregateFields);
        });

        emittedCloudEvents = new ArrayList<>();
        pipeline = (CompiledPipeline<Object>) (CompiledPipeline<?>)
                new PipelineCompiler().compile(definition, registry, emittedCloudEvents::add, EXPR);
        situationProvider = new DeploymentTopologySituationDefinitionProvider();
    }

    @Test
    void fullChain_faultsProduceDegradedPhase_thenSituationDetects() {
        var phaseOutput = new ArrayList<LevelEvent<?>>();
        pipeline.<Object>outputBus("phases").subscribe(e -> true, phaseOutput::add);

        // Publish 10 critical faults (fills anomaly window of 10)
        for (int i = 0; i < 10; i++) {
            publishFault("node-" + i, "agent", "PROVISION_FAILED", "timeout");
        }
        pipeline.tick(1000L).toCompletableFuture().join();

        // Tick past the phase window age (60s) to drain
        pipeline.tick(70_000L).toCompletableFuture().join();

        // Verify L3 phase transition emitted
        assertThat(phaseOutput).hasSizeGreaterThanOrEqualTo(1);
        assertThat(phaseOutput.get(0).payload()).isInstanceOfSatisfying(Map.class, m -> {
            assertThat(m).containsEntry("from", "HEALTHY");
            assertThat(m).containsEntry("to", "DEGRADED");
        });

        // Verify L3 CloudEvent was emitted
        var phaseEvents = emittedCloudEvents.stream()
                .filter(ce -> ce.getType().equals("io.casehub.ops.deployment.phase"))
                .toList();
        assertThat(phaseEvents).isNotEmpty();

        // Verify RAS ganglia would detect this
        var ganglia = situationProvider.ganglionDescriptors();
        var degradedGanglion = ganglia.stream()
                .filter(g -> g instanceof GanglionDescriptor.ExpressionRules er
                        && er.id().equals("deployment-degraded"))
                .findFirst()
                .orElseThrow();
        assertThat(degradedGanglion).isNotNull();
    }

    @Test
    void fullChain_recoveriesTransitionToHealthy() {
        var phaseOutput = new ArrayList<LevelEvent<?>>();
        pipeline.<Object>outputBus("phases").subscribe(e -> true, phaseOutput::add);

        // Phase 1: cause DEGRADED
        for (int i = 0; i < 10; i++) {
            publishFault("node-" + i, "agent", "PROVISION_FAILED", "timeout");
        }
        pipeline.tick(1000L).toCompletableFuture().join();
        pipeline.tick(70_000L).toCompletableFuture().join();
        assertThat(phaseOutput).hasSizeGreaterThanOrEqualTo(1);

        // Phase 2: cause RECOVERING (zero critical faults in next window)
        for (int i = 0; i < 20; i++) {
            publishRecovery("node-" + i, "agent");
        }
        pipeline.tick(71_000L).toCompletableFuture().join();
        pipeline.tick(140_000L).toCompletableFuture().join();

        var transitions = phaseOutput.stream()
                .filter(e -> e.payload() instanceof Map<?,?> m && "RECOVERING".equals(m.get("to")))
                .toList();
        assertThat(transitions).hasSizeGreaterThanOrEqualTo(1);
    }

    @SuppressWarnings("unchecked")
    private void publishFault(String nodeId, String nodeType,
                               String faultType, String reason) {
        var bus = (EventStreamBus<Object>) (Object) pipeline.inputBus();
        bus.publish(new LevelEvent<>(
                Map.<String, Object>of(
                        "nodeId", nodeId,
                        "nodeType", nodeType,
                        "faultType", faultType,
                        "reason", reason,
                        "graphVersion", 1L,
                        "parentNodeId", ""),
                System.currentTimeMillis(), INPUT, null));
    }

    @SuppressWarnings("unchecked")
    private void publishRecovery(String nodeId, String nodeType) {
        var bus = (EventStreamBus<Object>) (Object) pipeline.inputBus();
        bus.publish(new LevelEvent<>(
                Map.<String, Object>of(
                        "nodeId", nodeId,
                        "nodeType", nodeType,
                        "graphVersion", 1L,
                        "parentNodeId", ""),
                System.currentTimeMillis(), INPUT, null));
    }
}
```

- [ ] **Step 2: Run test to verify it passes**

Run: `mvn --batch-mode -o test -pl app -Dtest=DeploymentSummarisationIntegrationTest -DrunQuarkusTests`

Expected: PASS — both integration tests

- [ ] **Step 3: Run full module tests**

Run: `mvn --batch-mode test -pl api,app -DrunQuarkusTests`

Expected: All tests pass, including existing `OpsMonitoringSituationDefinitionProviderTest` and `OpsCloudEventTypesTest`

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/170/ops add \
  app/src/test/java/io/casehub/ops/app/lifecycle/summarisation/DeploymentSummarisationIntegrationTest.java
git -C /Users/mdproctor/claude/casehub/slots/170/ops commit -m "test(#84): add end-to-end deployment summarisation integration test

Validates full L1→L2→L3 chain: fault events produce classified
anomalies, anomalies trigger DEGRADED phase transition, recovery
events transition through RECOVERING to HEALTHY.
Refs casehubio/casehub-ops#84"
```

---

## References

- `docs/specs/issue-74-topology-implementation/2026-09-02-summarisation-ras-integration-design.md` — design spec (9 decisions: D1–D9)
- `blocks/summarisation-yaml/src/test/resources/META-INF/summarisation/logistics-hub.yaml` — reference YAML pipeline
- `blocks/summarisation-yaml/src/test/java/.../LogisticsIntegrationTest.java` — reference test pattern
- `app/src/main/java/io/casehub/ops/app/lifecycle/ras/OpsMonitoringSituationDefinitionProvider.java` — existing RAS pattern (37 ganglia)
- `src/main/java/io/casehub/desiredstate/ras/DesiredStateSituationDefinitionProvider.java` — situation registration pattern
- `src/main/java/io/casehub/desiredstate/api/DesiredStateEventTypes.java` — L1 CloudEvent type URIs
- `src/main/java/io/casehub/desiredstate/api/NodeFaultedData.java` — faultType field (L2 discriminator)
- `src/main/java/io/casehub/desiredstate/api/FaultType.java` — enum values for classification rules
- `blocks/summarisation-yaml/src/main/java/.../ThresholdClassifySummariser.java` — MVEL expression context = payload map
- `blocks/summarisation-yaml/src/main/java/.../PhaseDetectSummariser.java` — count-field/count-value transition predicates
- GitHub casehubio/casehub-ops#84 — focal issue
- GitHub casehubio/blocks#233 — prerequisite (CLOSED)
