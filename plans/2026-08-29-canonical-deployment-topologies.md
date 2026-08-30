# Canonical Deployment Topologies Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** TBD — create issue before starting implementation
**Issue group:** TBD

**Goal:** Prove the CaseHub desired-state YAML frontend can express and manage 14
canonical deployment topologies spanning 5 application architectures × 4
infrastructure topologies — using YAML as the primary interface.

**Architecture:** Two-repo change. `casehub-desiredstate` gets a `NodeSpecFactory`
SPI that bridges the `InfraNodeSpec`/`NodeSpec` type gap, generalised
`NodeSpecRegistry`, and jar-protocol module discovery. `casehub-ops` adds 5 new
`InfraNodeSpec` sealed variants, 4 topology YAML modules, 14 exemplars, and a 3-layer
test pyramid. The YAML frontend's existing primitives (modules, invariants, rules,
forEach, lifecycle phases) express topology patterns — no new compiler needed.

**Tech Stack:** Java 21 records, Maven, Jackson YAML, casehub-desiredstate-yaml,
casehub-ops-api, casehub-ops-infra, JUnit 5, AssertJ

## Global Constraints

- Java 21, Maven build: `mvn --batch-mode install`
- All new types are Java records with `@NodeTypeId` annotation
- `InfraNodeSpec.permits` clause must be updated for every new variant
- `resourceType()` values use snake_case: `load_balancer`, `dns_failover`, etc.
- YAML type names match `resourceType()` values exactly
- Optional record fields null-coalesce in compact constructors (YAML path passes null)
- `GenericResourceSpec` excluded from `@NodeTypeId` (dynamic resourceType, D16)
- Cross-repo: `casehub-desiredstate` changes must merge and release before
  `casehub-ops` Batch 3+ can compile
- Topology modules ship in `infra/src/main/resources/META-INF/desiredstate/modules/`
- YAML exemplars use real domain language (not "service-a")

---

## Batch 1: casehub-desiredstate — NodeSpecFactory SPI

Safe wrap point: `casehub-desiredstate` builds with generalized NodeSpecRegistry,
NodeSpecFactory SPI, and jar-protocol module discovery. All existing tests pass.
Backwards-compatible — existing NodeSpec types use DirectCastFactory automatically.

**Repo:** `casehub-desiredstate` (cross-repo prerequisite)

### Task 1: NodeSpecFactory + NodeSpecFactoryProvider SPIs

**Files:**
- Create: `api/src/main/java/io/casehub/desiredstate/api/NodeSpecFactory.java`
- Create: `api/src/main/java/io/casehub/desiredstate/api/NodeSpecFactoryProvider.java`
- Test: `api/src/test/java/io/casehub/desiredstate/api/NodeSpecFactoryTest.java`

**Interfaces:**
- Consumes: `NodeSpec`, `ObjectMapper` (Jackson)
- Produces: `NodeSpecFactory` SPI (used by `NodeSpecRegistry`), `NodeSpecFactoryProvider` SPI (used by CDI discovery). All downstream tasks depend on these interfaces.

- [ ] **Step 1: Write test for NodeSpecFactory contract**

```java
package io.casehub.desiredstate.api;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.Test;
import java.util.Map;
import static org.assertj.core.api.Assertions.assertThat;

class NodeSpecFactoryTest {

    @Test
    void factoryCreatesNodeSpecFromRawMap() {
        NodeSpecFactory factory = (mapper, raw, backendId) ->
                new TestSpec((String) raw.get("name"));
        var result = factory.create(new ObjectMapper(),
                Map.of("name", "test-node"), null);
        assertThat(result).isInstanceOf(TestSpec.class);
        assertThat(((TestSpec) result).name()).isEqualTo("test-node");
    }

    record TestSpec(String name) implements NodeSpec {
        public NodeType nodeType() { return NodeType.of("test"); }
    }
}
```

- [ ] **Step 2: Run test — verify FAIL (interface not defined)**

Run: `mvn --batch-mode test -pl api -Dtest=NodeSpecFactoryTest`
Expected: compilation failure — `NodeSpecFactory` does not exist.

- [ ] **Step 3: Create NodeSpecFactory interface**

```java
package io.casehub.desiredstate.api;

import com.fasterxml.jackson.databind.ObjectMapper;
import java.util.Map;

@FunctionalInterface
public interface NodeSpecFactory {
    NodeSpec create(ObjectMapper mapper, Map<String, Object> rawProperties,
                    String backendId);
}
```

- [ ] **Step 4: Create NodeSpecFactoryProvider interface**

```java
package io.casehub.desiredstate.api;

public interface NodeSpecFactoryProvider {
    boolean handles(Class<?> specClass);
    NodeSpecFactory createFactory(Class<?> specClass);
}
```

- [ ] **Step 5: Run test — verify PASS**

Run: `mvn --batch-mode test -pl api -Dtest=NodeSpecFactoryTest`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add api/src/main/java/io/casehub/desiredstate/api/NodeSpecFactory.java \
        api/src/main/java/io/casehub/desiredstate/api/NodeSpecFactoryProvider.java \
        api/src/test/java/io/casehub/desiredstate/api/NodeSpecFactoryTest.java
git commit -m "feat(api): NodeSpecFactory + NodeSpecFactoryProvider SPIs

Refs casehubio/casehub-desiredstate#TBD"
```

### Task 2: NodeSpecRegistry Generalisation + DirectCastFactoryProvider

**Files:**
- Modify: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/registry/NodeSpecRegistry.java`
- Create: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/registry/DirectCastFactory.java`
- Create: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/registry/DirectCastFactoryProvider.java`
- Modify: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/YamlGraphRecorder.java` (use factory)
- Modify: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/ForEachExpander.java` (use factory)
- Modify: `yaml/runtime/src/test/java/io/casehub/desiredstate/yaml/registry/NodeSpecRegistryTest.java`

**Interfaces:**
- Consumes: `NodeSpecFactory`, `NodeSpecFactoryProvider` from Task 1
- Produces: Generalized `NodeSpecRegistry` mapping type string → `NodeSpecFactory`. `YamlGraphRecorder` and `ForEachExpander` use `factory.create()` instead of `mapper.convertValue()`.

- [ ] **Step 1: Write test for factory-based NodeSpecRegistry**

```java
@Test
void resolveReturnsFactoryNotClass() {
    var provider = new DirectCastFactoryProvider();
    var registry = NodeSpecRegistry.of(
            Map.of("test", TestSpec.class.getName()),
            List.of(provider));
    NodeSpecFactory factory = registry.resolve("test");
    assertThat(factory).isNotNull();
    var spec = factory.create(new ObjectMapper(), Map.of("name", "x"), null);
    assertThat(spec).isInstanceOf(TestSpec.class);
}
```

- [ ] **Step 2: Run test — verify FAIL**

- [ ] **Step 3: Create DirectCastFactory**

```java
package io.casehub.desiredstate.yaml.registry;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.desiredstate.api.NodeSpec;
import io.casehub.desiredstate.api.NodeSpecFactory;
import java.util.Map;

public class DirectCastFactory implements NodeSpecFactory {
    private final Class<? extends NodeSpec> specClass;

    public DirectCastFactory(Class<? extends NodeSpec> specClass) {
        this.specClass = specClass;
    }

    @Override
    public NodeSpec create(ObjectMapper mapper, Map<String, Object> raw,
                           String backendId) {
        return mapper.convertValue(raw, specClass);
    }
}
```

- [ ] **Step 4: Create DirectCastFactoryProvider**

```java
package io.casehub.desiredstate.yaml.registry;

import io.casehub.desiredstate.api.NodeSpec;
import io.casehub.desiredstate.api.NodeSpecFactory;
import io.casehub.desiredstate.api.NodeSpecFactoryProvider;
import jakarta.enterprise.context.ApplicationScoped;

@ApplicationScoped
public class DirectCastFactoryProvider implements NodeSpecFactoryProvider {
    @Override
    public boolean handles(Class<?> specClass) {
        return NodeSpec.class.isAssignableFrom(specClass);
    }

    @SuppressWarnings("unchecked")
    @Override
    public NodeSpecFactory createFactory(Class<?> specClass) {
        return new DirectCastFactory((Class<? extends NodeSpec>) specClass);
    }
}
```

- [ ] **Step 5: Generalise NodeSpecRegistry**

Change internal storage from `Map<String, Class<? extends NodeSpec>>` to
`Map<String, NodeSpecFactory>`. The `of()` factory method takes a list of
`NodeSpecFactoryProvider` beans and delegates to the first matching provider per
class. The `resolve()` method returns `NodeSpecFactory` instead of `Class`.

- [ ] **Step 6: Update YamlGraphRecorder to use factory.create()**

Replace `mapper.convertValue(resolved, specClass)` with
`factory.create(mapper, resolved, yamlNode.backendId())` in both the inline-node
path and the forEach path.

- [ ] **Step 7: Update ForEachExpander to use factory.create()**

Same pattern — replace direct `convertValue` with factory delegation.

- [ ] **Step 8: Run full test suite — verify all pass**

Run: `mvn --batch-mode test -pl yaml/runtime`
Expected: ALL PASS (backwards-compatible via DirectCastFactory)

- [ ] **Step 9: Commit**

### Task 3: YamlNode.backendId + scanNodeTypes Generalisation + discoverModules Jar Protocol

**Files:**
- Modify: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/model/YamlNode.java` (add backendId field)
- Modify: `yaml/deployment/src/main/java/.../YamlDesiredStateProcessor.java` (scanNodeTypes: drop NodeSpec guard; discoverModules: jar protocol)
- Test: existing tests + new test for jar protocol

**Interfaces:**
- Consumes: `@NodeTypeId` annotation (existing in api)
- Produces: `YamlNode.backendId()` accessor. `scanNodeTypes()` discovers ALL `@NodeTypeId` classes. `discoverModules()` finds modules in dependent JARs.

- [ ] **Step 1: Add backendId to YamlNode**

Add `String backendId` field with `@JsonProperty("backendId")`. Default null.
Jackson deserializes it from YAML node-level metadata.

- [ ] **Step 2: Generalise scanNodeTypes()**

Remove the `index.getAllKnownImplementors(NODE_SPEC).contains(cls)` guard.
Discover ALL classes annotated with `@NodeTypeId`, regardless of hierarchy.

- [ ] **Step 3: Extend discoverModules() for jar protocol**

Mirror the existing `discoverYamlFiles()` jar-protocol handling:
```java
if ("jar".equals(url.getProtocol())) {
    JarURLConnection jarConn = (JarURLConnection) url.openConnection();
    // iterate jar entries matching prefix
}
```

- [ ] **Step 4: Run full build — verify all pass**

Run: `mvn --batch-mode install`
Expected: ALL PASS

- [ ] **Step 5: Commit**

---

## Batch 2: casehub-ops-api — Type Extensions

Safe wrap point: 5 new InfraNodeSpec variants + 3 enums + @NodeTypeId on all
existing variants + null-coalescing. `mvn install` passes on ops.

**Repo:** `casehub-ops`

**Prerequisite:** Batch 1 merged and released. Bump `casehub-desiredstate` version in
ops parent pom.xml.

### Task 4: Supporting Enums + 5 New InfraNodeSpec Sealed Variants

**Files:**
- Create: `api/src/main/java/io/casehub/ops/api/infra/types/LoadBalancerType.java`
- Create: `api/src/main/java/io/casehub/ops/api/infra/types/FailoverPolicy.java`
- Create: `api/src/main/java/io/casehub/ops/api/infra/types/ReplicationMode.java`
- Create: `api/src/main/java/io/casehub/ops/api/infra/LoadBalancerSpec.java`
- Create: `api/src/main/java/io/casehub/ops/api/infra/ServiceMeshControlPlaneSpec.java`
- Create: `api/src/main/java/io/casehub/ops/api/infra/SidecarProxySpec.java`
- Create: `api/src/main/java/io/casehub/ops/api/infra/DnsFailoverSpec.java`
- Create: `api/src/main/java/io/casehub/ops/api/infra/DataReplicationSpec.java`
- Modify: `api/src/main/java/io/casehub/ops/api/infra/InfraNodeSpec.java:3-8` (permits)
- Test: `api/src/test/java/io/casehub/ops/api/infra/TopologySpecsTest.java`

**Interfaces:**
- Consumes: `InfraNodeSpec`, `Labels`, `ResourceRequirements` from api/
- Produces: 5 sealed variants + 3 enums. Referenced by YAML type registry, topology modules, and all downstream tasks.

- [ ] **Step 1: Write test covering all 5 new records**

Tests: construction, resourceType(), null-coalescing of optional fields, YAML
deserialization via ObjectMapper. See spec §4.1–4.5 for exact field lists.

```java
package io.casehub.ops.api.infra;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.ops.api.infra.types.*;
import org.junit.jupiter.api.Test;
import java.util.*;
import static org.assertj.core.api.Assertions.*;

class TopologySpecsTest {

    private final ObjectMapper mapper = new ObjectMapper();

    @Test
    void loadBalancerSpecResourceType() {
        var spec = new LoadBalancerSpec("lb", "ns", LoadBalancerType.APPLICATION,
                "/health", 30, List.of("svc"), null);
        assertThat(spec.resourceType()).isEqualTo("load_balancer");
        assertThat(spec.labels()).isEqualTo(Labels.empty());
    }

    @Test
    void loadBalancerSpecFromYamlMap() {
        var map = Map.<String, Object>of(
                "name", "lb", "namespace", "ns", "type", "APPLICATION",
                "healthCheckPath", "/health", "healthCheckIntervalSeconds", 30,
                "targetServices", List.of("svc"));
        var spec = mapper.convertValue(map, LoadBalancerSpec.class);
        assertThat(spec.name()).isEqualTo("lb");
        assertThat(spec.type()).isEqualTo(LoadBalancerType.APPLICATION);
        assertThat(spec.labels()).isEqualTo(Labels.empty());
    }

    @Test
    void dnsFailoverSpecResourceType() {
        var spec = new DnsFailoverSpec("fo", "primary.com", "secondary.com",
                60, "/health", FailoverPolicy.AUTOMATIC);
        assertThat(spec.resourceType()).isEqualTo("dns_failover");
    }

    @Test
    void dataReplicationSpecResourceType() {
        var spec = new DataReplicationSpec("repl", "primary", "dr",
                ReplicationMode.ASYNC, "mydb", 30);
        assertThat(spec.resourceType()).isEqualTo("data_replication");
    }

    @Test
    void meshControlPlaneSpecNullLabels() {
        var spec = new ServiceMeshControlPlaneSpec("cp", "ns", "istio:1.20", 3, null);
        assertThat(spec.resourceType()).isEqualTo("mesh_control_plane");
        assertThat(spec.labels()).isEqualTo(Labels.empty());
    }

    @Test
    void sidecarProxySpecNullResources() {
        var spec = new SidecarProxySpec("proxy", "ns", "envoy:1.28", "svc", null, null);
        assertThat(spec.resourceType()).isEqualTo("sidecar_proxy");
        assertThat(spec.labels()).isEqualTo(Labels.empty());
    }
}
```

- [ ] **Step 2: Run test — verify FAIL**

- [ ] **Step 3: Create 3 enum types**

Per spec §4.6 — one-line each.

- [ ] **Step 4: Create 5 record types with @NodeTypeId + compact constructors**

Per spec §4.1–4.5. Each with `@NodeTypeId` matching `resourceType()`. Compact
constructors: `requireNonNull` for required fields, null-coalesce for optional
(`Labels`, `ResourceRequirements`).

- [ ] **Step 5: Update InfraNodeSpec permits clause**

Add: `LoadBalancerSpec, ServiceMeshControlPlaneSpec, SidecarProxySpec, DnsFailoverSpec, DataReplicationSpec`

- [ ] **Step 6: Run test — verify PASS**

- [ ] **Step 7: Commit**

### Task 5: @NodeTypeId on Existing Variants + Null-Coalescing

**Files:**
- Modify: all existing `InfraNodeSpec` records in `api/src/main/java/.../infra/` (add `@NodeTypeId`)
- Modify: `K8sNamespaceSpec.java`, `K8sDeploymentSpec.java`, `K8sServiceSpec.java`,
  `K8sIngressSpec.java`, `K8sConfigMapSpec.java` (null-coalesce per spec §4.8)
- Test: `api/src/test/java/io/casehub/ops/api/infra/InfraNodeSpecTest.java` (extend)

**Interfaces:**
- Consumes: existing `InfraNodeSpec` records
- Produces: all non-generic variants annotated with `@NodeTypeId`, YAML-compatible null-coalescing

- [ ] **Step 1: Write null-coalescing tests**

Test that constructing each record with null optional fields produces defaults
(Labels.empty(), List.of(), Map.of(), etc.) instead of NPE.

- [ ] **Step 2: Run tests — verify FAIL**

- [ ] **Step 3: Add @NodeTypeId to 14 existing non-generic variants**

e.g. `@NodeTypeId("k8s_namespace") public record K8sNamespaceSpec(...)`

- [ ] **Step 4: Update compact constructors per §4.8 table**

Change `requireNonNull` to null-coalescing for each field listed in the table.

- [ ] **Step 5: Run tests — verify PASS**

- [ ] **Step 6: Run full build — verify nothing breaks**

Run: `mvn --batch-mode install`

- [ ] **Step 7: Commit**

---

## Batch 3: casehub-ops-infra — Factory Wiring + Provisioner Updates

Safe wrap point: InfraWrappingFactory wires new types through the YAML frontend.
InfraNodeProvisioner and InfraActualStateAdapter dynamically register all sealed
variants. InfraGoalCompiler can parse all types.

### Task 6: InfraWrappingFactory + InfraNodeSpecFactoryProvider

**Files:**
- Create: `infra/src/main/java/io/casehub/ops/infra/yaml/InfraWrappingFactory.java`
- Create: `infra/src/main/java/io/casehub/ops/infra/yaml/InfraNodeSpecFactoryProvider.java`
- Test: `infra/src/test/java/io/casehub/ops/infra/yaml/InfraWrappingFactoryTest.java`

**Interfaces:**
- Consumes: `NodeSpecFactory`, `NodeSpecFactoryProvider` SPIs (from Batch 1). `InfraNodeSpec`, `InfraDesiredNodeSpec` from api.
- Produces: CDI bean `InfraNodeSpecFactoryProvider` — discovered by `NodeSpecRegistry` at construction time. Creates `InfraWrappingFactory` instances that wrap `InfraNodeSpec` records in `InfraDesiredNodeSpec`.

- [ ] **Step 1: Write test for InfraWrappingFactory**

```java
@Test
void wrapsInfraNodeSpecInDesiredNodeSpec() {
    var factory = new InfraWrappingFactory(LoadBalancerSpec.class, "standalone");
    var raw = Map.<String, Object>of(
            "name", "lb", "namespace", "ns", "type", "APPLICATION",
            "healthCheckPath", "/health", "healthCheckIntervalSeconds", 30,
            "targetServices", List.of("svc"));
    var result = factory.create(new ObjectMapper(), raw, null);

    assertThat(result).isInstanceOf(InfraDesiredNodeSpec.class);
    var wrapper = (InfraDesiredNodeSpec) result;
    assertThat(wrapper.backendId()).isEqualTo("standalone");
    assertThat(wrapper.resourceSpec()).isInstanceOf(LoadBalancerSpec.class);
}

@Test
void nodeBackendIdOverridesDefault() {
    var factory = new InfraWrappingFactory(LoadBalancerSpec.class, "standalone");
    var raw = Map.<String, Object>of("name", "lb", "namespace", "ns",
            "type", "APPLICATION", "healthCheckPath", "/health",
            "healthCheckIntervalSeconds", 30, "targetServices", List.of("svc"));
    var result = factory.create(new ObjectMapper(), raw, "cloud-aws");

    assertThat(((InfraDesiredNodeSpec) result).backendId()).isEqualTo("cloud-aws");
}
```

- [ ] **Step 2: Run test — verify FAIL**

- [ ] **Step 3: Implement InfraWrappingFactory + InfraNodeSpecFactoryProvider**

Per spec §3.3. Provider handles `InfraNodeSpec.class.isAssignableFrom(specClass)`.
`defaultBackend` from `@ConfigProperty`.

- [ ] **Step 4: Run test — verify PASS**

- [ ] **Step 5: Commit**

### Task 7: Dynamic handledTypes() + parseSpec Cases

**Files:**
- Modify: `infra/src/main/java/io/casehub/ops/infra/InfraNodeProvisioner.java:85-95`
- Modify: `infra/src/main/java/io/casehub/ops/infra/InfraActualStateAdapter.java` (handledTypes)
- Modify: `infra/src/main/java/io/casehub/ops/infra/InfraGoalCompiler.java` (6 parseSpec cases)
- Test: `infra/src/test/java/io/casehub/ops/infra/InfraGoalCompilerTest.java` (parseSpec coverage)

**Interfaces:**
- Consumes: `InfraNodeSpec` sealed hierarchy (all 15 variants incl. generic)
- Produces: Dynamic `handledTypes()` that auto-registers all sealed variants. `parseSpec()` handles 5 new types + `k8s_configmap`.

- [ ] **Step 1: Write parseSpec coverage test**

Test that every non-generic `InfraNodeSpec` variant has a `parseSpec()` case.

```java
@Test
void parseSpecCoversAllSealedVariants() {
    var handled = Arrays.stream(InfraNodeSpec.class.getPermittedSubclasses())
            .filter(c -> !c.equals(GenericResourceSpec.class))
            .map(c -> {
                try {
                    return ((InfraNodeSpec) c.getDeclaredConstructors()[0]
                            .newInstance(/* test args */)).resourceType();
                } catch (Exception e) { return null; }
            })
            .filter(Objects::nonNull)
            .collect(Collectors.toSet());

    // Verify each type has a parseSpec case by calling parseSpec with
    // a minimal JsonNode and verifying no exception for known types
    // ... (test each type individually)
}
```

- [ ] **Step 2: Run test — verify FAIL (5 new types have no parseSpec case)**

- [ ] **Step 3: Implement dynamic handledTypes()**

Replace hardcoded `Set.of(...)` with reflection over `InfraNodeSpec.getPermittedSubclasses()`,
reading `@NodeTypeId` from each (skipping `GenericResourceSpec`).

- [ ] **Step 4: Add 6 parseSpec() cases**

`load_balancer`, `mesh_control_plane`, `sidecar_proxy`, `dns_failover`,
`data_replication`, `k8s_configmap` — each parsing the corresponding record from JsonNode.

- [ ] **Step 5: Run tests — verify PASS**

Run: `mvn --batch-mode test -pl infra`

- [ ] **Step 6: Run full build**

Run: `mvn --batch-mode install`

- [ ] **Step 7: Commit**

---

## Batch 4: Topology Modules + Lifecycle Support

Safe wrap point: 4 YAML modules ship in infra.jar. Lifecycle+module interaction
works (cross-repo fix in desiredstate).

### Task 8: Topology YAML Modules

**Files:**
- Create: `infra/src/main/resources/META-INF/desiredstate/modules/load-balancer.yaml`
- Create: `infra/src/main/resources/META-INF/desiredstate/modules/ha-multi-az.yaml`
- Create: `infra/src/main/resources/META-INF/desiredstate/modules/service-mesh.yaml`
- Create: `infra/src/main/resources/META-INF/desiredstate/modules/multi-region.yaml`

**Interfaces:**
- Consumes: `LoadBalancerSpec`, `K8sIngressSpec`, `K8sDeploymentSpec`, `ServiceMeshControlPlaneSpec`, `DataReplicationSpec`, `DnsFailoverSpec` (all via YAML type names)
- Produces: 4 importable modules with parameters, invariants, and rules

- [ ] **Step 1: Create load-balancer.yaml**

Per spec §5.1. Module with `target_service`, `namespace`, `health_check_path`,
`lb_type` parameters. Nodes: `lb` (load_balancer), `ingress` (k8s_ingress).
Invariant: lb-has-target.

- [ ] **Step 2: Create ha-multi-az.yaml**

Per spec §5.2. Module with `namespace`, `region`, `zones` parameters.
Node: ha-control-plane (k8s_deployment). Invariant: ha-control-plane-in-namespace.

- [ ] **Step 3: Create service-mesh.yaml**

Per spec §5.3. Module with `namespace`, `control_plane_image`, `control_plane_replicas`
parameters. Node: mesh-control-plane. Rule: sidecar-depends-on-control-plane.

- [ ] **Step 4: Create multi-region.yaml**

Per spec §5.4. Module with `primary_cluster`, `dr_cluster`, `source_database`,
`failover_health_check`, `replication_mode` parameters. Nodes: data-replication,
dns-failover. Invariant: replication-before-failover.

- [ ] **Step 5: Verify modules parse without error**

Write a quick smoke test in infra that loads each module YAML and verifies Jackson
can deserialize the `YamlModuleFile` record.

- [ ] **Step 6: Commit**

### Task 9: Lifecycle + Module Interaction (cross-repo)

**Files (casehub-desiredstate):**
- Modify: `yaml/runtime/.../YamlGraphRecorder.java` — `createYamlLifecycleGoalCompiler()` gains `availableModules` parameter, calls `ModuleExpander` before phase loop
- Modify: `yaml/deployment/.../YamlDesiredStateProcessor.java` — remove `validateLifecycle()` imports guard, pass modules to lifecycle compiler

**Interfaces:**
- Consumes: `ModuleExpander`, `YamlModule`, `YamlLifecycle` (all existing)
- Produces: Lifecycle phases + module imports work together. Exemplars T4, T6, T10 depend on this.

- [ ] **Step 1: Write test for lifecycle + module composition**

Test that a YAML file with both `lifecycle:` phases and `imports:` compiles
successfully, with module nodes assigned to the correct phase.

- [ ] **Step 2: Run test — verify FAIL (imports guard blocks)**

- [ ] **Step 3: Remove validateLifecycle() imports guard**

- [ ] **Step 4: Add availableModules parameter to createYamlLifecycleGoalCompiler()**

Call `ModuleExpander.expand()` before the phase loop. Implement phase
auto-assignment algorithm (spec §5.5).

- [ ] **Step 5: Run test — verify PASS**

- [ ] **Step 6: Run full desiredstate build**

- [ ] **Step 7: Commit, release, bump version in ops pom.xml**

---

## Batch 5: Topology Exemplars + Compilation Tests

Safe wrap point: 14 YAML exemplars compile to correct DesiredStateGraphs. All
compilation tests pass. topology-tests module is fully wired.

### Task 10: topology-tests Module Scaffold

**Files:**
- Create: `topology-tests/pom.xml`
- Modify: `pom.xml` (parent — add topology-tests to modules list)
- Create: `topology-tests/src/test/java/io/casehub/ops/topology/TopologyTestBase.java`

**Interfaces:**
- Consumes: `YamlGraphRecorder`, `NodeSpecRegistry`, `InfraWrappingFactory`, all InfraNodeSpec types
- Produces: Base test class with `compileExemplar(path)` helper, assertion utilities

- [ ] **Step 1: Create pom.xml with dependencies**

Dependencies: casehub-ops-infra (test), casehub-desiredstate-yaml (test),
casehub-desiredstate (test), junit-jupiter, assertj.

- [ ] **Step 2: Create TopologyTestBase**

Helper: `compileExemplar(String path)` → reads YAML resource, builds
NodeSpecRegistry with InfraNodeSpecFactoryProvider, compiles via YamlGraphRecorder,
returns DesiredStateGraph. Assertion helpers: `assertNodeType()`,
`assertDependency()`, `assertNodeCount()`.

- [ ] **Step 3: Write smoke test**

```java
@Test
void baseClassCanCompileMinimalYaml() { /* single node YAML */ }
```

- [ ] **Step 4: Run — verify PASS**

- [ ] **Step 5: Commit**

### Task 11: Core Exemplars (T1–T5) + Compilation Tests

**Files:**
- Create: `topology-tests/src/test/resources/topologies/single-service/single-node/dev-blog.yaml`
- Create: `topology-tests/src/test/resources/topologies/single-service/lb-cluster/marketing-site.yaml`
- Create: `topology-tests/src/test/resources/topologies/multi-tier/single-node/local-dev-stack.yaml`
- Create: `topology-tests/src/test/resources/topologies/multi-tier/lb-cluster/ecommerce-storefront.yaml`
- Create: `topology-tests/src/test/resources/topologies/multi-tier/ha-multi-az/hospital-records.yaml`
- Test: `topology-tests/src/test/java/io/casehub/ops/topology/CoreTopologyCompilationTest.java`

- [ ] **Step 1: Write compilation tests for T1–T5**

Each test: compile exemplar, assert node count, assert key node types, assert
dependency edges, assert invariant enforcement.

- [ ] **Step 2: Create 5 exemplar YAML files**

Per spec §6 matrix. Real domain language. Use topology modules (load-balancer,
ha-multi-az) where specified.

- [ ] **Step 3: Run tests — verify PASS**

- [ ] **Step 4: Commit**

### Task 12: Advanced Exemplars (T6–T14) + Compilation Tests

**Files:**
- Create: 9 more YAML exemplars in `topology-tests/src/test/resources/topologies/`
- Test: `topology-tests/src/test/java/io/casehub/ops/topology/AdvancedTopologyCompilationTest.java`

- [ ] **Step 1: Write compilation tests for T6–T14**

T6 (banking multi-region), T7-T10 (microservices), T11-T12 (event-driven),
T13-T14 (sidecar/mesh). Each asserts correct compilation and topology-specific
invariant/rule behaviour.

- [ ] **Step 2: Create 9 exemplar YAML files**

Per spec §6 matrix. Use forEach for AZ stamping, service-mesh module for sidecar
injection, multi-region module for DR.

- [ ] **Step 3: Run tests — verify PASS**

- [ ] **Step 4: Run full build**

Run: `mvn --batch-mode install`

- [ ] **Step 5: Commit**

---

## Batch 6: Reconciliation + Live Tests

Safe wrap point: Reconciliation tests verify the full loop. Live tests are
profile-gated and ready for K8s environments.

### Task 13: Reconciliation Tests

**Files:**
- Create: `topology-tests/src/test/java/io/casehub/ops/topology/FailableResourceProvisioner.java`
- Create: `topology-tests/src/test/java/io/casehub/ops/topology/ReconciliationTest.java`

**Interfaces:**
- Consumes: `TransitionPlanner`, `ReconciliationLoop`, `InfraNodeProvisioner`, compiled DesiredStateGraphs from Batch 5
- Produces: Verified reconciliation: correct provision order, drift detection, fault policy escalation

- [ ] **Step 1: Create FailableResourceProvisioner**

Per spec §7.3. Wraps InMemoryResourceProvisioner with deterministic failure injection.

- [ ] **Step 2: Write reconciliation tests**

Tests: provision ordering (namespace before deployments), drift detection (modify
spec, re-reconcile), fault policy escalation (inject 3 failures, verify review node).

- [ ] **Step 3: Add Maven profile `-Preconciliation`**

In topology-tests pom.xml, gate ReconciliationTest with surefire includes.

- [ ] **Step 4: Run with profile — verify PASS**

Run: `mvn --batch-mode test -pl topology-tests -Preconciliation`

- [ ] **Step 5: Commit**

### Task 14: Live K8s Deployment Tests

**Files:**
- Create: `topology-tests/src/test/java/io/casehub/ops/topology/LiveDeploymentTest.java`

**Interfaces:**
- Consumes: K8s API (fabric8 client), real Docker images, compiled DesiredStateGraphs
- Produces: Verified live deployment of K8s-native types (namespace, deployment, service)

- [ ] **Step 1: Write live deployment tests**

Per spec §7.4. Scoped to K8s-native types. Asserts namespace creation, deployment
Ready state, service resolution.

- [ ] **Step 2: Add Maven profile `-Pinfra-live`**

Gate LiveDeploymentTest with surefire includes and fabric8 K8s client dependency.

- [ ] **Step 3: Commit**

---

## References

- [2026-08-29-canonical-deployment-topologies-design.md] — design spec (post-review)
- [docs/research/2026-08-29-canonical-deployment-topologies.md] — research document
- [api/src/main/java/io/casehub/ops/api/infra/InfraNodeSpec.java] — sealed hierarchy
- [infra/src/main/java/io/casehub/ops/infra/InfraNodeProvisioner.java:85-95] — handledTypes
- [infra/src/main/java/io/casehub/ops/infra/InfraGoalCompiler.java:129-141] — parseSpec
- [casehub-desiredstate/yaml/runtime/src/main/java/.../registry/NodeSpecRegistry.java] — type registry
- [casehub-desiredstate/yaml/runtime/src/main/java/.../YamlGraphRecorder.java] — YAML compiler
- [casehub-desiredstate/yaml/runtime/src/main/java/.../ForEachExpander.java] — forEach stamping
- [ARC42STORIES.MD §9.4 L1] — InfraNodeSpec/NodeSpec separation rationale
