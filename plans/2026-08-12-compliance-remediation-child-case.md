# Compliance Remediation Child Case Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #37 — compliance-remediation child case — full implementation
**Issue group:** #29 (epic), #30, #31, #34, #37, #38, #47

**Goal:** Replace the `StubChildCaseDescriptor` for `ops:compliance-remediation`
with a full `ComplianceRemediationCaseDescriptor` implementing assess → remediate
→ verify → escalate, with limited auto-fix for infrastructure-configurable
controls.

**Architecture:** Four-phase worker chain matching `IncidentResponseCaseDescriptor`
(#34). The case receives violation data from existing bindings (ApplicationCaseDescriptor
and ServiceCaseDescriptor), classifies the violation, applies config updates via
`ApplicationLifecycleService.updateServiceConfig()` for auto-fixable controls
(LOG_RETENTION, ENCRYPTION_AT_REST), and escalates everything else. Verification
uses `NodeConvergenceTracker`.

**Tech Stack:** Java 21, Quarkus, CaseHub engine worker API, AssertJ

## Global Constraints

- No dependency on the `compliance/` module from `app/` (single-domain CDI constraint)
- Worker methods are `static` package-private for direct unit testing (same as IncidentResponseCaseDescriptor)
- All context keys use `compliance` prefix (`.complianceAssessment`, `.complianceRemediationExecuted`, `.complianceEscalationRequired`, `.complianceStatus`)
- `WorkerResult.failed()` only for malformed input; lifecycle exceptions route to escalation

**Execution order:** Task 1 → Task 3 → Task 2. Task 2's remediate tests create
anonymous subclass overrides of `updateServiceConfig()`, which must exist before
those tests compile. Task 3 adds the method.

---

### Task 1: ComplianceRemediationCaseDescriptor — assess worker + case definition

**Files:**
- Create: `app/src/main/java/io/casehub/ops/app/case_/ComplianceRemediationCaseDescriptor.java`
- Test: `app/src/test/java/io/casehub/ops/app/case_/ComplianceRemediationCaseDescriptorTest.java`

**Interfaces:**
- Consumes: `CaseDefinition.builder()`, `Capability.of()`, `Worker.builder()`, `Binding.builder()`, `WorkerFunction.Sync`, `WorkerResult`, `ContextChangeTrigger` (all from casehub-engine API)
- Produces: `ComplianceRemediationCaseDescriptor.build(ApplicationLifecycleService, NodeConvergenceTracker) → CaseDefinition`, `assessCompliance(Map<String,Object>) → WorkerResult` (package-private static)

- [ ] **Step 1: Write failing tests for case definition identity and assess worker**

```java
package io.casehub.ops.app.case_;

import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.ContextChangeTrigger;
import io.casehub.worker.api.WorkerOutcome;
import io.casehub.worker.api.WorkerResult;
import org.junit.jupiter.api.Test;

import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

class ComplianceRemediationCaseDescriptorTest {

    // --- Case definition structure ---

    @Test
    void buildReturnsCorrectIdentity() {
        CaseDefinition def = ComplianceRemediationCaseDescriptor.build(null, null);
        assertThat(def.getNamespace()).isEqualTo("ops");
        assertThat(def.getName()).isEqualTo("compliance-remediation");
        assertThat(def.getVersion()).isEqualTo("1.0");
    }

    @Test
    void hasFourCapabilities() {
        CaseDefinition def = ComplianceRemediationCaseDescriptor.build(null, null);
        assertThat(def.getCapabilities()).hasSize(4);
        assertThat(def.getCapabilities()).extracting("name")
                .containsExactlyInAnyOrder("assess-compliance", "remediate-compliance",
                        "verify-compliance", "escalate-compliance");
    }

    @Test
    void hasFourWorkers() {
        CaseDefinition def = ComplianceRemediationCaseDescriptor.build(null, null);
        assertThat(def.getWorkers()).hasSize(4);
    }

    @Test
    void hasThreeInternalBindings() {
        CaseDefinition def = ComplianceRemediationCaseDescriptor.build(null, null);
        assertThat(def.getBindings()).hasSize(3);
        assertThat(def.getBindings()).extracting("name")
                .containsExactlyInAnyOrder("on-compliance-assessment",
                        "on-compliance-remediation-executed", "on-compliance-escalation-required");
    }

    @Test
    void hasCompletionPredicate() {
        CaseDefinition def = ComplianceRemediationCaseDescriptor.build(null, null);
        assertThat(def.getCompletion()).isNotNull();
    }

    @Test
    void assessmentBindingTriggersOnComplianceAssessment() {
        CaseDefinition def = ComplianceRemediationCaseDescriptor.build(null, null);
        var binding = def.getBindings().stream()
                .filter(b -> b.getName().equals("on-compliance-assessment"))
                .findFirst().orElseThrow();
        assertThat(binding.getOn()).isInstanceOf(ContextChangeTrigger.class);
    }

    // --- Assess worker: auto-fixable controls ---

    @Test
    void assessFailLogRetentionWithServiceIdReturnsUpdateConfig() {
        var input = violationInput("log-retention-policy", "LOG_RETENTION", "FAIL", "order-api");
        WorkerResult<?> result = ComplianceRemediationCaseDescriptor.assessCompliance(input);
        var assessment = extractAssessment(result);
        assertThat(assessment.get("action")).isEqualTo("update-config");
        assertThat(assessment.get("controlType")).isEqualTo("LOG_RETENTION");
        @SuppressWarnings("unchecked")
        Map<String, String> configUpdates = (Map<String, String>) assessment.get("configUpdates");
        assertThat(configUpdates).containsEntry("LOG_RETENTION_DAYS", "365");
        assertThat(configUpdates).containsEntry("LOG_RETENTION_ENABLED", "true");
    }

    @Test
    void assessFailEncryptionWithServiceIdReturnsUpdateConfig() {
        var input = violationInput("encryption-at-rest", "ENCRYPTION_AT_REST", "FAIL", "order-api");
        WorkerResult<?> result = ComplianceRemediationCaseDescriptor.assessCompliance(input);
        var assessment = extractAssessment(result);
        assertThat(assessment.get("action")).isEqualTo("update-config");
        @SuppressWarnings("unchecked")
        Map<String, String> configUpdates = (Map<String, String>) assessment.get("configUpdates");
        assertThat(configUpdates).containsEntry("ENCRYPTION_ENABLED", "true");
        assertThat(configUpdates).containsEntry("ENCRYPTION_CIPHER", "AES-256");
    }

    // --- Assess worker: escalation paths ---

    @Test
    @SuppressWarnings("unchecked")
    void assessFailAutoFixableNoServiceIdEscalates() {
        var input = violationInput("log-retention-policy", "LOG_RETENTION", "FAIL", null);
        WorkerResult<?> result = ComplianceRemediationCaseDescriptor.assessCompliance(input);
        Map<String, Object> output = (Map<String, Object>) result.output();
        assertThat(output).containsKey("complianceEscalationRequired");
    }

    @Test
    @SuppressWarnings("unchecked")
    void assessFailNonAutoFixableEscalates() {
        var input = violationInput("access-review-quarterly", "ACCESS_REVIEW", "FAIL", "order-api");
        WorkerResult<?> result = ComplianceRemediationCaseDescriptor.assessCompliance(input);
        Map<String, Object> output = (Map<String, Object>) result.output();
        assertThat(output).containsKey("complianceEscalationRequired");
    }

    @Test
    @SuppressWarnings("unchecked")
    void assessUnavailableOutcomeEscalates() {
        var input = violationInput("encryption-at-rest", "ENCRYPTION_AT_REST", "UNAVAILABLE", "order-api");
        WorkerResult<?> result = ComplianceRemediationCaseDescriptor.assessCompliance(input);
        Map<String, Object> output = (Map<String, Object>) result.output();
        assertThat(output).containsKey("complianceEscalationRequired");
    }

    @Test
    @SuppressWarnings("unchecked")
    void assessStaleOutcomeEscalates() {
        var input = violationInput("encryption-at-rest", "ENCRYPTION_AT_REST", "STALE", "order-api");
        WorkerResult<?> result = ComplianceRemediationCaseDescriptor.assessCompliance(input);
        Map<String, Object> output = (Map<String, Object>) result.output();
        assertThat(output).containsKey("complianceEscalationRequired");
    }

    // --- Assess worker: validation failures ---

    @Test
    void assessNullInputReturnsFailed() {
        WorkerResult<?> result = ComplianceRemediationCaseDescriptor.assessCompliance(null);
        assertThat(result.outcome()).isInstanceOf(WorkerOutcome.Failed.class);
    }

    @Test
    void assessMissingControlIdReturnsFailed() {
        var input = new LinkedHashMap<String, Object>();
        input.put("controlType", "LOG_RETENTION");
        input.put("outcome", "FAIL");
        input.put("tenancyId", "tenant-1");
        WorkerResult<?> result = ComplianceRemediationCaseDescriptor.assessCompliance(input);
        assertThat(result.outcome()).isInstanceOf(WorkerOutcome.Failed.class);
    }

    @Test
    void assessMissingOutcomeReturnsFailed() {
        var input = new LinkedHashMap<String, Object>();
        input.put("controlId", "encryption-at-rest");
        input.put("controlType", "ENCRYPTION_AT_REST");
        input.put("tenancyId", "tenant-1");
        WorkerResult<?> result = ComplianceRemediationCaseDescriptor.assessCompliance(input);
        assertThat(result.outcome()).isInstanceOf(WorkerOutcome.Failed.class);
    }

    // --- Test helpers ---

    @SuppressWarnings("unchecked")
    private Map<String, Object> extractAssessment(WorkerResult<?> result) {
        return (Map<String, Object>) ((Map<String, Object>) result.output()).get("complianceAssessment");
    }

    private Map<String, Object> violationInput(String controlId, String controlType,
                                                String outcome, String serviceId) {
        var input = new LinkedHashMap<String, Object>();
        input.put("controlId", controlId);
        input.put("controlType", controlType);
        input.put("outcome", outcome);
        input.put("tenancyId", "tenant-1");
        input.put("applicationId", UUID.randomUUID().toString());
        input.put("frameworks", List.of("SOC2:CC6.1", "GDPR:Art.32"));
        input.put("detail", "Test violation detail");
        if (serviceId != null) input.put("serviceId", serviceId);
        return input;
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn --batch-mode -o test -pl app -Dtest=ComplianceRemediationCaseDescriptorTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation failure — `ComplianceRemediationCaseDescriptor` does not exist

- [ ] **Step 3: Implement ComplianceRemediationCaseDescriptor with assess worker**

```java
package io.casehub.ops.app.case_;

import io.casehub.api.model.Binding;
import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.ContextChangeTrigger;
import io.casehub.ops.app.service.ApplicationLifecycleService;
import io.casehub.ops.app.service.NodeConvergenceTracker;
import io.casehub.worker.api.Capability;
import io.casehub.worker.api.Worker;
import io.casehub.worker.api.WorkerFunction;
import io.casehub.worker.api.WorkerResult;
import io.casehub.worker.api.WorkerScope;

import java.util.HashSet;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.UUID;

public final class ComplianceRemediationCaseDescriptor {

    private static final Map<String, Map<String, String>> AUTO_FIX_CONFIGS = Map.of(
            "LOG_RETENTION", Map.of("LOG_RETENTION_DAYS", "365", "LOG_RETENTION_ENABLED", "true"),
            "ENCRYPTION_AT_REST", Map.of("ENCRYPTION_ENABLED", "true", "ENCRYPTION_CIPHER", "AES-256"));

    private ComplianceRemediationCaseDescriptor() {}

    public static CaseDefinition build(ApplicationLifecycleService lifecycleService,
                                        NodeConvergenceTracker convergenceTracker) {
        return CaseDefinition.builder()
                .namespace("ops")
                .name("compliance-remediation")
                .version("1.0")
                .title("Compliance Remediation")
                .summary("Assesses compliance violations and applies config fixes or escalates")
                .capabilities(capabilities())
                .workers(workers(lifecycleService, convergenceTracker))
                .bindings(bindings())
                .completion(".complianceStatus == \"resolved\" || .complianceStatus == \"escalated\"")
                .build();
    }

    private static List<Capability> capabilities() {
        return List.of(
                Capability.of("assess-compliance", "any", "any"),
                Capability.of("remediate-compliance", "any", "any"),
                Capability.of("verify-compliance", "any", "any"),
                Capability.of("escalate-compliance", "any", "any"));
    }

    @SuppressWarnings("unchecked")
    private static List<Worker> workers(ApplicationLifecycleService lifecycleService,
                                         NodeConvergenceTracker convergenceTracker) {
        return List.of(
                Worker.builder()
                        .name("compliance-assess-worker")
                        .capabilityName("assess-compliance")
                        .function(new WorkerFunction.Sync<>(Map.class, Map.class,
                                (input, scope) -> assessCompliance(input)))
                        .build(),
                Worker.builder()
                        .name("compliance-remediate-worker")
                        .capabilityName("remediate-compliance")
                        .function(new WorkerFunction.Sync<>(Map.class, Map.class,
                                (input, scope) -> remediateCompliance(input, lifecycleService)))
                        .build(),
                Worker.builder()
                        .name("compliance-verify-worker")
                        .capabilityName("verify-compliance")
                        .function(new WorkerFunction.Sync<>(Map.class, Map.class,
                                (input, scope) -> verifyCompliance(input, scope, convergenceTracker)))
                        .build(),
                Worker.builder()
                        .name("compliance-escalate-worker")
                        .capabilityName("escalate-compliance")
                        .function(new WorkerFunction.Sync<>(Map.class, Map.class,
                                (input, scope) -> escalateCompliance(input)))
                        .build());
    }

    private static List<Binding> bindings() {
        return List.of(
                Binding.builder()
                        .name("on-compliance-assessment")
                        .on(new ContextChangeTrigger(".complianceAssessment"))
                        .capability(Capability.of("remediate-compliance", "any", "any"))
                        .build(),
                Binding.builder()
                        .name("on-compliance-remediation-executed")
                        .on(new ContextChangeTrigger(".complianceRemediationExecuted"))
                        .capability(Capability.of("verify-compliance", "any", "any"))
                        .build(),
                Binding.builder()
                        .name("on-compliance-escalation-required")
                        .on(new ContextChangeTrigger(".complianceEscalationRequired"))
                        .capability(Capability.of("escalate-compliance", "any", "any"))
                        .build());
    }

    static WorkerResult assessCompliance(Map<String, Object> input) {
        if (input == null) return WorkerResult.failed("Violation data is null");

        String controlId = (String) input.get("controlId");
        String controlType = (String) input.get("controlType");
        String outcome = (String) input.get("outcome");
        String tenancyId = (String) input.get("tenancyId");

        if (controlId == null || controlId.isBlank())
            return WorkerResult.failed("controlId is required");
        if (controlType == null || controlType.isBlank())
            return WorkerResult.failed("controlType is required");
        if (outcome == null || outcome.isBlank())
            return WorkerResult.failed("outcome is required");
        if (tenancyId == null || tenancyId.isBlank())
            return WorkerResult.failed("tenancyId is required");

        String serviceId = (String) input.get("serviceId");
        String applicationId = (String) input.get("applicationId");
        String detail = (String) input.get("detail");
        @SuppressWarnings("unchecked")
        List<String> frameworks = (List<String>) input.get("frameworks");

        boolean isAutoFixable = AUTO_FIX_CONFIGS.containsKey(controlType);
        boolean hasServiceId = serviceId != null && !serviceId.isBlank();
        boolean isFail = "FAIL".equals(outcome);

        String action = (isFail && isAutoFixable && hasServiceId) ? "update-config" : "escalate";

        var assessment = new LinkedHashMap<String, Object>();
        assessment.put("action", action);
        assessment.put("controlId", controlId);
        assessment.put("controlType", controlType);
        assessment.put("outcome", outcome);
        assessment.put("tenancyId", tenancyId);
        if (serviceId != null) assessment.put("serviceId", serviceId);
        if (applicationId != null) assessment.put("applicationId", applicationId);
        if (detail != null) assessment.put("detail", detail);
        if (frameworks != null) assessment.put("frameworks", frameworks);

        if ("update-config".equals(action)) {
            assessment.put("configUpdates", AUTO_FIX_CONFIGS.get(controlType));
            assessment.put("reason", controlType + " violation — applying config fix");
            return WorkerResult.of(Map.of("complianceAssessment", assessment));
        }

        assessment.put("reason", describeEscalation(controlType, outcome, hasServiceId));
        return WorkerResult.of(Map.of(
                "complianceEscalationRequired", true,
                "complianceAssessment", assessment));
    }

    @SuppressWarnings("unchecked")
    static WorkerResult remediateCompliance(Map<String, Object> input,
                                             ApplicationLifecycleService lifecycleService) {
        Map<String, Object> assessment = (Map<String, Object>) input.get("complianceAssessment");
        String applicationId = (String) assessment.get("applicationId");
        String serviceId = (String) assessment.get("serviceId");
        String tenancyId = (String) assessment.get("tenancyId");
        String controlId = (String) assessment.get("controlId");
        Map<String, String> configUpdates = (Map<String, String>) assessment.get("configUpdates");

        Set<String> affectedNodeIds;
        try {
            affectedNodeIds = lifecycleService.updateServiceConfig(
                    UUID.fromString(applicationId), serviceId, configUpdates, tenancyId);
        } catch (Exception e) {
            var escalation = new LinkedHashMap<String, Object>();
            escalation.put("reason", "Remediation failed: " + e.getMessage());
            escalation.put("controlId", controlId);
            escalation.put("serviceId", serviceId);
            return WorkerResult.of(Map.of("complianceEscalationRequired", true,
                                           "complianceRemediationError", escalation));
        }

        var executed = new LinkedHashMap<String, Object>();
        executed.put("action", "update-config");
        executed.put("controlId", controlId);
        executed.put("serviceId", serviceId);
        executed.put("affectedNodeIds", List.copyOf(affectedNodeIds));
        executed.put("configUpdates", configUpdates);

        return WorkerResult.of(Map.of("complianceRemediationExecuted", executed));
    }

    @SuppressWarnings("unchecked")
    static WorkerResult verifyCompliance(Map<String, Object> input,
                                          WorkerScope scope,
                                          NodeConvergenceTracker convergenceTracker) {
        Map<String, Object> executed = (Map<String, Object>) input.get("complianceRemediationExecuted");
        List<String> affectedNodeIdsList = (List<String>) executed.get("affectedNodeIds");
        Set<String> affectedNodeIds = new HashSet<>(affectedNodeIdsList);

        UUID caseId = scope.caseId();
        convergenceTracker.register(caseId, affectedNodeIds, "complianceStatus", "resolved");

        return WorkerResult.of(Map.of());
    }

    @SuppressWarnings("unchecked")
    static WorkerResult escalateCompliance(Map<String, Object> input) {
        Map<String, Object> assessment = (Map<String, Object>) input.getOrDefault(
                "complianceAssessment", Map.of());
        String controlId = (String) assessment.getOrDefault("controlId", "unknown");
        String controlType = (String) assessment.getOrDefault("controlType", "unknown");
        String outcome = (String) assessment.getOrDefault("outcome", "unknown");
        String serviceId = (String) assessment.getOrDefault("serviceId", "unknown");
        String detail = (String) assessment.getOrDefault("detail", "");
        @SuppressWarnings("unchecked")
        List<String> frameworks = (List<String>) assessment.getOrDefault("frameworks", List.of());

        Map<String, Object> remediationError = (Map<String, Object>) input.get("complianceRemediationError");
        String escalationDetail = remediationError != null
                ? (String) remediationError.get("reason")
                : "Compliance violation requires human review: " + controlType + " " + outcome;

        var escalation = new LinkedHashMap<String, Object>();
        escalation.put("summary", "Compliance violation on " + controlId + " requires human review");
        escalation.put("controlId", controlId);
        escalation.put("controlType", controlType);
        escalation.put("outcome", outcome);
        escalation.put("frameworks", frameworks);
        escalation.put("serviceId", serviceId);
        escalation.put("risk", "HIGH");
        escalation.put("detail", escalationDetail);

        return WorkerResult.of(Map.of(
                "complianceEscalation", escalation,
                "complianceStatus", "escalated"));
    }

    private static String describeEscalation(String controlType, String outcome, boolean hasServiceId) {
        if (!"FAIL".equals(outcome)) return outcome + " — escalating (outcome not actionable)";
        if (!hasServiceId) return controlType + " — escalating (no service target)";
        return controlType + " — escalating (no auto-fix available)";
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn --batch-mode -o test -pl app -Dtest=ComplianceRemediationCaseDescriptorTest`
Expected: all tests PASS

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/io/casehub/ops/app/case_/ComplianceRemediationCaseDescriptor.java app/src/test/java/io/casehub/ops/app/case_/ComplianceRemediationCaseDescriptorTest.java
git commit -m "feat(#37): ComplianceRemediationCaseDescriptor — assess worker + case definition

Four-phase worker chain: assess → remediate → verify → escalate.
Assess worker classifies violations by outcome × controlType × serviceId.
Auto-fix for LOG_RETENTION and ENCRYPTION_AT_REST; escalate all others.

Refs casehubio/casehub-ops#37"
```

---

### Task 2: Remediate, verify, and escalate worker tests + registrar wiring

**Files:**
- Modify: `app/src/test/java/io/casehub/ops/app/case_/ComplianceRemediationCaseDescriptorTest.java`
- Modify: `app/src/main/java/io/casehub/ops/app/case_/CaseDefinitionRegistrar.java`
- Modify: `app/src/test/java/io/casehub/ops/app/case_/CaseDefinitionRegistrarTest.java`

**Interfaces:**
- Consumes: `ComplianceRemediationCaseDescriptor.build(ApplicationLifecycleService, NodeConvergenceTracker)`, `ApplicationLifecycleService.updateServiceConfig()` (from Task 3), `NodeConvergenceTracker.register()`, `TestWorkerScope` (from IncidentResponseCaseDescriptorTest — same package)
- Produces: full test coverage for remediate/verify/escalate workers; registrar wiring complete

- [ ] **Step 1: Add remediate, verify, escalate tests to ComplianceRemediationCaseDescriptorTest**

Append the following tests to the test class:

```java
    // --- Remediate worker ---

    @Test
    @SuppressWarnings("unchecked")
    void remediateUpdateConfigCallsUpdateServiceConfig() {
        Set<String> returnedNodeIds = Set.of("cluster-1:order-api:deployment");
        ApplicationLifecycleService mockService = new ApplicationLifecycleService() {
            @Override
            public Set<String> updateServiceConfig(UUID appId, String serviceId,
                                                    Map<String, String> configUpdates, String tenancyId) {
                return returnedNodeIds;
            }
        };

        UUID appId = UUID.randomUUID();
        var assessment = new LinkedHashMap<String, Object>();
        assessment.put("action", "update-config");
        assessment.put("controlId", "encryption-at-rest");
        assessment.put("applicationId", appId.toString());
        assessment.put("serviceId", "order-api");
        assessment.put("tenancyId", "tenant-1");
        assessment.put("configUpdates", Map.of("ENCRYPTION_ENABLED", "true"));
        var input = Map.<String, Object>of("complianceAssessment", assessment);

        WorkerResult<?> result = ComplianceRemediationCaseDescriptor.remediateCompliance(
                input, mockService);

        Map<String, Object> executed = (Map<String, Object>)
                ((Map<String, Object>) result.output()).get("complianceRemediationExecuted");
        assertThat(executed.get("action")).isEqualTo("update-config");
        assertThat(executed.get("controlId")).isEqualTo("encryption-at-rest");
        assertThat(executed.get("serviceId")).isEqualTo("order-api");
        assertThat((List<String>) executed.get("affectedNodeIds"))
                .containsExactlyInAnyOrderElementsOf(returnedNodeIds);
    }

    @Test
    @SuppressWarnings("unchecked")
    void remediateServiceFailureRoutesToEscalation() {
        ApplicationLifecycleService failingService = new ApplicationLifecycleService() {
            @Override
            public Set<String> updateServiceConfig(UUID appId, String serviceId,
                                                    Map<String, String> configUpdates, String tenancyId) {
                throw new IllegalArgumentException("Application not found");
            }
        };

        UUID appId = UUID.randomUUID();
        var assessment = new LinkedHashMap<String, Object>();
        assessment.put("action", "update-config");
        assessment.put("controlId", "encryption-at-rest");
        assessment.put("applicationId", appId.toString());
        assessment.put("serviceId", "order-api");
        assessment.put("tenancyId", "tenant-1");
        assessment.put("configUpdates", Map.of("ENCRYPTION_ENABLED", "true"));
        var input = Map.<String, Object>of("complianceAssessment", assessment);

        WorkerResult<?> result = ComplianceRemediationCaseDescriptor.remediateCompliance(
                input, failingService);

        Map<String, Object> output = (Map<String, Object>) result.output();
        assertThat(output).containsKey("complianceEscalationRequired");
        assertThat(output).containsKey("complianceRemediationError");
    }

    // --- Verify worker ---

    @Test
    void verifyRegistersWithConvergenceTracker() {
        NodeConvergenceTracker tracker = new NodeConvergenceTracker(
                (caseId, path, value) -> {},
                new com.fasterxml.jackson.databind.ObjectMapper()
                        .registerModule(new com.fasterxml.jackson.datatype.jsr310.JavaTimeModule()));

        UUID caseId = UUID.randomUUID();
        WorkerScope scope = new IncidentResponseCaseDescriptorTest.TestWorkerScope(caseId);
        var input = Map.<String, Object>of(
                "complianceRemediationExecuted", Map.of(
                        "action", "update-config",
                        "controlId", "encryption-at-rest",
                        "serviceId", "order-api",
                        "affectedNodeIds", List.of("cluster-1:order-api:deployment")));

        WorkerResult<?> result = ComplianceRemediationCaseDescriptor.verifyCompliance(
                input, scope, tracker);

        assertThat((Map<?, ?>) result.output()).isEmpty();
        assertThat(tracker.isTracking(caseId)).isTrue();
    }

    // --- Escalate worker ---

    @Test
    @SuppressWarnings("unchecked")
    void escalateWritesStatusAndSummary() {
        var input = Map.<String, Object>of(
                "complianceAssessment", Map.of(
                        "controlId", "access-review-quarterly",
                        "controlType", "ACCESS_REVIEW",
                        "outcome", "FAIL",
                        "serviceId", "order-api",
                        "frameworks", List.of("SOC2:CC6.2"),
                        "detail", "Access review overdue"));

        WorkerResult<?> result = ComplianceRemediationCaseDescriptor.escalateCompliance(input);

        Map<String, Object> output = (Map<String, Object>) result.output();
        assertThat(output.get("complianceStatus")).isEqualTo("escalated");
        assertThat(output).containsKey("complianceEscalation");
        Map<String, Object> escalation = (Map<String, Object>) output.get("complianceEscalation");
        assertThat(escalation.get("controlId")).isEqualTo("access-review-quarterly");
        assertThat(escalation.get("risk")).isEqualTo("HIGH");
    }

    @Test
    @SuppressWarnings("unchecked")
    void escalateIncludesRemediationError() {
        var input = Map.<String, Object>of(
                "complianceAssessment", Map.of(
                        "controlId", "encryption-at-rest",
                        "controlType", "ENCRYPTION_AT_REST",
                        "outcome", "FAIL",
                        "serviceId", "order-api"),
                "complianceRemediationError", Map.of(
                        "reason", "Remediation failed: Application not found",
                        "controlId", "encryption-at-rest",
                        "serviceId", "order-api"));

        WorkerResult<?> result = ComplianceRemediationCaseDescriptor.escalateCompliance(input);

        Map<String, Object> output = (Map<String, Object>) result.output();
        Map<String, Object> escalation = (Map<String, Object>) output.get("complianceEscalation");
        assertThat((String) escalation.get("detail")).contains("Remediation failed");
    }
```

- [ ] **Step 2: Run full descriptor tests**

Run: `mvn --batch-mode -o test -pl app -Dtest=ComplianceRemediationCaseDescriptorTest`
Expected: all tests PASS (remediate/verify/escalate tests will compile-fail until `updateServiceConfig` exists — these tests use anonymous subclass override so they compile if the method signature exists, even if the base implementation isn't complete yet. If they fail, proceed to Task 3 first and return.)

- [ ] **Step 3: Wire registrar — replace stub with real descriptor**

In `CaseDefinitionRegistrar.java`, replace line 40:
```java
// Before:
StubChildCaseDescriptor.build("ops", "compliance-remediation", "1.0")
// After:
ComplianceRemediationCaseDescriptor.build(applicationLifecycleService, convergenceTracker)
```

- [ ] **Step 4: Update CaseDefinitionRegistrarTest**

In `registersThreeStubDefinitions`, change the assertion from 3 stubs to 2:
```java
// Before:
assertThat(stubNames).containsExactlyInAnyOrder(
        "cve-response", "service-upgrade",
        "compliance-remediation");
// After:
assertThat(stubNames).containsExactlyInAnyOrder(
        "cve-response", "service-upgrade");
```

Add a new test:
```java
    @Test
    void registersComplianceRemediationWithRealCapabilities() {
        var registry = new RecordingRegistry();
        var registrar = new CaseDefinitionRegistrar(registry,
                new io.casehub.ops.app.service.NodeConvergenceTracker(
                        (caseId, path, value) -> {},
                        new com.fasterxml.jackson.databind.ObjectMapper()
                                .registerModule(new com.fasterxml.jackson.datatype.jsr310.JavaTimeModule())),
                null);

        registrar.onStartup(null);

        var complianceDef = registry.registered.stream()
                .filter(d -> "compliance-remediation".equals(d.getName()))
                .findFirst().orElseThrow();
        assertThat(complianceDef.getTitle()).doesNotContain("stub");
        assertThat(complianceDef.getCapabilities()).extracting("name")
                .contains("assess-compliance", "remediate-compliance");
    }
```

- [ ] **Step 5: Run registrar tests**

Run: `mvn --batch-mode -o test -pl app -Dtest=CaseDefinitionRegistrarTest`
Expected: all tests PASS

- [ ] **Step 6: Commit**

```bash
git add app/src/test/java/io/casehub/ops/app/case_/ComplianceRemediationCaseDescriptorTest.java app/src/main/java/io/casehub/ops/app/case_/CaseDefinitionRegistrar.java app/src/test/java/io/casehub/ops/app/case_/CaseDefinitionRegistrarTest.java
git commit -m "feat(#37): wire ComplianceRemediationCaseDescriptor in registrar

Replace StubChildCaseDescriptor with real descriptor. Add remediate,
verify, and escalate worker tests. Update registrar test assertions.

Refs casehubio/casehub-ops#37"
```

---

### Task 3: ApplicationLifecycleService.updateServiceConfig()

**Files:**
- Modify: `app/src/main/java/io/casehub/ops/app/service/ApplicationLifecycleService.java`
- Modify: `app/src/test/java/io/casehub/ops/app/service/ApplicationLifecycleServiceTest.java`

**Interfaces:**
- Consumes: `ApplicationEntity`, `ServiceDefinition`, `ObjectMapper`, `ApplicationGoalCompiler`, `ReconciliationLoop`, `ClusterService`
- Produces: `updateServiceConfig(UUID applicationId, String serviceId, Map<String,String> configUpdates, String tenancyId) → Set<String>`

- [ ] **Step 1: Write failing tests for updateServiceConfig**

Append to `ApplicationLifecycleServiceTest.java`:

```java
    @Test
    void updateServiceConfigMergesEnvAndReturnsNodeIds() {
        // Create a RUNNING application with a service that has existing env
        var app = createRunningApp("order-api",
                Map.of("DB_HOST", "localhost", "DB_PORT", "5432"));
        UUID appId = app.id;

        Set<String> result = lifecycleService.updateServiceConfig(
                appId, "order-api",
                Map.of("ENCRYPTION_ENABLED", "true", "DB_PORT", "5433"),
                "default");

        assertThat(result).isNotEmpty();
        // Verify the env was merged (not replaced)
        var updatedApp = ApplicationEntity.<ApplicationEntity>findById(appId);
        var services = parseServices(updatedApp.servicesJson);
        var orderApi = services.stream()
                .filter(s -> s.serviceId().equals("order-api")).findFirst().orElseThrow();
        assertThat(orderApi.env()).containsEntry("DB_HOST", "localhost");
        assertThat(orderApi.env()).containsEntry("DB_PORT", "5433");
        assertThat(orderApi.env()).containsEntry("ENCRYPTION_ENABLED", "true");
    }

    @Test
    void updateServiceConfigUnknownServiceThrows() {
        var app = createRunningApp("order-api", Map.of());
        assertThatThrownBy(() ->
                lifecycleService.updateServiceConfig(
                        app.id, "nonexistent", Map.of("KEY", "val"), "default"))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("nonexistent");
    }

    @Test
    void updateServiceConfigUnknownApplicationThrows() {
        UUID bogusId = UUID.randomUUID();
        assertThatThrownBy(() ->
                lifecycleService.updateServiceConfig(
                        bogusId, "order-api", Map.of("KEY", "val"), "default"))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining(bogusId.toString());
    }

    @Test
    void updateServiceConfigEmptyUpdatesReturnsEmpty() {
        var app = createRunningApp("order-api", Map.of("DB_HOST", "localhost"));

        Set<String> result = lifecycleService.updateServiceConfig(
                app.id, "order-api", Map.of(), "default");

        assertThat(result).isEmpty();
    }
```

Note: `createRunningApp` and `parseServices` are test helper methods. If they don't
exist, create them following the pattern of the existing test helpers in the file.
`createRunningApp(serviceId, env)` should create an `ApplicationEntity` with status
RUNNING, persist it, and return it. Check the existing test helpers for patterns.

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn --batch-mode -o test -pl app -Dtest=ApplicationLifecycleServiceTest#updateServiceConfig*`
Expected: compilation failure — `updateServiceConfig` does not exist

- [ ] **Step 3: Implement updateServiceConfig in ApplicationLifecycleService**

Add the method after `rollbackService()` (around line 318). Follow the same
pattern as `updateServiceReplicas()`:

```java
    public Set<String> updateServiceConfig(UUID applicationId, String serviceId,
                                            Map<String, String> configUpdates, String tenancyId) {
        if (configUpdates == null || configUpdates.isEmpty()) return Set.of();

        ApplicationEntity app = ApplicationEntity.findById(applicationId);
        if (app == null)
            throw new IllegalArgumentException("Application not found: " + applicationId);

        List<ServiceDefinition> services = parseServices(app.servicesJson);
        boolean found = false;
        List<ServiceDefinition> updated = new java.util.ArrayList<>();
        for (ServiceDefinition sd : services) {
            if (sd.serviceId().equals(serviceId)) {
                found = true;
                var mergedEnv = new LinkedHashMap<>(sd.env() != null ? sd.env() : Map.<String, String>of());
                mergedEnv.putAll(configUpdates);
                updated.add(new ServiceDefinition(
                        sd.serviceId(), sd.name(), sd.image(), sd.replicas(),
                        sd.ports(), mergedEnv, sd.resources(), sd.dependsOn(),
                        sd.healthCheck(), sd.targetClusters(), sd.restartGeneration()));
            } else {
                updated.add(sd);
            }
        }
        if (!found) throw new IllegalArgumentException("Service not found: " + serviceId);

        try {
            app.servicesJson = objectMapper.writeValueAsString(updated);
        } catch (Exception e) {
            throw new IllegalStateException("Failed to serialize services", e);
        }
        app.persist();

        return recompileAndUpdateLoops(app, tenancyId);
    }
```

Note: `recompileAndUpdateLoops` is a refactored helper that may already exist. If not,
extract the common recompile-and-update pattern from `updateServiceReplicas()` — it
loads clusters, calls `goalCompiler.compileForCluster()`, calls
`reconciliationLoop.updateDesired()`, and collects affected node IDs. Both methods
do the same thing after modifying the service definition. Check the existing code
and either reuse the pattern or extract a shared helper.

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn --batch-mode -o test -pl app -Dtest=ApplicationLifecycleServiceTest`
Expected: all tests PASS

- [ ] **Step 5: Run full app test suite**

Run: `mvn --batch-mode -o test -pl app`
Expected: all tests PASS (verify no regressions)

- [ ] **Step 6: Commit**

```bash
git add app/src/main/java/io/casehub/ops/app/service/ApplicationLifecycleService.java app/src/test/java/io/casehub/ops/app/service/ApplicationLifecycleServiceTest.java
git commit -m "feat(#37): ApplicationLifecycleService.updateServiceConfig()

Merges config key-value pairs into a service's env map, persists the
updated application, and recompiles desired state per cluster. Returns
affected node IDs for convergence tracking.

Refs casehubio/casehub-ops#37"
```
