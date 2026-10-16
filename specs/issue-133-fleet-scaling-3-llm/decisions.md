## D1: How to express 3 LLM instances in topology YAML

**Choice:** forEach + iterations pattern
**Alternatives:**
- Explicit 3 nodes — doesn't demonstrate fleet scaling as a concept, just copy-paste
- replicas: 3 on single node — won't work; claudony ignores replicas field, registers apps individually
**Rationale:** Teaches the forEach expansion pattern ("declare once, deploy many"), is the established pattern in this codebase (ha-multi-az-trading.yaml), and scales cleanly to N instances
**Trade-offs:** Slightly more complex YAML than 3 explicit nodes, but that complexity IS the lesson
**Sources:** topology-tests/src/test/resources/topologies/microservices/ha-multi-az-trading.yaml, claudony/src/main/java/io/casehub/ops/claudony/ClaudonyDeploymentTarget.java:103-114
**Exploration:** quick
**Status:** captured

## D3: Per-instance parameterization mechanism

**Choice:** CSV data source with inline data block
**Alternatives:**
- Simple list + variables lookup — requires nested variable resolution (${var.fleet.${each.name}.capability}) which may not be supported
- Simple list, number only — doesn't demonstrate differentiation, misses the pedagogical point
**Rationale:** CSV data source is already proven in ForEachExpander (csvDataSource_expandsWithRowFields test). Naturally expresses a "fleet manifest" — the data block describes fleet composition (name, capability), the topology references it. Each instance gets unique config that flows through to eidos for identity generation.
**Trade-offs:** Slightly more YAML surface area than a simple list, but the data block IS the lesson — fleet composition as data, not duplicated structure
**Depends on:** D1 (forEach pattern)
**Sources:** desiredstate/yaml/runtime ForEachExpanderTest.java:252-271, CsvDataSource
**Exploration:** quick
**Status:** captured

## D4: Eidos identity integration

**Choice:** Both levels — static descriptors work (Level A), topology parameters demonstrate flow (Level B)
**Alternatives:**
- Static descriptors only — works but doesn't show the parameter flow concept
- Dynamic-only — requires eidos changes that are out of scope for #133
**Rationale:** The topology YAML includes per-instance config (capability, role/unit identifier) that flows through to claudony registration. Static descriptors.yaml provides 3 entries as the working baseline. The config values are there for eidos to consume when it gets parameterized (separate work). Even passing UUIDs as unit identifiers demonstrates the concept.
**Trade-offs:** Two sources of truth temporarily (descriptors.yaml + topology config), but that's the migration path — Level B eventually makes descriptors.yaml unnecessary
**Depends on:** D3 (CSV data source for per-instance params)
**Sources:** topology-tests/src/test/resources/META-INF/eidos/descriptors.yaml
**Exploration:** quick
**Status:** captured

## D2: Integration test strategy

**Choice:** Single test class following HelloLlmIntegrationTest pattern
**Alternatives:**
- Separate tests per concern (registration, health, request handling) — unnecessary isolation for S/Low scope
**Rationale:** Follows established pattern from #139. Compile 04-llm-fleet.yaml → deploy all nodes via real claudony → assert all 3 LLM instances register and report healthy. Proportional to feature scope.
**Trade-offs:** Less granular failure isolation, but the assert-per-instance pattern gives clear enough signal
**Sources:** topology-tests/src/test/java/io/casehub/ops/topology/integration/HelloLlmIntegrationTest.java
**Exploration:** quick
**Status:** captured
