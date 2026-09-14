# Cross-Domain Dependency Graphs Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** casehubio/casehub-ops#23 — cross-domain dependency graphs
**Issue group:** casehubio/casehub-desiredstate#140, casehubio/casehub-ops#23

**Goal:** Enable cross-domain dependency ordering so domains like infra, deployment, compliance, and IoT can declare and enforce provisioning order through a unified orchestration layer.

**Architecture:** The orchestrator discovers `DomainRegistration` beans via CDI, calls each domain's `compile()`, and merges per-domain graphs into a single `DesiredStateGraph` using `sequenceAfter()` (coarse ordering) and `withDependency()` (fine-grained edges). Cross-domain dependencies are declared in a `domains:` YAML block. The merged graph feeds one `ReconciliationLoop` per tenant — existing routers handle multi-type dispatch.

**Tech Stack:** Java 21, Quarkus (CDI), Maven, casehub-desiredstate-api/runtime, casehub-ops modules

## Global Constraints

- All new SPI interfaces go in `casehub-desiredstate-api` (`api/` module)
- All runtime implementations go in `casehub-desiredstate-runtime` (`runtime/` module)
- Domain modules must ensure disjoint `NodeType` sets — routers throw on duplicates
- `DomainOrchestrator` requires `CompilationResult.SingleGraph` — rejects `Lifecycle`
- Export identifiers are stable aliases, not internal `NodeId` values
- `sequenceAfter()` computes roots/leaves on original graphs before `overlay()`

---

## Batch 1: Graph Composition + DomainRegistration SPI [desiredstate#140]

### Task 1: Add `sequenceAfter()` to DesiredStateGraph

**Repo:** casehub-desiredstate
**Files:**
- Modify: `api/src/main/java/io/casehub/desiredstate/api/DesiredStateGraph.java`
- Modify: `runtime/src/main/java/io/casehub/desiredstate/runtime/ImmutableDesiredStateGraph.java`
- Create: `runtime/src/test/java/io/casehub/desiredstate/runtime/ImmutableDesiredStateGraphSequenceAfterTest.java`

**Interfaces:**
- Consumes: `DesiredStateGraph.overlay()`, `DesiredStateGraph.withDependency()`, `DesiredStateGraph.roots()`, `DesiredStateGraph.leaves()`
- Produces: `DesiredStateGraph.sequenceAfter(DesiredStateGraph upstream)` — returns a new graph containing nodes from both graphs, with `Dependency(thisRoot, upstreamLeaf)` for each (root, leaf) pair

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.desiredstate.runtime;

import io.casehub.desiredstate.api.*;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class ImmutableDesiredStateGraphSequenceAfterTest {

    private final DesiredStateGraphFactory factory = new DefaultDesiredStateGraphFactory();

    @Test
    void sequenceAfter_creates_edges_from_downstream_roots_to_upstream_leaves() {
        // Upstream: A → B (A=root, B=leaf)
        DesiredNode nodeA = new DesiredNode(NodeId.of("a"), stubSpec("infra"), HumanGating.NONE);
        DesiredNode nodeB = new DesiredNode(NodeId.of("b"), stubSpec("infra"), HumanGating.NONE);
        DesiredStateGraph upstream = factory.of(
            java.util.List.of(nodeA, nodeB),
            java.util.List.of(new Dependency(NodeId.of("b"), NodeId.of("a")))
        );

        // Downstream: C → D (C=root, D=leaf)
        DesiredNode nodeC = new DesiredNode(NodeId.of("c"), stubSpec("deploy"), HumanGating.NONE);
        DesiredNode nodeD = new DesiredNode(NodeId.of("d"), stubSpec("deploy"), HumanGating.NONE);
        DesiredStateGraph downstream = factory.of(
            java.util.List.of(nodeC, nodeD),
            java.util.List.of(new Dependency(NodeId.of("d"), NodeId.of("c")))
        );

        DesiredStateGraph merged = downstream.sequenceAfter(upstream);

        // All 4 nodes present
        assertEquals(4, merged.nodes().size());

        // Downstream root C depends on upstream leaf B
        assertTrue(merged.dependenciesOf(NodeId.of("c")).contains(NodeId.of("b")));

        // Original edges preserved
        assertTrue(merged.dependenciesOf(NodeId.of("b")).contains(NodeId.of("a")));
        assertTrue(merged.dependenciesOf(NodeId.of("d")).contains(NodeId.of("c")));
    }

    @Test
    void sequenceAfter_uses_original_roots_leaves_not_overlaid() {
        // Upstream: A (root+leaf, single node)
        DesiredNode nodeA = new DesiredNode(NodeId.of("a"), stubSpec("infra"), HumanGating.NONE);
        DesiredStateGraph upstream = factory.of(java.util.List.of(nodeA), java.util.List.of());

        // Downstream: B (root+leaf, single node)
        DesiredNode nodeB = new DesiredNode(NodeId.of("b"), stubSpec("deploy"), HumanGating.NONE);
        DesiredStateGraph downstream = factory.of(java.util.List.of(nodeB), java.util.List.of());

        DesiredStateGraph merged = downstream.sequenceAfter(upstream);

        // B depends on A (downstream root → upstream leaf)
        assertTrue(merged.dependenciesOf(NodeId.of("b")).contains(NodeId.of("a")));
        // NOT a cycle — A does not depend on B
        assertFalse(merged.dependenciesOf(NodeId.of("a")).contains(NodeId.of("b")));
    }

    @Test
    void sequenceAfter_detects_cycles_from_conflicting_edges() {
        // Upstream: A depends on B
        DesiredNode nodeA = new DesiredNode(NodeId.of("a"), stubSpec("infra"), HumanGating.NONE);
        DesiredNode nodeB = new DesiredNode(NodeId.of("b"), stubSpec("deploy"), HumanGating.NONE);
        DesiredStateGraph upstream = factory.of(
            java.util.List.of(nodeA, nodeB),
            java.util.List.of(new Dependency(NodeId.of("a"), NodeId.of("b")))
        );

        // Downstream has B as root — but B is already in upstream
        // This would create a cycle: B depends on A (sequenceAfter), A depends on B (upstream)
        DesiredStateGraph downstream = factory.of(
            java.util.List.of(nodeB),
            java.util.List.of()
        );

        // overlay() throws on conflicting nodes (same id, same content = ok, different = throws)
        // sequenceAfter with shared node B: B is root of downstream AND leaf of upstream
        // Edge: Dependency(B, A) from sequenceAfter + Dependency(A, B) from upstream = cycle
        assertThrows(CyclicDependencyException.class, () -> downstream.sequenceAfter(upstream));
    }

    private NodeSpec stubSpec(String type) {
        return new NodeSpec() {
            @Override public NodeType type() { return NodeType.of(type); }
        };
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl runtime -Dtest=ImmutableDesiredStateGraphSequenceAfterTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — `sequenceAfter` method does not exist

- [ ] **Step 3: Add `sequenceAfter()` to `DesiredStateGraph` interface**

Add default method to `DesiredStateGraph.java`:

```java
DesiredStateGraph sequenceAfter(DesiredStateGraph upstream);
```

- [ ] **Step 4: Implement `sequenceAfter()` in `ImmutableDesiredStateGraph`**

```java
@Override
public DesiredStateGraph sequenceAfter(DesiredStateGraph upstream) {
    // Compute roots/leaves on ORIGINAL graphs before overlay
    Set<NodeId> downstreamRoots = this.roots();
    Set<NodeId> upstreamLeaves = upstream.leaves();

    // Overlay nodes and edges
    DesiredStateGraph overlaid = this.overlay(upstream);

    // Add edges: every downstream root depends on every upstream leaf
    DesiredStateGraph result = overlaid;
    for (NodeId root : downstreamRoots) {
        for (NodeId leaf : upstreamLeaves) {
            result = result.withDependency(new Dependency(root, leaf));
        }
    }
    return result;
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl runtime -Dtest=ImmutableDesiredStateGraphSequenceAfterTest`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add api/src/main/java/io/casehub/desiredstate/api/DesiredStateGraph.java \
        runtime/src/main/java/io/casehub/desiredstate/runtime/ImmutableDesiredStateGraph.java \
        runtime/src/test/java/io/casehub/desiredstate/runtime/ImmutableDesiredStateGraphSequenceAfterTest.java
git commit -m "feat: add sequenceAfter() for strict cross-domain ordering Refs #140"
```

---

### Task 2: Create DomainRegistration SPI + export validation exceptions

**Repo:** casehub-desiredstate
**Files:**
- Create: `api/src/main/java/io/casehub/desiredstate/api/DomainRegistration.java`
- Create: `api/src/main/java/io/casehub/desiredstate/api/InvalidExportException.java`
- Create: `api/src/main/java/io/casehub/desiredstate/api/UnresolvableEdgeException.java`
- Create: `api/src/test/java/io/casehub/desiredstate/api/DomainRegistrationTest.java`

**Interfaces:**
- Consumes: `CompilationResult`, `DesiredStateGraphFactory`, `NodeId`
- Produces: `DomainRegistration` interface with `domainId()`, `compile(String, DesiredStateGraphFactory)`, `exportedNodes()`; `InvalidExportException`; `UnresolvableEdgeException`

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.desiredstate.api;

import org.junit.jupiter.api.Test;
import java.util.Map;
import static org.junit.jupiter.api.Assertions.*;

class DomainRegistrationTest {

    @Test
    void domainRegistration_provides_identity_and_exports() {
        DomainRegistration reg = new DomainRegistration() {
            @Override public String domainId() { return "infra"; }
            @Override public CompilationResult compile(String goalsPath, DesiredStateGraphFactory factory) {
                return CompilationResult.single(factory.of(java.util.List.of(), java.util.List.of()));
            }
            @Override public Map<String, NodeId> exportedNodes() {
                return Map.of("namespace-ready", NodeId.of("k8s-ns-prod"));
            }
        };

        assertEquals("infra", reg.domainId());
        assertEquals(NodeId.of("k8s-ns-prod"), reg.exportedNodes().get("namespace-ready"));
    }

    @Test
    void invalidExportException_carries_domain_and_export_info() {
        var ex = new InvalidExportException("infra", "namespace-ready", NodeId.of("k8s-ns-prod"));
        assertTrue(ex.getMessage().contains("infra"));
        assertTrue(ex.getMessage().contains("namespace-ready"));
    }

    @Test
    void unresolvableEdgeException_carries_edge_info() {
        var ex = new UnresolvableEdgeException("deployment", "agent-bootstrap", "infra", "missing-export");
        assertTrue(ex.getMessage().contains("agent-bootstrap"));
        assertTrue(ex.getMessage().contains("missing-export"));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl api -Dtest=DomainRegistrationTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — classes do not exist

- [ ] **Step 3: Create `DomainRegistration` interface**

```java
package io.casehub.desiredstate.api;

import java.util.Map;

public interface DomainRegistration {
    String domainId();
    CompilationResult compile(String goalsPath, DesiredStateGraphFactory factory);
    Map<String, NodeId> exportedNodes();
}
```

- [ ] **Step 4: Create `InvalidExportException`**

```java
package io.casehub.desiredstate.api;

public class InvalidExportException extends RuntimeException {
    public InvalidExportException(String domainId, String exportName, NodeId nodeId) {
        super("Domain '" + domainId + "' exports '" + exportName
              + "' → " + nodeId.value() + " but that node does not exist in the compiled graph");
    }
}
```

- [ ] **Step 5: Create `UnresolvableEdgeException`**

```java
package io.casehub.desiredstate.api;

public class UnresolvableEdgeException extends RuntimeException {
    public UnresolvableEdgeException(String fromDomain, String fromNode,
                                      String toDomain, String toExport) {
        super("Edge from '" + fromDomain + ":" + fromNode
              + "' to '" + toDomain + ":" + toExport
              + "' — export '" + toExport + "' not found in domain '" + toDomain + "'");
    }
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl api -Dtest=DomainRegistrationTest`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add api/src/main/java/io/casehub/desiredstate/api/DomainRegistration.java \
        api/src/main/java/io/casehub/desiredstate/api/InvalidExportException.java \
        api/src/main/java/io/casehub/desiredstate/api/UnresolvableEdgeException.java \
        api/src/test/java/io/casehub/desiredstate/api/DomainRegistrationTest.java
git commit -m "feat: add DomainRegistration SPI and export validation exceptions Refs #140"
```

---

## Batch 2: DomainOrchestrator [desiredstate#140]

### Task 3: Create DomainOrchestrator — graph merging + export validation

**Repo:** casehub-desiredstate
**Files:**
- Create: `runtime/src/main/java/io/casehub/desiredstate/runtime/DomainOrchestrator.java`
- Create: `runtime/src/main/java/io/casehub/desiredstate/runtime/OrchestrationConfig.java`
- Create: `runtime/src/test/java/io/casehub/desiredstate/runtime/DomainOrchestratorTest.java`

**Interfaces:**
- Consumes: `DomainRegistration`, `DesiredStateGraphFactory`, `DesiredStateGraph.sequenceAfter()`, `DesiredStateGraph.overlay()`, `DesiredStateGraph.withDependency()`, `CompilationResult.SingleGraph`, `InvalidExportException`, `UnresolvableEdgeException`
- Produces: `DomainOrchestrator.orchestrate(OrchestrationConfig config, DesiredStateGraphFactory factory)` → `DesiredStateGraph` (merged graph); `OrchestrationConfig` record with domain ordering and edges

- [ ] **Step 1: Write the failing test — basic two-domain merge with coarse ordering**

```java
package io.casehub.desiredstate.runtime;

import io.casehub.desiredstate.api.*;
import org.junit.jupiter.api.Test;
import java.util.*;
import static org.junit.jupiter.api.Assertions.*;

class DomainOrchestratorTest {

    private final DesiredStateGraphFactory factory = new DefaultDesiredStateGraphFactory();

    @Test
    void orchestrate_merges_two_domains_with_coarse_ordering() {
        DomainRegistration infra = stubRegistration("infra",
            List.of(node("ns", "infra")), List.of(),
            Map.of("namespace-ready", NodeId.of("ns")));

        DomainRegistration deployment = stubRegistration("deployment",
            List.of(node("agent", "deploy")), List.of(),
            Map.of());

        var config = new OrchestrationConfig(
            List.of(
                new OrchestrationConfig.DomainEntry("infra", "goals/infra.yaml", List.of(), List.of()),
                new OrchestrationConfig.DomainEntry("deployment", "goals/deploy.yaml",
                    List.of("infra"), List.of())
            )
        );

        var orchestrator = new DomainOrchestrator(Map.of("infra", infra, "deployment", deployment));
        DesiredStateGraph merged = orchestrator.orchestrate(config, factory);

        assertEquals(2, merged.nodes().size());
        // deployment root "agent" depends on infra leaf "ns"
        assertTrue(merged.dependenciesOf(NodeId.of("agent")).contains(NodeId.of("ns")));
    }

    @Test
    void orchestrate_validates_exports_against_compiled_graph() {
        DomainRegistration bad = stubRegistration("infra",
            List.of(node("actual-node", "infra")), List.of(),
            Map.of("namespace-ready", NodeId.of("stale-node-id")));

        var config = new OrchestrationConfig(List.of(
            new OrchestrationConfig.DomainEntry("infra", "goals/infra.yaml", List.of(), List.of())
        ));

        var orchestrator = new DomainOrchestrator(Map.of("infra", bad));
        assertThrows(InvalidExportException.class, () -> orchestrator.orchestrate(config, factory));
    }

    @Test
    void orchestrate_adds_fine_grained_cross_domain_edges() {
        DomainRegistration infra = stubRegistration("infra",
            List.of(node("ns", "infra")), List.of(),
            Map.of("namespace-ready", NodeId.of("ns")));

        DomainRegistration deploy = stubRegistration("deployment",
            List.of(node("agent", "deploy"), node("channel", "deploy")),
            List.of(new Dependency(NodeId.of("channel"), NodeId.of("agent"))),
            Map.of());

        var config = new OrchestrationConfig(List.of(
            new OrchestrationConfig.DomainEntry("infra", "goals/infra.yaml", List.of(), List.of()),
            new OrchestrationConfig.DomainEntry("deployment", "goals/deploy.yaml", List.of(),
                List.of(new OrchestrationConfig.EdgeEntry("agent", "infra", "namespace-ready")))
        ));

        var orchestrator = new DomainOrchestrator(Map.of("infra", infra, "deployment", deploy));
        DesiredStateGraph merged = orchestrator.orchestrate(config, factory);

        assertEquals(3, merged.nodes().size());
        // Fine-grained: agent depends on ns
        assertTrue(merged.dependenciesOf(NodeId.of("agent")).contains(NodeId.of("ns")));
        // Intra-domain: channel depends on agent (preserved)
        assertTrue(merged.dependenciesOf(NodeId.of("channel")).contains(NodeId.of("agent")));
    }

    @Test
    void orchestrate_rejects_lifecycle_compilation_result() {
        DomainRegistration lifecycle = new DomainRegistration() {
            @Override public String domainId() { return "bad"; }
            @Override public CompilationResult compile(String goalsPath, DesiredStateGraphFactory f) {
                return CompilationResult.lifecycle(List.of(
                    new Phase("p1", f.of(List.of(), List.of()), CompletionCondition.allPresent())
                ));
            }
            @Override public Map<String, NodeId> exportedNodes() { return Map.of(); }
        };

        var config = new OrchestrationConfig(List.of(
            new OrchestrationConfig.DomainEntry("bad", "goals/bad.yaml", List.of(), List.of())
        ));

        var orchestrator = new DomainOrchestrator(Map.of("bad", lifecycle));
        assertThrows(IllegalArgumentException.class, () -> orchestrator.orchestrate(config, factory));
    }

    private DesiredNode node(String id, String type) {
        return new DesiredNode(NodeId.of(id), new NodeSpec() {
            @Override public NodeType type() { return NodeType.of(type); }
        }, HumanGating.NONE);
    }

    private DomainRegistration stubRegistration(String domainId, List<DesiredNode> nodes,
                                                 List<Dependency> deps, Map<String, NodeId> exports) {
        return new DomainRegistration() {
            @Override public String domainId() { return domainId; }
            @Override public CompilationResult compile(String goalsPath, DesiredStateGraphFactory f) {
                return CompilationResult.single(f.of(nodes, deps));
            }
            @Override public Map<String, NodeId> exportedNodes() { return exports; }
        };
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl runtime -Dtest=DomainOrchestratorTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — classes do not exist

- [ ] **Step 3: Create `OrchestrationConfig` record**

```java
package io.casehub.desiredstate.runtime;

import java.util.List;

public record OrchestrationConfig(List<DomainEntry> domains) {

    public record DomainEntry(
        String domainId,
        String goalsPath,
        List<String> dependsOn,
        List<EdgeEntry> edges
    ) {}

    public record EdgeEntry(
        String fromNode,
        String toDomain,
        String toExport
    ) {}
}
```

- [ ] **Step 4: Create `DomainOrchestrator`**

```java
package io.casehub.desiredstate.runtime;

import io.casehub.desiredstate.api.*;
import java.util.*;

public class DomainOrchestrator {

    private final Map<String, DomainRegistration> registrations;

    public DomainOrchestrator(Map<String, DomainRegistration> registrations) {
        this.registrations = Map.copyOf(registrations);
    }

    public DesiredStateGraph orchestrate(OrchestrationConfig config, DesiredStateGraphFactory factory) {
        // Phase 1: Compile each domain
        Map<String, DesiredStateGraph> domainGraphs = new LinkedHashMap<>();
        for (OrchestrationConfig.DomainEntry entry : config.domains()) {
            DomainRegistration reg = registrations.get(entry.domainId());
            if (reg == null) {
                throw new IllegalArgumentException(
                    "No DomainRegistration found for domain: " + entry.domainId());
            }

            CompilationResult result = reg.compile(entry.goalsPath(), factory);
            if (!(result instanceof CompilationResult.SingleGraph single)) {
                throw new IllegalArgumentException(
                    "Domain '" + entry.domainId()
                    + "' returned Lifecycle compilation result — multi-domain orchestration requires SingleGraph");
            }
            domainGraphs.put(entry.domainId(), single.graph());
        }

        // Phase 2: Validate exports
        for (OrchestrationConfig.DomainEntry entry : config.domains()) {
            DomainRegistration reg = registrations.get(entry.domainId());
            DesiredStateGraph graph = domainGraphs.get(entry.domainId());
            for (Map.Entry<String, NodeId> export : reg.exportedNodes().entrySet()) {
                if (!graph.nodes().containsKey(export.getValue())) {
                    throw new InvalidExportException(
                        entry.domainId(), export.getKey(), export.getValue());
                }
            }
        }

        // Phase 3: Merge graphs with coarse ordering
        DesiredStateGraph merged = null;
        for (OrchestrationConfig.DomainEntry entry : config.domains()) {
            DesiredStateGraph domainGraph = domainGraphs.get(entry.domainId());

            if (merged == null) {
                merged = domainGraph;
                continue;
            }

            if (!entry.dependsOn().isEmpty()) {
                // Coarse ordering: sequenceAfter each dependency
                for (String upstreamId : entry.dependsOn()) {
                    DesiredStateGraph upstreamGraph = domainGraphs.get(upstreamId);
                    if (upstreamGraph == null) {
                        throw new IllegalArgumentException(
                            "Domain '" + entry.domainId() + "' depends on '"
                            + upstreamId + "' which is not declared");
                    }
                    domainGraph = domainGraph.sequenceAfter(upstreamGraph);
                }
                // Overlay onto merged (sequenceAfter already included upstream nodes)
                // Re-overlay to pick up nodes from domains not in dependsOn
                DesiredStateGraph temp = merged;
                for (var node : domainGraph.nodes().values()) {
                    if (!temp.nodes().containsKey(node.id())) {
                        temp = temp.withNode(node);
                    }
                }
                for (var dep : domainGraph.dependencies()) {
                    if (!temp.dependencies().contains(dep)) {
                        temp = temp.withDependency(dep);
                    }
                }
                merged = temp;
            } else {
                merged = merged.overlay(domainGraph);
            }
        }

        // Phase 4: Add fine-grained cross-domain edges
        for (OrchestrationConfig.DomainEntry entry : config.domains()) {
            for (OrchestrationConfig.EdgeEntry edge : entry.edges()) {
                NodeId fromNode = NodeId.of(edge.fromNode());
                DomainRegistration toDomainReg = registrations.get(edge.toDomain());
                if (toDomainReg == null) {
                    throw new UnresolvableEdgeException(
                        entry.domainId(), edge.fromNode(), edge.toDomain(), edge.toExport());
                }
                NodeId toNode = toDomainReg.exportedNodes().get(edge.toExport());
                if (toNode == null) {
                    throw new UnresolvableEdgeException(
                        entry.domainId(), edge.fromNode(), edge.toDomain(), edge.toExport());
                }
                merged = merged.withDependency(new Dependency(fromNode, toNode));
            }
        }

        return merged;
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl runtime -Dtest=DomainOrchestratorTest`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add runtime/src/main/java/io/casehub/desiredstate/runtime/DomainOrchestrator.java \
        runtime/src/main/java/io/casehub/desiredstate/runtime/OrchestrationConfig.java \
        runtime/src/test/java/io/casehub/desiredstate/runtime/DomainOrchestratorTest.java
git commit -m "feat: add DomainOrchestrator — graph merging, export validation, coarse+fine-grained edges Refs #140"
```

---

## Batch 3: Ops Domain Registrations [ops#23]

### Task 4: Create InfraDomainRegistration + DeploymentDomainRegistration

**Repo:** casehub-ops
**Files:**
- Create: `infra/src/main/java/io/casehub/ops/infra/InfraDomainRegistration.java`
- Create: `deployment/src/main/java/io/casehub/ops/deployment/DeploymentDomainRegistration.java`
- Create: `infra/src/test/java/io/casehub/ops/infra/InfraDomainRegistrationTest.java`
- Create: `deployment/src/test/java/io/casehub/ops/deployment/DeploymentDomainRegistrationTest.java`

**Interfaces:**
- Consumes: `DomainRegistration`, `InfraGoalCompiler`, `InfraGoals`, `DeploymentGoalCompiler`, `DeploymentGoals`, `CompilationResult`, `DesiredStateGraphFactory`, `NodeId`
- Produces: `InfraDomainRegistration` (domainId="infra", exports: namespace-ready, cluster-ready); `DeploymentDomainRegistration` (domainId="deployment", exports: topology-ready)

- [ ] **Step 1: Write the failing test for InfraDomainRegistration**

```java
package io.casehub.ops.infra;

import io.casehub.desiredstate.api.*;
import org.junit.jupiter.api.Test;
import java.util.Map;
import static org.junit.jupiter.api.Assertions.*;

class InfraDomainRegistrationTest {

    @Test
    void domainId_is_infra() {
        InfraDomainRegistration reg = new InfraDomainRegistration(new InfraGoalCompiler());
        assertEquals("infra", reg.domainId());
    }

    @Test
    void exportedNodes_contains_expected_exports() {
        InfraDomainRegistration reg = new InfraDomainRegistration(new InfraGoalCompiler());
        Map<String, NodeId> exports = reg.exportedNodes();
        assertFalse(exports.isEmpty());
        assertTrue(exports.containsKey("namespace-ready"));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode -o test -pl infra -Dtest=InfraDomainRegistrationTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — class does not exist

- [ ] **Step 3: Implement `InfraDomainRegistration`**

```java
package io.casehub.ops.infra;

import io.casehub.desiredstate.api.*;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.Map;

@ApplicationScoped
public class InfraDomainRegistration implements DomainRegistration {

    private final InfraGoalCompiler compiler;

    @Inject
    public InfraDomainRegistration(InfraGoalCompiler compiler) {
        this.compiler = compiler;
    }

    @Override
    public String domainId() { return "infra"; }

    @Override
    public CompilationResult compile(String goalsPath, DesiredStateGraphFactory factory) {
        InfraGoals goals = InfraGoals.load(goalsPath);
        return compiler.compile(goals, factory);
    }

    @Override
    public Map<String, NodeId> exportedNodes() {
        return Map.of("namespace-ready", NodeId.of("k8s-namespace-prod"));
    }
}
```

- [ ] **Step 4: Write the failing test for DeploymentDomainRegistration**

```java
package io.casehub.ops.deployment;

import io.casehub.desiredstate.api.*;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class DeploymentDomainRegistrationTest {

    @Test
    void domainId_is_deployment() {
        // Constructor params will depend on DeploymentGoalCompiler's actual constructor
        // Adapt based on the actual dependencies
        DeploymentDomainRegistration reg = createRegistration();
        assertEquals("deployment", reg.domainId());
    }

    @Test
    void exportedNodes_contains_topology_ready() {
        DeploymentDomainRegistration reg = createRegistration();
        assertTrue(reg.exportedNodes().containsKey("topology-ready"));
    }

    private DeploymentDomainRegistration createRegistration() {
        // Use actual constructor — adapt to DeploymentGoalCompiler's dependencies
        return new DeploymentDomainRegistration(null); // placeholder — resolve actual deps
    }
}
```

- [ ] **Step 5: Implement `DeploymentDomainRegistration`**

```java
package io.casehub.ops.deployment;

import io.casehub.desiredstate.api.*;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.Map;

@ApplicationScoped
public class DeploymentDomainRegistration implements DomainRegistration {

    private final DeploymentGoalCompiler compiler;

    @Inject
    public DeploymentDomainRegistration(DeploymentGoalCompiler compiler) {
        this.compiler = compiler;
    }

    @Override
    public String domainId() { return "deployment"; }

    @Override
    public CompilationResult compile(String goalsPath, DesiredStateGraphFactory factory) {
        DeploymentGoals goals = DeploymentGoals.load(goalsPath);
        return compiler.compile(goals, factory);
    }

    @Override
    public Map<String, NodeId> exportedNodes() {
        return Map.of("topology-ready", NodeId.of("deployment-topology"));
    }
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `mvn --batch-mode -o test -pl infra,deployment -Dtest="InfraDomainRegistrationTest,DeploymentDomainRegistrationTest"`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add infra/src/main/java/io/casehub/ops/infra/InfraDomainRegistration.java \
        deployment/src/main/java/io/casehub/ops/deployment/DeploymentDomainRegistration.java \
        infra/src/test/java/io/casehub/ops/infra/InfraDomainRegistrationTest.java \
        deployment/src/test/java/io/casehub/ops/deployment/DeploymentDomainRegistrationTest.java
git commit -m "feat: add InfraDomainRegistration + DeploymentDomainRegistration Refs #23"
```

---

### Task 5: Create ComplianceDomainRegistration + IoTDomainRegistration

**Repo:** casehub-ops
**Files:**
- Create: `compliance/src/main/java/io/casehub/ops/compliance/ComplianceDomainRegistration.java`
- Create: `iot/src/main/java/io/casehub/ops/iot/IoTDomainRegistration.java`
- Create: `compliance/src/test/java/io/casehub/ops/compliance/ComplianceDomainRegistrationTest.java`
- Create: `iot/src/test/java/io/casehub/ops/iot/IoTDomainRegistrationTest.java`

**Interfaces:**
- Consumes: `DomainRegistration`, `ComplianceGoalCompiler`, `IoTGoalCompiler`, `CompilationResult`, `DesiredStateGraphFactory`, `NodeId`
- Produces: `ComplianceDomainRegistration` (domainId="compliance", no exports); `IoTDomainRegistration` (domainId="iot", no exports)

- [ ] **Step 1: Write failing tests**

```java
// ComplianceDomainRegistrationTest.java
package io.casehub.ops.compliance;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class ComplianceDomainRegistrationTest {
    @Test
    void domainId_is_compliance() {
        ComplianceDomainRegistration reg = new ComplianceDomainRegistration(null);
        assertEquals("compliance", reg.domainId());
    }

    @Test
    void exportedNodes_is_empty_for_leaf_domain() {
        ComplianceDomainRegistration reg = new ComplianceDomainRegistration(null);
        assertTrue(reg.exportedNodes().isEmpty());
    }
}
```

```java
// IoTDomainRegistrationTest.java
package io.casehub.ops.iot;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class IoTDomainRegistrationTest {
    @Test
    void domainId_is_iot() {
        IoTDomainRegistration reg = new IoTDomainRegistration(null);
        assertEquals("iot", reg.domainId());
    }

    @Test
    void exportedNodes_is_empty_for_leaf_domain() {
        IoTDomainRegistration reg = new IoTDomainRegistration(null);
        assertTrue(reg.exportedNodes().isEmpty());
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn --batch-mode -o test -pl compliance,iot -Dtest="ComplianceDomainRegistrationTest,IoTDomainRegistrationTest" -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL

- [ ] **Step 3: Implement `ComplianceDomainRegistration`**

```java
package io.casehub.ops.compliance;

import io.casehub.desiredstate.api.*;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.Map;

@ApplicationScoped
public class ComplianceDomainRegistration implements DomainRegistration {

    private final ComplianceGoalCompiler compiler;

    @Inject
    public ComplianceDomainRegistration(ComplianceGoalCompiler compiler) {
        this.compiler = compiler;
    }

    @Override
    public String domainId() { return "compliance"; }

    @Override
    public CompilationResult compile(String goalsPath, DesiredStateGraphFactory factory) {
        ComplianceGoals goals = ComplianceGoals.load(goalsPath);
        return compiler.compile(goals, factory);
    }

    @Override
    public Map<String, NodeId> exportedNodes() { return Map.of(); }
}
```

- [ ] **Step 4: Implement `IoTDomainRegistration`**

```java
package io.casehub.ops.iot;

import io.casehub.desiredstate.api.*;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.Map;

@ApplicationScoped
public class IoTDomainRegistration implements DomainRegistration {

    private final IoTGoalCompiler compiler;

    @Inject
    public IoTDomainRegistration(IoTGoalCompiler compiler) {
        this.compiler = compiler;
    }

    @Override
    public String domainId() { return "iot"; }

    @Override
    public CompilationResult compile(String goalsPath, DesiredStateGraphFactory factory) {
        IoTGoals goals = IoTGoals.load(goalsPath);
        return compiler.compile(goals, factory);
    }

    @Override
    public Map<String, NodeId> exportedNodes() { return Map.of(); }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn --batch-mode -o test -pl compliance,iot -Dtest="ComplianceDomainRegistrationTest,IoTDomainRegistrationTest"`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add compliance/src/main/java/io/casehub/ops/compliance/ComplianceDomainRegistration.java \
        iot/src/main/java/io/casehub/ops/iot/IoTDomainRegistration.java \
        compliance/src/test/java/io/casehub/ops/compliance/ComplianceDomainRegistrationTest.java \
        iot/src/test/java/io/casehub/ops/iot/IoTDomainRegistrationTest.java
git commit -m "feat: add ComplianceDomainRegistration + IoTDomainRegistration Refs #23"
```

---

## Batch 4: Integration Wiring + ARC42STORIES [ops#23]

### Task 6: Orchestration YAML + integration tests + ARC42STORIES update

**Repo:** casehub-ops
**Files:**
- Create: `app/src/main/resources/casehub-orchestration.yaml`
- Create: `app/src/test/java/io/casehub/ops/app/CrossDomainOrchestrationTest.java`
- Modify: `ARC42STORIES.MD` — §2 Constraints, §10 Architectural Decisions

**Interfaces:**
- Consumes: `DomainOrchestrator`, `OrchestrationConfig`, all four `DomainRegistration` implementations, `DesiredStateGraphFactory`, `ReconciliationLoop`
- Produces: Integration test verifying four-domain merged graph with correct ordering

- [ ] **Step 1: Create orchestration YAML**

```yaml
# app/src/main/resources/casehub-orchestration.yaml
domains:
  infra:
    goals: casehub-infra.yaml

  deployment:
    goals: casehub-deployment.yaml
    depends-on: [infra]

  compliance:
    goals: casehub-compliance.yaml
    depends-on: [deployment]
    edges:
      - from: soc2-encryption
        to: infra:namespace-ready

  iot:
    goals: casehub-iot.yaml
    edges:
      - from: gateway-config
        to: infra:namespace-ready
```

- [ ] **Step 2: Write the integration test**

```java
package io.casehub.ops.app;

import io.casehub.desiredstate.api.*;
import io.casehub.desiredstate.runtime.*;
import org.junit.jupiter.api.Test;
import java.util.*;
import static org.junit.jupiter.api.Assertions.*;

class CrossDomainOrchestrationTest {

    private final DesiredStateGraphFactory factory = new DefaultDesiredStateGraphFactory();

    @Test
    void four_domain_merge_respects_ordering() {
        // Stub registrations with minimal nodes matching orchestration YAML
        DomainRegistration infra = stubReg("infra",
            List.of(node("k8s-namespace-prod", "infra")),
            Map.of("namespace-ready", NodeId.of("k8s-namespace-prod")));

        DomainRegistration deploy = stubReg("deployment",
            List.of(node("agent-alpha", "deploy")),
            Map.of("topology-ready", NodeId.of("agent-alpha")));

        DomainRegistration compliance = stubReg("compliance",
            List.of(node("soc2-encryption", "compliance")),
            Map.of());

        DomainRegistration iot = stubReg("iot",
            List.of(node("gateway-config", "iot")),
            Map.of());

        var config = new OrchestrationConfig(List.of(
            new OrchestrationConfig.DomainEntry("infra", "casehub-infra.yaml", List.of(), List.of()),
            new OrchestrationConfig.DomainEntry("deployment", "casehub-deployment.yaml",
                List.of("infra"), List.of()),
            new OrchestrationConfig.DomainEntry("compliance", "casehub-compliance.yaml",
                List.of("deployment"),
                List.of(new OrchestrationConfig.EdgeEntry("soc2-encryption", "infra", "namespace-ready"))),
            new OrchestrationConfig.DomainEntry("iot", "casehub-iot.yaml",
                List.of(),
                List.of(new OrchestrationConfig.EdgeEntry("gateway-config", "infra", "namespace-ready")))
        ));

        var orchestrator = new DomainOrchestrator(Map.of(
            "infra", infra, "deployment", deploy, "compliance", compliance, "iot", iot));
        DesiredStateGraph merged = orchestrator.orchestrate(config, factory);

        // All 4 nodes present
        assertEquals(4, merged.nodes().size());

        // Coarse: deployment root depends on infra leaf
        assertTrue(merged.dependenciesOf(NodeId.of("agent-alpha")).contains(NodeId.of("k8s-namespace-prod")));

        // Coarse: compliance root depends on deployment leaf
        assertTrue(merged.dependenciesOf(NodeId.of("soc2-encryption")).contains(NodeId.of("agent-alpha")));

        // Fine-grained: soc2-encryption depends on infra namespace
        assertTrue(merged.dependenciesOf(NodeId.of("soc2-encryption")).contains(NodeId.of("k8s-namespace-prod")));

        // Fine-grained: gateway-config depends on infra namespace
        assertTrue(merged.dependenciesOf(NodeId.of("gateway-config")).contains(NodeId.of("k8s-namespace-prod")));

        // IoT is NOT sequenced after infra (no depends-on, only fine-grained edge)
        // gateway-config only depends on k8s-namespace-prod, not on agent-alpha
        assertFalse(merged.dependenciesOf(NodeId.of("gateway-config")).contains(NodeId.of("agent-alpha")));
    }

    private DesiredNode node(String id, String type) {
        return new DesiredNode(NodeId.of(id), new NodeSpec() {
            @Override public NodeType type() { return NodeType.of(type); }
        }, HumanGating.NONE);
    }

    private DomainRegistration stubReg(String domainId, List<DesiredNode> nodes,
                                        Map<String, NodeId> exports) {
        return new DomainRegistration() {
            @Override public String domainId() { return domainId; }
            @Override public CompilationResult compile(String p, DesiredStateGraphFactory f) {
                return CompilationResult.single(f.of(nodes, List.of()));
            }
            @Override public Map<String, NodeId> exportedNodes() { return exports; }
        };
    }
}
```

- [ ] **Step 3: Run test to verify it passes**

Run: `mvn --batch-mode -o test -pl app -Dtest=CrossDomainOrchestrationTest`
Expected: PASS

- [ ] **Step 4: Update ARC42STORIES §2 — replace single-domain constraint**

Replace the constraint in §2 Architectural Constraints:

Old:
> Single-domain deployment: one domain module per classpath. Multiple domain modules create CDI ambiguity...

New:
> Multi-domain deployment: multiple domain modules may coexist on the same classpath. The desiredstate runtime routes `ActualStateAdapter` and `NodeProvisioner` calls by `NodeType`; `FaultPolicy` and `EventSource` implementations are collected and merged. `GoalCompiler` ambiguity is resolved via `@DesiredStateQualifier` and `DomainRegistration`. Domain modules must ensure their `handledTypes()` sets are disjoint — the routers throw on duplicate `NodeType` claims.

- [ ] **Step 5: Update ARC42STORIES §10 — document cross-domain decision**

Add to §10 Architectural Decisions:

> **Cross-domain orchestration via flat graph merge.** Multiple domain modules' graphs are merged into a single `DesiredStateGraph` via `sequenceAfter()` (coarse ordering) and `withDependency()` (fine-grained edges). Domain ordering is declared in a `casehub-orchestration.yaml` file. The `DomainOrchestrator` in `casehub-desiredstate-runtime` handles discovery, compilation, merging, and export validation. Hierarchical multi-process deployment is a documented future extension. See casehubio/casehub-desiredstate#140 and casehubio/casehub-ops#23.

- [ ] **Step 6: Run full build**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS

- [ ] **Step 7: Commit**

```bash
git add app/src/main/resources/casehub-orchestration.yaml \
        app/src/test/java/io/casehub/ops/app/CrossDomainOrchestrationTest.java \
        ARC42STORIES.MD
git commit -m "feat: add cross-domain orchestration YAML, integration tests, update ARC42STORIES Refs #23"
```

---

## References

- [2026-09-12-cross-domain-dependency-graphs-design.md] — design spec this plan implements
- [ImmutableDesiredStateGraph.java:283-298] — existing `connect()` pattern for `sequenceAfter()`
- [DefaultActualStateAdapterRouter.java] — NodeType-based routing (multi-domain ready)
- [DefaultNodeProvisionerRouter.java] — NodeType-based routing (multi-domain ready)
- [ReconciliationLoop.java] — per-tenant reconciliation with type-filtered resync
- [CompilationResult.java] — SingleGraph/Lifecycle sealed variants
- [@DesiredStateQualifier] — GoalCompiler CDI qualification (#110)
- [casehubio/casehub-desiredstate#140] — generic framework issue
- [casehubio/casehub-ops#23] — focal issue
