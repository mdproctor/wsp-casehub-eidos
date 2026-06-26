---
layout: post
title: "When the Bridge Module Died"
date: 2026-06-26
type: phase-update
entry_type: note
subtype: diary
projects: [eidos]
tags: [cdi, architecture, drift-detection, design-review]
series: issue-060-desiredstate-bridge
---

I started issue #60 planning to build `casehub-eidos-desiredstate` — an optional bridge module that would displace the ops drift checker via CDI `@Alternative @Priority(1)`. The existing `AgentDriftChecker` compares sorted capability names only. A bridge module felt natural: eidos owns the descriptor types, so eidos should own the comparison logic. Drop the jar on the classpath, the richer checker activates, the default vanishes.

Three rounds of spec review killed that design.

## The CDI trap

The first review caught something I'd missed entirely. I'd proposed making the existing `AgentDriftChecker` a `@DefaultBean` so the bridge module could displace it. The problem: `@DefaultBean` suppression operates on interface types, not concrete classes. `DeploymentActualStateAdapter` collects all `NodeDriftChecker` implementations via `Instance<NodeDriftChecker>` — four beans, each serving a different `nodeType()`. Making `AgentDriftChecker` a `@DefaultBean` would cause `ChannelDriftChecker` (same interface) to trigger suppression. Agent drift checking silently stops working even without the bridge module on the classpath.

This is a genuinely nasty failure mode. `@DefaultBean` works correctly for single-injection-point replacement — six SPIs in eidos use exactly this pattern. The mental model transfers. But `Instance<T>` multi-bean collection with a runtime discriminator is structurally different, and there's no warning when you get it wrong. The bean vanishes, the `nodeType()` key is never checked, and nobody throws an exception.

The alternative — `@Alternative @Priority(1)` — has its own problem. Both the existing checker and the bridge checker appear in the `Instance` iteration with the same `"agent"` key. `HashMap.put()` picks one non-deterministically.

## Where comparison logic actually belongs

With CDI displacement off the table, the reviewer pushed harder: should this be a bridge module at all? `AgentNodeSpec` already imports `AgentCapability`, `AgentDisposition`, and `DispositionAxis` from eidos-api. The dependency path already exists. When eidos adds a field to `AgentDescriptor`, `AgentNodeSpec` must be updated in ops regardless — so the "co-evolution" argument for a separate module doesn't hold.

I considered a composition approach — `AgentNodeSpec` wrapping `AgentDescriptor` instead of duplicating its 17 fields. That would eliminate field duplication at the source and make comparison trivial. But `AgentDescriptor`'s compact constructor requires `tenancyId`, which is a deployment-time concern absent from the spec. Making it optional would weaken a platform-wide invariant for the sake of one consumer.

The answer turned out to be simpler. `AgentDescriptorComparator` goes in eidos-api as a pure Java utility — type owner defines content equality, same pattern as `AgentDescriptorValidator`. The ops `AgentDriftChecker` calls it. No new module, no CDI, no displacement mechanics.

## Structural sync — guarding the comparator

The comparator handles 16 of 18 record components (skipping `agentId` and `tenancyId`). When someone adds a 19th component, the comparator silently ignores it — no compile error forces an update.

The fix is a reflection-based count test: declare `COMPARED_FIELD_COUNT = 16` on the comparator and assert it equals `AgentDescriptor.class.getRecordComponents().length - 2`. Three constants, three tests — one each for `AgentDescriptor`, `AgentCapability`, and `AgentDisposition`. The pattern generalises to any utility that must track a record's fields across module boundaries.

The `tenancyId` question keeps surfacing. It's arguably a scoping key — where the agent is deployed — not an identity attribute. Extracting it from `AgentDescriptor` would enable the composition approach and make content equality just `descriptor.equals(registered)`. That's a platform-wide rearchitect, filed separately. The comparator works within current constraints.
