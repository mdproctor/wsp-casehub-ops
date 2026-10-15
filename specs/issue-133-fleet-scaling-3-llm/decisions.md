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
