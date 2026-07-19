# Handoff — casehub-ops

## Last Session
Implemented #59, #60, #61, #62 on one branch (issue-59-approval-and-cdi-fixes). Fixed upstream desiredstate CDI bridge classes (#84), removed App* workaround in ops. Added JPA-backed PlanStore with Jackson mixins for polymorphic spec serialization. K8sApprovalEvaluator now classifies risk by namespace context (configurable critical/production lists). ApprovalAuthorizer SPI with role-based gates wired into ApprovalResource. Squashed and pushed to upstream. 838 tests green before squash. Cross-repo: desiredstate#84 on branch `issue-84-cdi-no-args-constructors` — needs push to upstream.

## Immediate Next Step
Push desiredstate#84 to upstream (`git -C ~/claude/casehub/desiredstate push upstream issue-84-cdi-no-args-constructors`), then pick next work. #25 (fsitrading adaptive ops) is the natural next — all scaling + approval infrastructure now in place. Run `/work` to start.

## What's Left
- desiredstate#84 branch needs push to upstream (local only) · XS · Low
- 11 unstamped closed branches (pre-existing hygiene debt)
- 1 unrecovered blog on closed branch issue-29
- Pre-existing: @QuarkusTest + H2 + Hibernate 6.6 JOINED inheritance DDL failure (GE-20260718-d18dc0)
- ARC42STORIES.MD not yet synced for #43 approval workflow additions · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #25 | fsitrading adaptive ops | L | High | First real consumer — all scaling + approval infrastructure now in place |
| #26 | SOC adaptive ops | L | High | Second consumer |
| #16 | Compliance demo | M | Med | Unblocked — case model + real EvidenceCollectors |
| #17 | Infra demo | M | Med | Unblocked |
| #45 | K8s-aware FaultPolicy responses | M | Med | Needs operational feedback |
| #19 | Integration test hardening | M | Low | Unblocked |

## References
- Architecture: `ARC42STORIES.MD`
- Blog: `blog/2026-07-19-mdp01-four-fixes-one-branch.md`
- Garden: GE-20260718-d18dc0 (H2/Hibernate JOINED inheritance gotcha), GE-20260719-1309d7 (Jackson mixin technique)
