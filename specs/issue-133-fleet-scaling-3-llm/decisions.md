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

## D2: Integration test strategy

**Choice:** Single test class following HelloLlmIntegrationTest pattern
**Alternatives:**
- Separate tests per concern (registration, health, request handling) — unnecessary isolation for S/Low scope
**Rationale:** Follows established pattern from #139. Compile 04-llm-fleet.yaml → deploy all nodes via real claudony → assert all 3 LLM instances register and report healthy. Proportional to feature scope.
**Trade-offs:** Less granular failure isolation, but the assert-per-instance pattern gives clear enough signal
**Sources:** topology-tests/src/test/java/io/casehub/ops/topology/integration/HelloLlmIntegrationTest.java
**Exploration:** quick
**Status:** captured
