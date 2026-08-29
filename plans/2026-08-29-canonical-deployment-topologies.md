# Canonical Deployment Topologies Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** TBD — create issue before starting implementation
**Issue group:** TBD

**Goal:** Prove the CaseHub desired-state YAML frontend can express, compile, and
validate 14 canonical deployment topologies spanning 5 application architectures
and 4 infrastructure topologies.

**Architecture:** YAML-native composition using existing `casehub-desiredstate-yaml`
primitives (modules, forEach, invariants, rules, lifecycle phases). New Java record
types extend `InfraNodeSpec` sealed hierarchy for infrastructure node types not yet
covered (load balancer, DNS failover, data replication, service mesh, sidecar proxy).
A new `topology-tests/` module holds all YAML exemplars and compilation tests.

**Tech Stack:** Java 21 records, Maven, Jackson YAML, casehub-desiredstate-yaml,
casehub-ops-api, JUnit 5, AssertJ

## Global Constraints

- Java 21, Maven build: `mvn --batch-mode install`
- All new types are Java records implementing `InfraNodeSpec` via sealed hierarchy
- `InfraNodeSpec.permits` clause must be updated for every new variant
- `resourceType()` return values use snake_case: `load_balancer`, `dns_failover`, etc.
- YAML type names match `resourceType()` values exactly
- Type registry maps YAML type name → fully qualified Java class name
- topology-tests module is test-scope only — no production code
- YAML exemplars use real domain language and recognisable scenarios (not "service-a")
- Compilation tests use AssertJ, match existing casehub-ops test style

---

## Batch 1: Foundation — Java Types + Module Scaffold

Safe wrap point: new InfraNodeSpec variants compile, topology-tests module scaffolded
with a passing smoke test.

### Task 1: InfraNodeSpec Extensions

**Files:**
- Create: `api/src/main/java/io/casehub/ops/api/infra/LoadBalancerSpec.java`
- Create: `api/src/main/java/io/casehub/ops/api/infra/DnsFailoverSpec.java`
- Create: `api/src/main/java/io/casehub/ops/api/infra/DataReplicationSpec.java`
- Create: `api/src/main/java/io/casehub/ops/api/infra/ServiceMeshControlPlaneSpec.java`
- Create: `api/src/main/java/io/casehub/ops/api/infra/SidecarProxySpec.java`
- Create: `api/src/main/java/io/casehub/ops/api/infra/types/LoadBalancerType.java`
- Create: `api/src/main/java/io/casehub/ops/api/infra/types/FailoverPolicy.java`
- Create: `api/src/main/java/io/casehub/ops/api/infra/types/ReplicationMode.java`
- Modify: `api/src/main/java/io/casehub/ops/api/infra/InfraNodeSpec.java:3-8` (permits clause)

**Interfaces:**
- Consumes: `InfraNodeSpec` sealed interface, `Labels`, `ResourceRequirements` from `api/`
- Produces: 5 new `InfraNodeSpec` implementations + 3 enums. All later tasks reference these types by `resourceType()` string in the YAML type registry.

- [ ] **Step 1: Write unit test for LoadBalancerSpec**

```java
// api/src/test/java/io/casehub/ops/api/infra/LoadBalancerSpecTest.java
package io.casehub.ops.api.infra;

import io.casehub.ops.api.infra.types.Labels;
import io.casehub.ops.api.infra.types.LoadBalancerType;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import static org.assertj.core.api.Assertions.assertThat;

class LoadBalancerSpecTest {

    @Test
    void resourceTypeIsLoadBalancer() {
        var spec = new LoadBalancerSpec(
                "web-lb", "storefront", LoadBalancerType.APPLICATION,
                "/health", 80, List.of("storefront-nginx"),
                Labels.of(Map.of("managed-by", "casehub-ops")));
        assertThat(spec.resourceType()).isEqualTo("load_balancer");
    }

    @Test
    void implementsInfraNodeSpec() {
        var spec = new LoadBalancerSpec(
                "web-lb", "storefront", LoadBalancerType.APPLICATION,
                "/health", 80, List.of("web"), Labels.of(Map.of()));
        assertThat(spec).isInstanceOf(InfraNodeSpec.class);
    }
}
```

- [ ] **Step 2: Run test — expect FAIL (class not found)**

Run: `mvn --batch-mode -o test -pl api -Dtest=LoadBalancerSpecTest`
Expected: compilation error — `LoadBalancerSpec` does not exist

- [ ] **Step 3: Create enums**

```java
// api/src/main/java/io/casehub/ops/api/infra/types/LoadBalancerType.java
package io.casehub.ops.api.infra.types;
public enum LoadBalancerType { APPLICATION, NETWORK }

// api/src/main/java/io/casehub/ops/api/infra/types/FailoverPolicy.java
package io.casehub.ops.api.infra.types;
public enum FailoverPolicy { LATENCY, FAILOVER, WEIGHTED }

// api/src/main/java/io/casehub/ops/api/infra/types/ReplicationMode.java
package io.casehub.ops.api.infra.types;
public enum ReplicationMode { ASYNC, SEMI_SYNC }
```

- [ ] **Step 4: Create LoadBalancerSpec**

```java
// api/src/main/java/io/casehub/ops/api/infra/LoadBalancerSpec.java
package io.casehub.ops.api.infra;

import io.casehub.ops.api.infra.types.Labels;
import io.casehub.ops.api.infra.types.LoadBalancerType;
import java.util.List;
import java.util.Objects;

public record LoadBalancerSpec(
        String name,
        String namespace,
        LoadBalancerType type,
        String healthCheckPath,
        int healthCheckPort,
        List<String> targetServices,
        Labels labels) implements InfraNodeSpec {

    public LoadBalancerSpec {
        Objects.requireNonNull(name, "name");
        Objects.requireNonNull(namespace, "namespace");
        Objects.requireNonNull(type, "type");
        Objects.requireNonNull(healthCheckPath, "healthCheckPath");
        Objects.requireNonNull(targetServices, "targetServices");
        targetServices = List.copyOf(targetServices);
        Objects.requireNonNull(labels, "labels");
    }

    @Override
    public String resourceType() { return "load_balancer"; }
}
```

- [ ] **Step 5: Create remaining 4 specs**

```java
// api/src/main/java/io/casehub/ops/api/infra/DnsFailoverSpec.java
package io.casehub.ops.api.infra;

import io.casehub.ops.api.infra.types.FailoverPolicy;
import java.util.Objects;

public record DnsFailoverSpec(
        String domainName,
        String primaryEndpoint,
        String secondaryEndpoint,
        int ttlSeconds,
        FailoverPolicy policy) implements InfraNodeSpec {

    public DnsFailoverSpec {
        Objects.requireNonNull(domainName, "domainName");
        Objects.requireNonNull(primaryEndpoint, "primaryEndpoint");
        Objects.requireNonNull(secondaryEndpoint, "secondaryEndpoint");
        Objects.requireNonNull(policy, "policy");
    }

    @Override
    public String resourceType() { return "dns_failover"; }
}

// api/src/main/java/io/casehub/ops/api/infra/DataReplicationSpec.java
package io.casehub.ops.api.infra;

import io.casehub.ops.api.infra.types.ReplicationMode;
import java.util.Objects;

public record DataReplicationSpec(
        String sourceCluster,
        String targetCluster,
        String sourceService,
        ReplicationMode mode,
        int lagToleranceSeconds) implements InfraNodeSpec {

    public DataReplicationSpec {
        Objects.requireNonNull(sourceCluster, "sourceCluster");
        Objects.requireNonNull(targetCluster, "targetCluster");
        Objects.requireNonNull(sourceService, "sourceService");
        Objects.requireNonNull(mode, "mode");
    }

    @Override
    public String resourceType() { return "data_replication"; }
}

// api/src/main/java/io/casehub/ops/api/infra/ServiceMeshControlPlaneSpec.java
package io.casehub.ops.api.infra;

import io.casehub.ops.api.infra.types.Labels;
import java.util.Objects;

public record ServiceMeshControlPlaneSpec(
        String namespace,
        String image,
        int replicas,
        Labels labels) implements InfraNodeSpec {

    public ServiceMeshControlPlaneSpec {
        Objects.requireNonNull(namespace, "namespace");
        Objects.requireNonNull(image, "image");
        Objects.requireNonNull(labels, "labels");
    }

    @Override
    public String resourceType() { return "mesh_control_plane"; }
}

// api/src/main/java/io/casehub/ops/api/infra/SidecarProxySpec.java
package io.casehub.ops.api.infra;

import io.casehub.ops.api.infra.types.ResourceRequirements;
import java.util.Objects;

public record SidecarProxySpec(
        String targetService,
        String image,
        ResourceRequirements resources) implements InfraNodeSpec {

    public SidecarProxySpec {
        Objects.requireNonNull(targetService, "targetService");
        Objects.requireNonNull(image, "image");
        Objects.requireNonNull(resources, "resources");
    }

    @Override
    public String resourceType() { return "sidecar_proxy"; }
}
```

- [ ] **Step 6: Update InfraNodeSpec permits clause**

Use `ide_replace_member` to update the sealed interface:

```java
public sealed interface InfraNodeSpec
        permits K8sNamespaceSpec, K8sDeploymentSpec, K8sServiceSpec, K8sIngressSpec,
                K8sConfigMapSpec,
                ComputeInstanceSpec, DatabaseClusterSpec,
                TerraformWorkspaceSpec, AnsiblePlaybookSpec,
                GenericResourceSpec,
                LoadBalancerSpec, DnsFailoverSpec, DataReplicationSpec,
                ServiceMeshControlPlaneSpec, SidecarProxySpec {

    String resourceType();
}
```

- [ ] **Step 7: Run tests — verify compilation and test pass**

Run: `mvn --batch-mode -o test -pl api`
Expected: all tests pass, including `LoadBalancerSpecTest`

- [ ] **Step 8: Commit**

```bash
git add api/src/main/java/io/casehub/ops/api/infra/LoadBalancerSpec.java \
        api/src/main/java/io/casehub/ops/api/infra/DnsFailoverSpec.java \
        api/src/main/java/io/casehub/ops/api/infra/DataReplicationSpec.java \
        api/src/main/java/io/casehub/ops/api/infra/ServiceMeshControlPlaneSpec.java \
        api/src/main/java/io/casehub/ops/api/infra/SidecarProxySpec.java \
        api/src/main/java/io/casehub/ops/api/infra/types/LoadBalancerType.java \
        api/src/main/java/io/casehub/ops/api/infra/types/FailoverPolicy.java \
        api/src/main/java/io/casehub/ops/api/infra/types/ReplicationMode.java \
        api/src/main/java/io/casehub/ops/api/infra/InfraNodeSpec.java \
        api/src/test/java/io/casehub/ops/api/infra/LoadBalancerSpecTest.java
git commit -m "feat: InfraNodeSpec extensions for deployment topologies

Add LoadBalancerSpec, DnsFailoverSpec, DataReplicationSpec,
ServiceMeshControlPlaneSpec, SidecarProxySpec sealed variants.
Supporting enums: LoadBalancerType, FailoverPolicy, ReplicationMode.

Refs #TBD"
```

---

### Task 2: topology-tests Module Scaffold

**Files:**
- Create: `topology-tests/pom.xml`
- Create: `topology-tests/src/test/java/io/casehub/ops/topology/TopologyTestHelper.java`
- Create: `topology-tests/src/test/java/io/casehub/ops/topology/TopologySmokeTest.java`
- Modify: `pom.xml` (parent pom, add topology-tests module)

**Interfaces:**
- Consumes: `YamlGraphRecorder`, `YamlGraph`, `NodeSpecRegistry` from casehub-desiredstate-yaml; all `InfraNodeSpec` types from api/
- Produces: `TopologyTestHelper` — reusable test helper for all compilation tests. Provides `compileTopology(yamlPath)` and `compileTopologyWithModules(yamlPath, modulePaths)`.

- [ ] **Step 1: Create pom.xml**

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

    <artifactId>casehub-ops-topology-tests</artifactId>

    <name>CaseHub Ops :: Topology Tests</name>
    <description>Compilation and integration tests for canonical deployment topologies. Test scope only.</description>

    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-ops-api</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-desiredstate-yaml</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-desiredstate</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-junit5</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.dataformat</groupId>
            <artifactId>jackson-dataformat-yaml</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
```

- [ ] **Step 2: Add topology-tests to parent pom modules list**

Add `<module>topology-tests</module>` after the existing modules in the parent `pom.xml`.

- [ ] **Step 3: Write TopologyTestHelper**

```java
// topology-tests/src/test/java/io/casehub/ops/topology/TopologyTestHelper.java
package io.casehub.ops.topology;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import io.casehub.desiredstate.annotations.runtime.DependencyDescriptor;
import io.casehub.desiredstate.annotations.runtime.GraphDescriptor;
import io.casehub.desiredstate.annotations.runtime.NodeDescriptor;
import io.casehub.desiredstate.annotations.runtime.ResolvedInvariant;
import io.casehub.desiredstate.api.CompilationResult;
import io.casehub.desiredstate.api.DesiredStateGraph;
import io.casehub.desiredstate.api.GoalCompiler;
import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.api.NodeSpec;
import io.casehub.desiredstate.runtime.DefaultDesiredStateGraphFactory;
import io.casehub.desiredstate.yaml.YamlGraphRecorder;
import io.casehub.desiredstate.yaml.YamlInvariantConverter;
import io.casehub.desiredstate.yaml.model.YamlGraph;
import io.casehub.desiredstate.yaml.model.YamlModule;
import io.casehub.desiredstate.yaml.model.YamlModuleFile;
import io.casehub.desiredstate.yaml.model.YamlNode;

import java.io.InputStream;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

public final class TopologyTestHelper {

    private static final ObjectMapper YAML_MAPPER = new ObjectMapper(new YAMLFactory());
    private static final DefaultDesiredStateGraphFactory FACTORY = new DefaultDesiredStateGraphFactory();

    public static final Map<String, String> INFRA_TYPE_REGISTRY = Map.ofEntries(
            Map.entry("k8s_namespace", "io.casehub.ops.api.infra.K8sNamespaceSpec"),
            Map.entry("k8s_deployment", "io.casehub.ops.api.infra.K8sDeploymentSpec"),
            Map.entry("k8s_service", "io.casehub.ops.api.infra.K8sServiceSpec"),
            Map.entry("k8s_ingress", "io.casehub.ops.api.infra.K8sIngressSpec"),
            Map.entry("k8s_configmap", "io.casehub.ops.api.infra.K8sConfigMapSpec"),
            Map.entry("load_balancer", "io.casehub.ops.api.infra.LoadBalancerSpec"),
            Map.entry("dns_failover", "io.casehub.ops.api.infra.DnsFailoverSpec"),
            Map.entry("data_replication", "io.casehub.ops.api.infra.DataReplicationSpec"),
            Map.entry("mesh_control_plane", "io.casehub.ops.api.infra.ServiceMeshControlPlaneSpec"),
            Map.entry("sidecar_proxy", "io.casehub.ops.api.infra.SidecarProxySpec"));

    private TopologyTestHelper() {}

    @SuppressWarnings("unchecked")
    public static DesiredStateGraph compileTopology(String yamlPath) {
        return compileTopologyWithModules(yamlPath, List.of());
    }

    @SuppressWarnings("unchecked")
    public static DesiredStateGraph compileTopologyWithModules(String yamlPath, List<String> modulePaths) {
        try (InputStream is = TopologyTestHelper.class.getClassLoader().getResourceAsStream(yamlPath)) {
            if (is == null) throw new IllegalArgumentException("YAML not found: " + yamlPath);
            YamlGraph yamlGraph = YAML_MAPPER.readValue(is, YamlGraph.class);

            Map<String, YamlModule> modules = new HashMap<>();
            for (String modPath : modulePaths) {
                try (InputStream modIs = TopologyTestHelper.class.getClassLoader().getResourceAsStream(modPath)) {
                    if (modIs == null) throw new IllegalArgumentException("Module not found: " + modPath);
                    YamlModuleFile moduleFile = YAML_MAPPER.readValue(modIs, YamlModuleFile.class);
                    YamlModule module = moduleFile.toModule();
                    modules.put(module.name(), module);
                }
            }

            List<ResolvedInvariant> invariants = new ArrayList<>();
            if (yamlGraph.invariants() != null) {
                for (var inv : yamlGraph.invariants().entrySet()) {
                    invariants.add(YamlInvariantConverter.toDeclarativeInvariant(inv.getKey(), inv.getValue()));
                }
            }

            YamlGraphRecorder recorder = new YamlGraphRecorder();
            GoalCompiler<Void> compiler = (GoalCompiler<Void>) recorder.createYamlGoalCompiler(
                    toGraphDescriptor(yamlGraph, INFRA_TYPE_REGISTRY),
                    INFRA_TYPE_REGISTRY,
                    yamlGraph.variables() != null ? yamlGraph.variables() : Map.of(),
                    invariants, yamlGraph, modules.isEmpty() ? null : modules).getValue();

            return ((CompilationResult.SingleGraph) compiler.compile(null, FACTORY)).graph();
        } catch (Exception e) {
            throw new RuntimeException("Failed to compile topology: " + yamlPath, e);
        }
    }

    public static void assertDependency(DesiredStateGraph graph, String from, String to) {
        var deps = graph.dependenciesOf(NodeId.of(from));
        if (!deps.contains(NodeId.of(to))) {
            throw new AssertionError("Expected dependency " + from + " → " + to
                    + " but dependencies of " + from + " are: " + deps);
        }
    }

    public static <T extends NodeSpec> T nodeSpec(DesiredStateGraph graph, String nodeId, Class<T> type) {
        var node = graph.nodes().get(NodeId.of(nodeId));
        if (node == null) throw new AssertionError("Node not found: " + nodeId
                + ". Available: " + graph.nodes().keySet());
        return type.cast(node.spec());
    }

    private static GraphDescriptor toGraphDescriptor(YamlGraph yamlGraph, Map<String, String> typeRegistry) {
        List<NodeDescriptor> nodes = new ArrayList<>();
        List<DependencyDescriptor> deps = new ArrayList<>();
        for (Map.Entry<String, YamlNode> entry : yamlGraph.nodes().entrySet()) {
            String nodeId = entry.getKey();
            YamlNode yamlNode = entry.getValue();
            String specClassName = typeRegistry.get(yamlNode.type());
            nodes.add(new NodeDescriptor.InlineNode(
                    nodeId, specClassName,
                    yamlNode.spec() != null ? yamlNode.spec() : Map.of(),
                    yamlNode.humanGating()));
            for (String dep : yamlNode.dependencyNodeIds()) {
                deps.add(new DependencyDescriptor(nodeId, dep));
            }
        }
        return new GraphDescriptor(
                yamlGraph.desiredState().namespace(),
                yamlGraph.desiredState().name(),
                null, null, nodes, deps,
                List.of(), null, List.of(), List.of());
    }
}
```

- [ ] **Step 4: Write smoke test**

```java
// topology-tests/src/test/java/io/casehub/ops/topology/TopologySmokeTest.java
package io.casehub.ops.topology;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class TopologySmokeTest {

    @Test
    void typeRegistryCoversAllInfraTypes() {
        assertThat(TopologyTestHelper.INFRA_TYPE_REGISTRY)
                .containsKeys("k8s_namespace", "k8s_deployment", "k8s_service",
                        "load_balancer", "dns_failover", "data_replication",
                        "mesh_control_plane", "sidecar_proxy");
    }
}
```

- [ ] **Step 5: Create directory structure for YAML files**

```bash
mkdir -p topology-tests/src/test/resources/modules
mkdir -p topology-tests/src/test/resources/topologies/single-service
mkdir -p topology-tests/src/test/resources/topologies/multi-tier
mkdir -p topology-tests/src/test/resources/topologies/microservices
mkdir -p topology-tests/src/test/resources/topologies/event-driven
mkdir -p topology-tests/src/test/resources/topologies/sidecar-mesh
```

- [ ] **Step 6: Run build — verify module compiles**

Run: `mvn --batch-mode install -pl topology-tests -am`
Expected: BUILD SUCCESS, smoke test passes

- [ ] **Step 7: Commit**

```bash
git add topology-tests/ pom.xml
git commit -m "feat: scaffold topology-tests module with test helper

TopologyTestHelper provides compileTopology() and
compileTopologyWithModules() using YamlGraphRecorder.
INFRA_TYPE_REGISTRY maps all InfraNodeSpec types for YAML
compilation.

Refs #TBD"
```

---

## Batch 2: Baseline Proof — Single Service + Load Balancer Module

Safe wrap point: simplest topology compiles correctly, load-balancer module works.

### Task 3: Single-Service/Single-Node Exemplar (Core Test 1)

**Files:**
- Create: `topology-tests/src/test/resources/topologies/single-service/single-node-blog.yaml`
- Create: `topology-tests/src/test/java/io/casehub/ops/topology/compilation/SingleServiceCompilationTest.java`

**Interfaces:**
- Consumes: `TopologyTestHelper.compileTopology()`, `K8sNamespaceSpec`, `K8sDeploymentSpec`, `K8sServiceSpec`
- Produces: First working topology exemplar — validates the entire compilation pipeline

- [ ] **Step 1: Write the YAML exemplar**

```yaml
# topology-tests/src/test/resources/topologies/single-service/single-node-blog.yaml
#
# Topology: Single Service / Single Node
# Domain:   Personal dev blog (Ghost)
#
# The simplest possible deployment: one blog engine on one node.
# No load balancer, no replication, no service mesh.
# This is what you run on a Raspberry Pi or a $5 VPS.

desiredState:
  namespace: blog
  name: personal-blog

variables:
  blog_image: ghost:5-alpine
  blog_port: 2368

nodes:
  blog-namespace:
    type: k8s_namespace
    spec:
      name: blog
      labels:
        managed-by: casehub-ops

  ghost-blog:
    type: k8s_deployment
    dependsOn: [blog-namespace]
    spec:
      namespace: blog
      name: ghost-blog
      image: ${var.blog_image}
      replicas: 1
      resources:
        cpu: 250m
        memory: 512Mi
        cpuRequest: 100m
        memoryRequest: 256Mi
      labels:
        app: ghost-blog
        managed-by: casehub-ops
      ports:
        - containerPort: 2368
          servicePort: 2368
          protocol: TCP
      env:
        NODE_ENV: production
        url: http://localhost:2368

  ghost-service:
    type: k8s_service
    dependsOn: [ghost-blog]
    spec:
      namespace: blog
      name: ghost-blog
      port: 2368
      targetPort: 2368
      type: CLUSTER_IP
      labels:
        app: ghost-blog
        managed-by: casehub-ops
      selectorLabels:
        app: ghost-blog
```

- [ ] **Step 2: Write the failing compilation test**

```java
// topology-tests/src/test/java/io/casehub/ops/topology/compilation/SingleServiceCompilationTest.java
package io.casehub.ops.topology.compilation;

import io.casehub.desiredstate.api.DesiredStateGraph;
import io.casehub.desiredstate.api.NodeId;
import io.casehub.ops.api.infra.K8sDeploymentSpec;
import io.casehub.ops.api.infra.K8sNamespaceSpec;
import io.casehub.ops.api.infra.K8sServiceSpec;
import io.casehub.ops.topology.TopologyTestHelper;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static io.casehub.ops.topology.TopologyTestHelper.*;

class SingleServiceCompilationTest {

    private static DesiredStateGraph graph;

    @BeforeAll
    static void compile() {
        graph = compileTopology("topologies/single-service/single-node-blog.yaml");
    }

    @Test
    void threeNodes_namespace_deployment_service() {
        assertThat(graph.nodes()).hasSize(3);
    }

    @Test
    void deploymentDependsOnNamespace() {
        assertDependency(graph, "ghost-blog", "blog-namespace");
    }

    @Test
    void serviceDependsOnDeployment() {
        assertDependency(graph, "ghost-service", "ghost-blog");
    }

    @Test
    void variablesResolvedInDeploymentSpec() {
        var spec = nodeSpec(graph, "ghost-blog", K8sDeploymentSpec.class);
        assertThat(spec.image()).isEqualTo("ghost:5-alpine");
        assertThat(spec.replicas()).isEqualTo(1);
    }

    @Test
    void namespaceHasCorrectName() {
        var spec = nodeSpec(graph, "blog-namespace", K8sNamespaceSpec.class);
        assertThat(spec.name()).isEqualTo("blog");
    }

    @Test
    void serviceTargetsCorrectPort() {
        var spec = nodeSpec(graph, "ghost-service", K8sServiceSpec.class);
        assertThat(spec.port()).isEqualTo(2368);
        assertThat(spec.targetPort()).isEqualTo(2368);
    }
}
```

- [ ] **Step 3: Run test — verify it fails (YAML not found or compilation error)**

Run: `mvn --batch-mode -o test -pl topology-tests -Dtest=SingleServiceCompilationTest`
Expected: FAIL — either file not found or YAML compilation issue. Debug and fix until the pipeline works.

- [ ] **Step 4: Fix any compilation issues and run until green**

This is the critical integration point — first time the topology YAML compiles through
the desiredstate-yaml pipeline with InfraNodeSpec types. Common issues:
- Jackson can't deserialize into the record (missing no-arg constructor or field mismatch)
- `resourceType()` value doesn't match the type registry key
- `Labels` or `ResourceRequirements` don't have Jackson-compatible constructors

Run: `mvn --batch-mode -o test -pl topology-tests -Dtest=SingleServiceCompilationTest`
Expected: all 6 tests PASS

- [ ] **Step 5: Commit**

```bash
git add topology-tests/src/test/resources/topologies/single-service/single-node-blog.yaml \
        topology-tests/src/test/java/io/casehub/ops/topology/compilation/SingleServiceCompilationTest.java
git commit -m "feat: single-service/single-node blog topology — baseline proof

First topology exemplar: Ghost blog on single node. Validates the
full YAML compilation pipeline with InfraNodeSpec types.

Refs #TBD"
```

---

### Task 4: Load-Balancer Module + LB-Cluster Exemplar

**Files:**
- Create: `topology-tests/src/test/resources/modules/load-balancer.yaml`
- Create: `topology-tests/src/test/resources/topologies/single-service/lb-cluster-website.yaml`
- Modify: `topology-tests/src/test/java/io/casehub/ops/topology/compilation/SingleServiceCompilationTest.java` (add LB test)

**Interfaces:**
- Consumes: `TopologyTestHelper.compileTopologyWithModules()`, `LoadBalancerSpec`
- Produces: Working load-balancer module, validated by a company-website topology

- [ ] **Step 1: Write load-balancer module YAML**

```yaml
# topology-tests/src/test/resources/modules/load-balancer.yaml
module:
  name: load-balancer
  parameters:
    target_service:
      type: string
      required: true
    health_check_path:
      type: string
      default: "/health"
    namespace:
      type: string
      required: true
    lb_type:
      type: string
      default: "APPLICATION"

nodes:
  lb:
    type: load_balancer
    spec:
      name: "${var.target_service}-lb"
      namespace: ${var.namespace}
      type: ${var.lb_type}
      healthCheckPath: ${var.health_check_path}
      healthCheckPort: 80
      targetServices: ["${var.target_service}"]
      labels:
        managed-by: casehub-ops
```

- [ ] **Step 2: Write company-website YAML**

```yaml
# topology-tests/src/test/resources/topologies/single-service/lb-cluster-website.yaml
#
# Topology: Single Service / Load-Balanced Cluster
# Domain:   Company marketing website
#
# A static marketing site served by nginx behind a load balancer.
# Three replicas for availability, application LB for health-checked
# traffic distribution. The simplest production-grade deployment.

desiredState:
  namespace: marketing
  name: company-website

variables:
  replicas: 3

nodes:
  marketing-namespace:
    type: k8s_namespace
    spec:
      name: marketing
      labels:
        managed-by: casehub-ops

  company-site:
    type: k8s_deployment
    dependsOn: [marketing-namespace]
    spec:
      namespace: marketing
      name: company-site
      image: nginx:1.25
      replicas: ${var.replicas}
      resources:
        cpu: 250m
        memory: 256Mi
        cpuRequest: 100m
        memoryRequest: 128Mi
      labels:
        app: company-site
        managed-by: casehub-ops
      ports:
        - containerPort: 80
          servicePort: 80
          protocol: TCP
      env: {}
      healthCheck:
        path: /health
        port: 80

  company-site-svc:
    type: k8s_service
    dependsOn: [company-site]
    spec:
      namespace: marketing
      name: company-site
      port: 80
      targetPort: 80
      type: CLUSTER_IP
      labels:
        app: company-site
        managed-by: casehub-ops
      selectorLabels:
        app: company-site

imports:
  - module: load-balancer
    as: website-lb
    parameters:
      target_service: company-site
      namespace: marketing
      health_check_path: /health
```

- [ ] **Step 3: Write the test**

```java
// Add to SingleServiceCompilationTest.java or create a new test class
@Test
void lbClusterWebsite_compilesWithLoadBalancer() {
    var graph = compileTopologyWithModules(
            "topologies/single-service/lb-cluster-website.yaml",
            List.of("modules/load-balancer.yaml"));

    // 4 nodes: namespace + deployment + service + LB (from module)
    assertThat(graph.nodes()).hasSizeGreaterThanOrEqualTo(4);

    // LB node exists
    var lbNode = graph.nodes().values().stream()
            .filter(n -> n.spec() instanceof LoadBalancerSpec)
            .findFirst().orElseThrow(() -> new AssertionError("No LoadBalancerSpec node"));

    var lbSpec = (LoadBalancerSpec) lbNode.spec();
    assertThat(lbSpec.targetServices()).contains("company-site");
    assertThat(lbSpec.healthCheckPath()).isEqualTo("/health");
}
```

- [ ] **Step 4: Run and verify**

Run: `mvn --batch-mode -o test -pl topology-tests`
Expected: all tests pass including the LB-cluster test

- [ ] **Step 5: Commit**

```bash
git add topology-tests/src/test/resources/modules/load-balancer.yaml \
        topology-tests/src/test/resources/topologies/single-service/lb-cluster-website.yaml \
        topology-tests/src/test/java/io/casehub/ops/topology/compilation/SingleServiceCompilationTest.java
git commit -m "feat: load-balancer module + company website topology

Reusable load-balancer YAML module with parameterised target service.
Company marketing website exemplar validates module import compilation.

Refs #TBD"
```

---

## Batch 3: Multi-Tier + Microservices (Core Tests 2-3)

Safe wrap point: lifecycle phases and forEach AZ replication both proven.

### Task 5: Multi-Tier/Single-Node Exemplar (Core Test 2)

**Files:**
- Create: `topology-tests/src/test/resources/topologies/multi-tier/single-node-dev.yaml`
- Create: `topology-tests/src/test/java/io/casehub/ops/topology/compilation/MultiTierCompilationTest.java`

**Interfaces:**
- Consumes: `TopologyTestHelper.compileTopology()`, `K8sDeploymentSpec`
- Produces: Multi-tier topology with lifecycle phases proving ordered rollout

- [ ] **Step 1: Write multi-tier dev stack YAML with lifecycle phases**

A local dev stack for an e-commerce application: nginx → catalog-api → postgres.
Uses lifecycle phases to order: infrastructure → data → application → web.

```yaml
# topology-tests/src/test/resources/topologies/multi-tier/single-node-dev.yaml
#
# Topology: Multi-Tier / Single Node
# Domain:   Local development stack for an e-commerce storefront
#
# The classic three-tier architecture running on a developer's laptop.
# Lifecycle phases ensure infrastructure (namespace) provisions first,
# then the database, then the API, then the web front-end.
# Each phase completes before the next begins.

desiredState:
  namespace: ecommerce-dev
  name: local-dev-stack

variables:
  db_password: dev-only-password
  api_port: 8080

lifecycle:
  phases:
    - id: infrastructure
      completionCondition: allPresent
      nodes:
        dev-namespace:
          type: k8s_namespace
          spec:
            name: ecommerce-dev
            labels:
              managed-by: casehub-ops
              environment: dev

    - id: data-tier
      completionCondition: allPresent
      nodes:
        product-db:
          type: k8s_deployment
          dependsOn: [dev-namespace]
          spec:
            namespace: ecommerce-dev
            name: product-db
            image: postgres:16
            replicas: 1
            resources:
              cpu: 500m
              memory: 1Gi
              cpuRequest: 250m
              memoryRequest: 512Mi
            labels:
              app: product-db
              tier: data
            ports:
              - containerPort: 5432
                servicePort: 5432
                protocol: TCP
            env:
              POSTGRES_DB: catalog
              POSTGRES_PASSWORD: ${var.db_password}

    - id: application-tier
      completionCondition: allPresent
      nodes:
        catalog-api:
          type: k8s_deployment
          dependsOn: [product-db]
          spec:
            namespace: ecommerce-dev
            name: catalog-api
            image: ecom/catalog-api:dev
            replicas: 1
            resources:
              cpu: 500m
              memory: 1Gi
              cpuRequest: 250m
              memoryRequest: 512Mi
            labels:
              app: catalog-api
              tier: application
            ports:
              - containerPort: 8080
                servicePort: 8080
                protocol: TCP
            env:
              DB_HOST: product-db
              DB_NAME: catalog
            healthCheck:
              path: /health
              port: 8080

    - id: web-tier
      completionCondition: allPresent
      nodes:
        storefront-nginx:
          type: k8s_deployment
          dependsOn: [catalog-api]
          spec:
            namespace: ecommerce-dev
            name: storefront-nginx
            image: nginx:1.25
            replicas: 1
            resources:
              cpu: 250m
              memory: 256Mi
              cpuRequest: 100m
              memoryRequest: 128Mi
            labels:
              app: storefront-nginx
              tier: web
            ports:
              - containerPort: 80
                servicePort: 80
                protocol: TCP
            env:
              API_URL: http://catalog-api:8080
```

- [ ] **Step 2: Write the compilation test**

Note: lifecycle compilation returns `CompilationResult.Lifecycle` with phases.
Update `TopologyTestHelper` to handle this:

```java
// Add to TopologyTestHelper.java:
@SuppressWarnings("unchecked")
public static CompilationResult compileTopologyResult(String yamlPath) {
    // Similar to compileTopology but returns CompilationResult directly
    // so callers can check for SingleGraph vs Lifecycle
    // ... (implementation similar to compileTopology but returns the raw result)
}
```

```java
// topology-tests/src/test/java/io/casehub/ops/topology/compilation/MultiTierCompilationTest.java
package io.casehub.ops.topology.compilation;

import io.casehub.desiredstate.api.CompilationResult;
import io.casehub.desiredstate.api.DesiredStateGraph;
import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.api.Phase;
import io.casehub.ops.api.infra.K8sDeploymentSpec;
import io.casehub.ops.topology.TopologyTestHelper;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static io.casehub.ops.topology.TopologyTestHelper.*;

class MultiTierCompilationTest {

    private static List<Phase> phases;

    @BeforeAll
    static void compile() {
        // Use lifecycle compilation path
        phases = compileLifecycleTopology("topologies/multi-tier/single-node-dev.yaml");
    }

    @Test
    void fourLifecyclePhases() {
        assertThat(phases).hasSize(4);
        assertThat(phases.get(0).id()).isEqualTo("infrastructure");
        assertThat(phases.get(1).id()).isEqualTo("data-tier");
        assertThat(phases.get(2).id()).isEqualTo("application-tier");
        assertThat(phases.get(3).id()).isEqualTo("web-tier");
    }

    @Test
    void webTierPhaseContainsAllPreviousNodes() {
        DesiredStateGraph finalGraph = phases.get(3).graph();
        assertThat(finalGraph.nodes()).containsKey(NodeId.of("dev-namespace"));
        assertThat(finalGraph.nodes()).containsKey(NodeId.of("product-db"));
        assertThat(finalGraph.nodes()).containsKey(NodeId.of("catalog-api"));
        assertThat(finalGraph.nodes()).containsKey(NodeId.of("storefront-nginx"));
    }

    @Test
    void dependencyChain_web_app_data() {
        DesiredStateGraph finalGraph = phases.get(3).graph();
        assertDependency(finalGraph, "storefront-nginx", "catalog-api");
        assertDependency(finalGraph, "catalog-api", "product-db");
        assertDependency(finalGraph, "product-db", "dev-namespace");
    }

    @Test
    void variablesResolvedInDbSpec() {
        DesiredStateGraph dataGraph = phases.get(1).graph();
        var dbSpec = nodeSpec(dataGraph, "product-db", K8sDeploymentSpec.class);
        assertThat(dbSpec.env()).containsEntry("POSTGRES_PASSWORD", "dev-only-password");
    }
}
```

- [ ] **Step 3: Add lifecycle compilation to TopologyTestHelper**

Add `compileLifecycleTopology()` that uses `YamlGraphRecorder.createYamlLifecycleGoalCompiler()` and returns `List<Phase>`.

- [ ] **Step 4: Run and verify**

Run: `mvn --batch-mode -o test -pl topology-tests -Dtest=MultiTierCompilationTest`
Expected: all tests pass

- [ ] **Step 5: Commit**

```bash
git add topology-tests/
git commit -m "feat: multi-tier/single-node dev stack topology with lifecycle phases

Four-phase ordered rollout: infrastructure → data → application → web.
Proves lifecycle phase compilation with carry-forward nodes.

Refs #TBD"
```

---

### Task 6: Microservices/HA-Multi-AZ Exemplar (Core Test 3)

**Files:**
- Create: `topology-tests/src/test/resources/modules/ha-multi-az.yaml`
- Create: `topology-tests/src/test/resources/topologies/microservices/ha-multi-az-trading.yaml`
- Create: `topology-tests/src/test/java/io/casehub/ops/topology/compilation/MicroservicesCompilationTest.java`

**Interfaces:**
- Consumes: `TopologyTestHelper.compileTopologyWithModules()`, `K8sDeploymentSpec`, `LoadBalancerSpec`
- Produces: forEach-expanded microservices across 3 AZs with auto-wiring rules

- [ ] **Step 1: Write ha-multi-az module YAML**

Uses the module spec from the design doc §5.2.

- [ ] **Step 2: Write trading platform YAML**

A simplified equities trading platform with 4 services (pricing, order-routing,
risk-engine, settlement) stamped across 3 AZs via forEach. An auto-wiring rule
creates a ClusterIP service for every deployment.

- [ ] **Step 3: Write compilation test**

Verify forEach produces 12 deployment nodes (4 services × 3 AZs), auto-wiring
rule adds 12 service nodes, dependencies are correct across AZ copies.

- [ ] **Step 4: Run and verify**

Run: `mvn --batch-mode -o test -pl topology-tests -Dtest=MicroservicesCompilationTest`
Expected: all tests pass

- [ ] **Step 5: Commit**

```bash
git add topology-tests/
git commit -m "feat: microservices/HA-multi-AZ trading platform topology

forEach stamps 4 services across 3 AZs. Auto-wiring rule adds
ClusterIP services. ha-multi-az module provides HA infra.

Refs #TBD"
```

---

## Batch 4: Event-Driven + Sidecar/Mesh (Core Tests 4-5)

Safe wrap point: all 5 core test intersections proven. All YAML primitives exercised.

### Task 7: Event-Driven/LB-Cluster + Sidecar-Mesh/LB-Cluster (Core Tests 4-5)

**Files:**
- Create: `topology-tests/src/test/resources/modules/service-mesh.yaml`
- Create: `topology-tests/src/test/resources/topologies/event-driven/lb-cluster-telemetry.yaml`
- Create: `topology-tests/src/test/resources/topologies/sidecar-mesh/lb-cluster-logistics.yaml`
- Create: `topology-tests/src/test/java/io/casehub/ops/topology/compilation/EventDrivenCompilationTest.java`
- Create: `topology-tests/src/test/java/io/casehub/ops/topology/compilation/SidecarMeshCompilationTest.java`

**Interfaces:**
- Consumes: `TopologyTestHelper`, `SidecarProxySpec`, `ServiceMeshControlPlaneSpec`, `LoadBalancerSpec`
- Produces: All 5 core topology intersections proven

- [ ] **Step 1: Write service-mesh module YAML**

Uses the module spec from the design doc §5.4, including the sidecar-injection rule
and mesh-requires-control-plane invariant.

- [ ] **Step 2: Write IoT telemetry pipeline YAML**

An event-driven pipeline: sensor-ingestion → RabbitMQ broker → anomaly-detector +
timeseries-writer → timeseries-db. Invariant enforces all non-broker services depend
on the broker.

- [ ] **Step 3: Write logistics fleet tracking YAML**

Four services (fleet-tracker, route-optimizer, parcel-service, warehouse-api) with
the service-mesh module imported. The sidecar-injection rule auto-adds proxy nodes.

- [ ] **Step 4: Write event-driven test**

Verify broker node exists, all services depend on it, invariant catches a missing
broker dependency if one is removed.

- [ ] **Step 5: Write sidecar-mesh test**

Verify mesh control plane exists, sidecar proxy nodes are auto-injected (4 proxy
nodes for 4 services), proxy nodes depend on both the service and the control plane.

- [ ] **Step 6: Run and verify all 5 core tests**

Run: `mvn --batch-mode -o test -pl topology-tests`
Expected: all tests pass — all 5 YAML primitives exercised (forEach, modules, rules,
invariants, lifecycle phases)

- [ ] **Step 7: Commit**

```bash
git add topology-tests/
git commit -m "feat: event-driven telemetry + sidecar-mesh logistics topologies

Completes all 5 core test intersections. Event-driven validates broker
dependency invariants. Sidecar-mesh validates auto-injection rules and
control plane requirement invariant.

Refs #TBD"
```

---

## Batch 5: Tutorial Exemplars

Safe wrap point: full 14-topology matrix. All exemplars compile. Tutorial material ready.

### Task 8: Tutorial Exemplars (9 remaining topologies)

**Files:**
- Create: 9 YAML exemplar files under `topology-tests/src/test/resources/topologies/`
- Create: `topology-tests/src/test/java/io/casehub/ops/topology/compilation/TutorialExemplarCompilationTest.java`

**Interfaces:**
- Consumes: `TopologyTestHelper`, all modules from Batch 2-4
- Produces: Complete 14-topology matrix with compilation tests

Exemplar list:
1. `multi-tier/lb-cluster-ecommerce.yaml` — E-commerce storefront with LB
2. `multi-tier/ha-multi-az-healthcare.yaml` — Hospital records system
3. `multi-tier/multi-region-banking.yaml` — Retail banking with DR
4. `microservices/single-node-dev.yaml` — Food delivery local dev
5. `microservices/lb-cluster-delivery.yaml` — Food delivery production
6. `microservices/multi-region-payments.yaml` — Global payments
7. `event-driven/single-node-dev.yaml` — Telemetry local dev
8. `sidecar-mesh/ha-multi-az-insurance.yaml` — Insurance claims
9. `single-service/lb-cluster-website.yaml` — already exists from Task 4

- [ ] **Step 1: Write 8 YAML exemplar files**

Each file follows the pattern established in core tests. Rich domain-specific comments
explain what the topology represents and why each choice was made. Tutorial-first:
readability over test completeness.

- [ ] **Step 2: Write parameterised compilation test**

```java
@ParameterizedTest
@MethodSource("allExemplars")
void exemplarCompiles(String yamlPath, List<String> modules, int minNodes) {
    DesiredStateGraph graph;
    if (modules.isEmpty()) {
        graph = compileTopology(yamlPath);
    } else {
        graph = compileTopologyWithModules(yamlPath, modules);
    }
    assertThat(graph.nodes()).hasSizeGreaterThanOrEqualTo(minNodes);
}
```

- [ ] **Step 3: Run full test suite**

Run: `mvn --batch-mode -o test -pl topology-tests`
Expected: all 14 topology exemplars compile successfully

- [ ] **Step 4: Commit**

```bash
git add topology-tests/
git commit -m "feat: complete 14-topology matrix with tutorial exemplars

9 tutorial exemplars spanning multi-tier (ecommerce, healthcare,
banking), microservices (delivery, payments), event-driven (dev),
and sidecar-mesh (insurance) topologies. All compile successfully.

Refs #TBD"
```

---

## References

- [2026-08-29-canonical-deployment-topologies-design.md] — design spec this plan implements
- [api/src/main/java/io/casehub/ops/api/infra/InfraNodeSpec.java] — sealed hierarchy to extend
- [api/src/main/java/io/casehub/ops/api/infra/K8sDeploymentSpec.java] — pattern for new record types
- [desiredstate/examples/webapp-yaml/] — test infrastructure pattern (TutorialTestHelper, YamlGraphRecorder usage)
- [desiredstate/yaml/runtime/YamlGraphRecorder.java] — YAML compilation pipeline
- [desiredstate/yaml/runtime/ForEachExpander.java] — forEach expansion
- [desiredstate/yaml/runtime/ModuleExpander.java] — module import expansion
- [docs/research/2026-08-29-canonical-deployment-topologies.md] — full research with web sources
