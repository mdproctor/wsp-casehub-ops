# Design Journal — issue-10-s-issues-batch

### 2026-07-21 · §9.4·L5 IoT

IoTFaultPolicy moves from no-op to fault-count escalation. After 3 consecutive
PROVISION_FAILED events on a device-config node, the policy adds an `iot-review`
node (HumanGating.ALL) to the graph — the runtime creates a WorkItem for human
investigation. This follows the ProvisionEscalationFaultPolicy pattern from the
desiredstate examples (fault count tracking, idempotent review node creation,
infinite-regress guard on review nodes). IoTNodeProvisioner gains `iot-review`
in handledTypes() and handles IoTReviewSpec as a no-op Success — the review is
complete once the human approves the WorkItem. New IoTReviewSpec record in api/iot.

### 2026-07-21 · §9.4·L4 Compliance

ComplianceNodeProvisioner overrides resyncInterval() to return 1 hour. The default
5-minute interval was too aggressive — compliance evidence staleness is measured in
days (evidenceMaxAgeDays), not minutes. A 1-hour resync catches drift within an
acceptable window without wasting cycles re-evaluating evidence that is valid for
days.
