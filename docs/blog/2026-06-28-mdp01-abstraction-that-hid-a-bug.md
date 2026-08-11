---
layout: post
title: "CaseHub Ops — The Abstraction That Hid a Bug"
date: 2026-06-28
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ops]
tags: [drift-detection, multi-tenancy, functional-interface, abstraction]
---

# CaseHub Ops — The Abstraction That Hid a Bug

**Date:** 2026-06-28
**Type:** phase-update

---

## What I was trying to achieve: enrich the ChannelDriftChecker

ops#14 came from a redirect — qhorus#287 originally proposed a bridge module in qhorus itself, but that created an upward coupling from Foundation to Integration tier. The enrichment belonged in ops, where `ChannelDriftChecker` already lived.

The issue had six requirements: expand field comparison from 4 to 8 fields, add connector binding drift detection, fix a tenancy gap, handle CSV set ordering, detect reverse binding asymmetry, and clean up a stale PLATFORM.md reference.

## What I found going in: a cross-repo sweep before the work

Before starting ops#14, I ran a scan across every casehub workspace HANDOFF.md and every repo's open issues. The goal was to find stale ops references and gaps — work that implied ops issues but hadn't been filed.

Two finds. platform#117 (verify EndpointRegistered CDI event fires) was already implemented — `InMemoryEndpointRegistry.register()` had the `Event.fireAsync()` call with error logging. Closed it. More interesting: desiredstate#14 (ReconciliationLoop PendingApproval workflow) has downstream implications for ops that nobody had tracked. When the runtime ships PendingApproval handling, ops provisioners need to implement the approval-side lifecycle. Filed ops#13 for that.

## The functional interface that masked a tenancy parameter

The interesting part of ops#14 wasn't the field comparison — that was mechanical. It was the tenancy bug.

`ChannelDriftChecker` had a nested `@FunctionalInterface`:

```java
@FunctionalInterface
public interface ChannelLookup {
    Optional<Channel> findByName(String name);
}
```

Clean design. Single-method interface. Easy to test with lambdas. And it was hiding a bug.

The `check()` method received `tenancyId` as a parameter. But `ChannelLookup.findByName()` had no slot for it — the interface predated multi-tenancy. So `tenancyId` was passed in, received, and silently ignored. A channel in tenant A could satisfy a spec intended for tenant B.

The abstraction looked correct at every level. The interface compiled. The implementation compiled. Tests passed — because the test stubs didn't enforce tenancy scoping either. The bug was invisible at the type level, only visible by tracing the data flow from `check()` through the lookup to the store.

The fix was to delete `ChannelLookup` entirely and inject `CrossTenantChannelStore` directly. `findByNameAndTenancy(name, tenancyId)` makes the tenancy parameter visible and required at the call site. No abstraction layer, no parameter to forget. This follows the pattern `AgentDriftChecker` already uses — direct injection of `AgentRegistry` with no wrapper.

## What it is now

`ChannelDriftChecker` compares 8 mutable channel fields (including CSV set comparison that handles `"a,b"` vs `"b,a"` without false drift), checks 4 connector binding fields with reverse asymmetry detection, and logs every mismatched field at DEBUG. 18 tests cover the full matrix.

The functional-interface gotcha went into the garden as GE-20260628-f5c99f. The general lesson: when you wrap a store lookup in a `@FunctionalInterface` for testability, you freeze the query's parameter surface. If the underlying store later needs more context — tenancy, scope, locale — the interface has no slot for it, and the caller has no way to pass it. The type system won't warn you. The tests won't catch it. The abstraction that was supposed to make things cleaner silently drops the parameter you most need.
