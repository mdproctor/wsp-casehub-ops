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
- All new code in casehub-ops (app/, api/, and examples/ modules)
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

### Task 3: Run full module tests

- [ ] **Step 1: Run full module tests**

Run: `mvn --batch-mode test -pl api,app -DrunQuarkusTests`

Expected: All tests pass, including existing `OpsMonitoringSituationDefinitionProviderTest` and `OpsCloudEventTypesTest`

---

## Batch 3: Deployment Monitoring Example

### Task 4: Example Module — Deployment Monitoring Scenario

Self-contained example module demonstrating the full L1→L2→L3→RAS chain with a
realistic deployment failure cascade. Follows the desiredstate `examples/pipeline/`
pattern — production-like code with separate tests verifying the example runs.

**Files:**
- Create: `examples/deployment-monitoring/pom.xml`
- Create: `examples/deployment-monitoring/src/main/java/io/casehub/ops/example/monitoring/DeploymentEventSimulator.java`
- Create: `examples/deployment-monitoring/src/main/java/io/casehub/ops/example/monitoring/DeploymentMonitoringExample.java`
- Create: `examples/deployment-monitoring/src/main/resources/META-INF/summarisation/deployment-monitoring.yaml` (copy from app/)
- Create: `examples/deployment-monitoring/src/test/java/io/casehub/ops/example/monitoring/DeploymentMonitoringExampleTest.java`
- Modify: `pom.xml` (root) — add `examples/deployment-monitoring` to `<modules>`

**Interfaces:**
- Consumes: `PipelineCompiler`, `SummariserRegistry`, `ThresholdClassifySummariser`, `PhaseDetectSummariser` (blocks summarisation-yaml), `GanglionDescriptor.ExpressionRules` (ras-api), `LambdaExpression` (platform-api), `EventStreamBus`, `LevelEvent`, `EventLevel` (blocks summarisation-api)
- Produces: `DeploymentEventSimulator` — generates realistic multi-phase event sequences; `DeploymentMonitoringExample` — runs the full pipeline with text output showing each event, classification, phase transition, and situation detection

- [ ] **Step 1: Create pom.xml for example module**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-ops-parent</artifactId>
        <version>0.2-SNAPSHOT</version>
    </parent>

    <artifactId>casehub-ops-example-deployment-monitoring</artifactId>

    <name>CaseHub Ops :: Examples :: Deployment Monitoring</name>
    <description>Teaching example: summarisation-enhanced RAS detection for deployment
        topologies. Demonstrates L1→L2→L3 event pipeline with phase detection and
        situation awareness.</description>

    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-blocks-summarisation-yaml</artifactId>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-ras-api</artifactId>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-platform-expression</artifactId>
        </dependency>

        <!-- Test -->
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
```

- [ ] **Step 2: Add module to root pom.xml**

Add `<module>examples/deployment-monitoring</module>` to the `<modules>` section in the root pom.xml.

- [ ] **Step 3: Copy the pipeline YAML**

Copy `app/src/main/resources/META-INF/summarisation/deployment-monitoring.yaml` to
`examples/deployment-monitoring/src/main/resources/META-INF/summarisation/deployment-monitoring.yaml`.

Same YAML — the example is self-contained.

- [ ] **Step 4: Create DeploymentEventSimulator**

Generates realistic event sequences mimicking a deployment failure cascade:
Phase 1 (steady state) → Phase 2 (cascading failures) → Phase 3 (partial recovery) → Phase 4 (full recovery).

```java
package io.casehub.ops.example.monitoring;

import io.casehub.blocks.summarisation.EventLevel;
import io.casehub.blocks.summarisation.EventStreamBus;
import io.casehub.blocks.summarisation.LevelEvent;

import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

public class DeploymentEventSimulator {

    public record SimulatedEvent(String type, Map<String, Object> payload, long timestamp) {}

    private static final EventLevel INPUT = new EventLevel("input", 0);

    public static List<SimulatedEvent> cascadeScenario() {
        var events = new ArrayList<SimulatedEvent>();
        long t = 0;

        // Phase 1: steady state — a few recoveries
        for (int i = 0; i < 3; i++) {
            events.add(recovery("agent-" + i, "agent", t += 1000));
        }

        // Phase 2: cascading failures — database dependency goes down
        events.add(fault("db-primary", "database", "DEPENDENCY_UNAVAILABLE",
                "connection refused: postgres:5432", t += 2000));
        for (int i = 0; i < 6; i++) {
            events.add(fault("agent-" + i, "agent", "PROVISION_FAILED",
                    "dependency db-primary unavailable", t += 500));
        }
        events.add(fault("channel-inbound", "channel", "PROVISION_FAILED",
                "upstream agent-0 not ready", t += 500));
        events.add(fault("channel-outbound", "channel", "PROVISION_FAILED",
                "upstream agent-1 not ready", t += 500));
        events.add(fault("endpoint-api", "endpoint", "NODE_DEGRADED",
                "health check failing", t += 1000));

        // Phase 3: partial recovery — database comes back, agents start recovering
        events.add(recovery("db-primary", "database", t += 5000));
        for (int i = 0; i < 4; i++) {
            events.add(recovery("agent-" + i, "agent", t += 1000));
        }

        // Phase 4: full recovery — remaining nodes recover
        for (int i = 4; i < 6; i++) {
            events.add(recovery("agent-" + i, "agent", t += 1000));
        }
        events.add(recovery("channel-inbound", "channel", t += 500));
        events.add(recovery("channel-outbound", "channel", t += 500));
        events.add(recovery("endpoint-api", "endpoint", t += 500));

        return List.copyOf(events);
    }

    public static void publishTo(EventStreamBus<Object> bus, List<SimulatedEvent> events) {
        for (var event : events) {
            bus.publish(new LevelEvent<>(event.payload(), event.timestamp(), INPUT, null));
        }
    }

    private static SimulatedEvent fault(String nodeId, String nodeType,
                                         String faultType, String reason, long timestamp) {
        var payload = new LinkedHashMap<String, Object>();
        payload.put("nodeId", nodeId);
        payload.put("nodeType", nodeType);
        payload.put("faultType", faultType);
        payload.put("reason", reason);
        payload.put("graphVersion", 1L);
        payload.put("parentNodeId", "");
        return new SimulatedEvent("io.casehub.desiredstate.node.faulted", payload, timestamp);
    }

    private static SimulatedEvent recovery(String nodeId, String nodeType, long timestamp) {
        var payload = new LinkedHashMap<String, Object>();
        payload.put("nodeId", nodeId);
        payload.put("nodeType", nodeType);
        payload.put("graphVersion", 1L);
        payload.put("parentNodeId", "");
        return new SimulatedEvent("io.casehub.desiredstate.node.recovered", payload, timestamp);
    }
}
```

- [ ] **Step 5: Create DeploymentMonitoringExample**

Wires the pipeline, runs the scenario, and prints a clear timeline of what happened at each level.

```java
package io.casehub.ops.example.monitoring;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import io.casehub.blocks.summarisation.LevelEvent;
import io.casehub.blocks.summarisation.yaml.*;
import io.casehub.blocks.summarisation.yaml.builtin.*;
import io.casehub.platform.expression.MvelExpressionEngine;

import java.io.IOException;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;

public class DeploymentMonitoringExample {

    public record Result(List<Map<String, Object>> anomalies,
                         List<Map<String, Object>> phases) {}

    @SuppressWarnings("unchecked")
    public static Result run() throws IOException {
        var mapper = new ObjectMapper(new YAMLFactory());
        var expr = new MvelExpressionEngine();

        var yaml = DeploymentMonitoringExample.class.getResourceAsStream(
                "/META-INF/summarisation/deployment-monitoring.yaml");
        var definition = mapper.readValue(yaml, PipelineWrapper.class).pipeline();

        var registry = new SummariserRegistry();
        registry.register("threshold-classify", (SummariserFactory)
                config -> ThresholdClassifySummariser.create(config, expr));
        registry.register("phase-detect", (SummariserFactory) config -> {
            var aggregateFields = definition.levels().stream()
                    .filter(l -> l.summariser().type().equals("phase-detect"))
                    .findFirst()
                    .map(LevelDefinition::aggregateFields)
                    .orElse(List.of());
            return PhaseDetectSummariser.create(config, aggregateFields);
        });

        var pipeline = new PipelineCompiler().compile(definition, registry, null, expr);

        var anomalies = new ArrayList<Map<String, Object>>();
        var phases = new ArrayList<Map<String, Object>>();

        pipeline.<Object>outputBus("anomalies").subscribe(e -> true, e -> {
            if (e.payload() instanceof Map<?, ?> m) {
                anomalies.add((Map<String, Object>) m);
            }
        });
        pipeline.<Object>outputBus("phases").subscribe(e -> true, e -> {
            if (e.payload() instanceof Map<?, ?> m) {
                phases.add((Map<String, Object>) m);
            }
        });

        var scenario = DeploymentEventSimulator.cascadeScenario();

        System.out.println("=== Deployment Monitoring — Cascade Failure Scenario ===");
        System.out.println();
        System.out.printf("  %d L1 events in scenario%n", scenario.size());
        System.out.println();

        // Publish all events
        DeploymentEventSimulator.publishTo(
                (io.casehub.blocks.summarisation.EventStreamBus<Object>) (Object) pipeline.inputBus(),
                scenario);

        // Tick to drain all windows
        long maxTimestamp = scenario.stream().mapToLong(DeploymentEventSimulator.SimulatedEvent::timestamp).max().orElse(0);
        pipeline.tick(maxTimestamp + 1000).toCompletableFuture().join();
        pipeline.tick(maxTimestamp + 70_000).toCompletableFuture().join();
        pipeline.flush().toCompletableFuture().join();

        // Print L2 anomalies
        System.out.println("--- L2: Anomaly Classification ---");
        for (var a : anomalies) {
            System.out.printf("  [%s] %s — %s  (severity: %s)%n",
                    a.getOrDefault("category", "?"),
                    a.getOrDefault("nodeId", "?"),
                    a.getOrDefault("reason", a.getOrDefault("nodeType", "")),
                    a.getOrDefault("severity", "?"));
        }
        System.out.println();

        // Print L3 phase transitions
        System.out.println("--- L3: Phase Transitions ---");
        if (phases.isEmpty()) {
            System.out.println("  (no phase transitions detected)");
        }
        for (var p : phases) {
            System.out.printf("  %s → %s%n",
                    p.getOrDefault("from", "?"),
                    p.getOrDefault("to", "?"));
        }
        System.out.println();
        System.out.println("=== Done ===");

        return new Result(List.copyOf(anomalies), List.copyOf(phases));
    }

    public static void main(String[] args) throws IOException {
        run();
    }
}
```

- [ ] **Step 6: Write the example verification test**

```java
package io.casehub.ops.example.monitoring;

import org.junit.jupiter.api.Test;

import java.io.IOException;

import static org.assertj.core.api.Assertions.assertThat;

class DeploymentMonitoringExampleTest {

    @Test
    void scenarioProducesAnomalies() throws IOException {
        var result = DeploymentMonitoringExample.run();
        assertThat(result.anomalies()).isNotEmpty();
    }

    @Test
    void scenarioProducesPhaseTransitions() throws IOException {
        var result = DeploymentMonitoringExample.run();
        assertThat(result.phases()).isNotEmpty();
        assertThat(result.phases()).anyMatch(p -> "DEGRADED".equals(p.get("to")));
    }

    @Test
    void anomaliesIncludeProvisionFailures() throws IOException {
        var result = DeploymentMonitoringExample.run();
        var provisionFailures = result.anomalies().stream()
                .filter(a -> "PROVISION_FAILURE".equals(a.get("category")))
                .toList();
        assertThat(provisionFailures).isNotEmpty();
    }

    @Test
    void anomaliesIncludeStabilisingEvents() throws IOException {
        var result = DeploymentMonitoringExample.run();
        var stabilising = result.anomalies().stream()
                .filter(a -> "STABILISING".equals(a.get("category")))
                .toList();
        assertThat(stabilising).isNotEmpty();
    }
}
```

- [ ] **Step 7: Run example tests**

Run: `mvn --batch-mode test -pl examples/deployment-monitoring`

Expected: PASS — all 4 tests, with scenario output printed to console

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/170/ops add \
  pom.xml \
  examples/deployment-monitoring/
git -C /Users/mdproctor/claude/casehub/slots/170/ops commit -m "feat(#84): add deployment monitoring example

Self-contained example demonstrating the full L1→L2→L3 summarisation
chain. DeploymentEventSimulator generates a realistic cascade failure
scenario (db down → agents fail → channels fail → recovery).
DeploymentMonitoringExample wires the YAML pipeline and prints the
classified anomalies and phase transitions.
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
