---
layout: post
title: "IoT desired state — the type boundary nobody warns you about"
date: 2026-06-25
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ops]
tags: [iot, desired-state, bigdecimal, type-normalization]
series: issue-4-iot-desired-state
---

*Part of a series on [#4 — Epic 3: IoT desired state domain](https://github.com/casehubio/casehub-ops/issues/4). Previous: [ARC42STORIES.MD migration](2026-06-23-mdp01-arc42stories-migration.md).*

## The type boundary nobody warns you about

The IoT module is the fourth domain in casehub-ops — after infra, deployment, and compliance. By now the SPI quad pattern (GoalCompiler, ActualStateAdapter, NodeProvisioner, FaultPolicy, EventSource) is well-established. I expected IoT to be the quickest: casehub-iot already provides DeviceProvider, DeviceRegistry, StateChangeEvent, and DeviceCommand. The infrastructure maps directly to the desiredstate SPIs.

It didn't turn out that way. The interesting problems were all at the type boundary between YAML-parsed values and casehub-iot's typed domain model.

## Two node types, derived from bytecode

The issue spec said two node types: physical device (human must install) and logical configuration (DeviceProvider.dispatch). The design question was whether to use one node per device with dynamic routing, or two nodes with an explicit dependency edge.

I read the `SimpleTransitionExecutor` bytecode to settle it. The executor checks `requiresHuman` at compile time — if true, it returns `StepOutcome.Skipped("requires human")`. No WorkItem, no callback, no notification. Just skipped. That means `requiresHuman` must be set when the GoalCompiler builds the graph, not decided at provisioning time.

Two nodes with a dependency: the physical-device node (requiresHuman=true) tracks whether the device exists, and the device-config node (requiresHuman=false, depends on physical) tracks whether the configuration is correct. When the runtime eventually adds WorkItem integration for human nodes, the physical-device path works automatically — zero changes to our module.

## The BigDecimal trap

This is where the session got genuinely tricky. YAML parses `22` as `Integer`. Temperature uses `BigDecimal`. And `BigDecimal.equals()` is scale-sensitive — `new BigDecimal("22.0").equals(new BigDecimal("22"))` returns false.

The capability comparison needed to work across both sides: actual (from `DeviceEntity.capabilities()`, which puts Temperature objects and ThermostatMode enums into the map) and desired (from YAML, which produces Integers, Strings, and LinkedHashMaps). We wrote a CapabilityNormalizer that converts both sides to a canonical form — all numerics to `BigDecimal.stripTrailingZeros()`, all enums to their `name()` string, and recursive descent into nested Maps for compound types like Temperature.

The integration test surfaced one more gotcha: `DeviceEntity.capabilities()` includes null values for Optional fields (LightDevice puts null for `brightness` and `colorTemp` when they're unset). `Map.copyOf()` throws NPE on null values — no error message, just a raw NullPointerException from the JDK internals. The fix is to filter nulls before copying.

## DRIFTED works now

The spec initially documented desiredstate#38 (TransitionPlanner DRIFTED handling) as an open limitation. During the fourth round of spec review, re-reading the TransitionPlanner bytecode showed the fix had landed: `case DRIFTED, ABSENT, UNKNOWN -> true` in the provision switch. Config drift self-healing works today — the NodeProvisioner dispatches DeviceCommands for drifted capabilities without waiting for any runtime changes.

The one remaining runtime gap is requiresHuman → WorkItem creation (#43). Physical device nodes are silently skipped until that ships. The design is forward-compatible — when it lands, no code changes needed.

## What the module looks like

Sealed `IoTNodeSpec` hierarchy with `PhysicalDeviceSpec` and `DeviceConfigSpec`. Single `IoTDeviceGoal` record with boxed `Boolean physical` (null from YAML → defaults to true in the compact constructor — primitive boolean would silently default to false). GoalCompiler produces two nodes per device when physical=true, one node when physical=false. NodeProvisioner routes to DeviceProvider by `providerId`, normalises capabilities on both sides, dispatches DeviceCommands for the diff.

The four spec review rounds tightened the integration details: Temperature reconstruction from YAML maps, `turnOn()` taking a Map parameters argument while `turnOff()` does not, `dispatchedBy` and `correlationId` fields for audit trail, and provider selection via `device.providerId()` → `Map<String, DeviceProvider>` lookup.

122 tests across the api and iot modules. The full casehub-ops build passes.
