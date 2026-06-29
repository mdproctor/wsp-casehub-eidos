---
layout: post
title: "Eidos — Backlog Zero"
date: 2026-06-29
type: phase-update
entry_type: note
subtype: diary
projects: [Eidos]
---

# Eidos — Backlog Zero

**Date:** 2026-06-29
**Type:** phase-update

---

## What I was trying to achieve: take stock of where eidos stands

I came back to eidos after a few days away. The previous session had shipped `AgentDescriptorComparator` for drift detection and closed out the desiredstate bridge work. Time to see what's left.

## The backlog hits zero

One open issue: #28, Belbin-based agent composition for casehub-engine. I checked its dependencies — #26 (Belbin vocabulary) and #27 (theoretical framework grounding) both shipped weeks ago. The proposed work in #28 — `PhaseRoleRequirement`, `BelbinAwareRoutingStrategy`, phase context threading through `WorkOrchestrator` — all lives in engine, not eidos. The issue was filed here to track the eidos prerequisites, and those prerequisites are done.

I closed #28 and filed [engine#577](https://github.com/casehubio/engine/issues/577) to capture the engine-side work: phase-aware agent selection by Belbin archetype, team composition constraints, and the routing infrastructure it depends on (engine#505).

Eidos's backlog is empty. Every open issue has been closed or moved to the repo that owns the remaining work.

## A hidden compile error

The full-reactor aggregator build caught something `-pl` had been hiding: a switch expression in eval's `AgentProviderChatModel` only covered `TextDelta`, but `AgentEvent` is a sealed interface with five permitted subtypes. Java 21 requires exhaustive switches, and the error had been invisible because incremental builds with `-pl eval` don't compile test sources in non-targeted submodules.

The fix was a `default -> ""` branch — eval only extracts text content. But the discovery is the interesting part: sealed interface exhaustiveness is a cross-module contract, and `-pl` silently opts you out of checking it in sibling test sources. You can go multiple sessions without noticing.

## Where this leaves eidos

The library is feature-complete for its current scope: four-layer agent descriptors, pluggable vocabulary, capability health with learned specialization, system prompt rendering in three formats, and an eval harness. The remaining questions — positive specialization learning, semantic capability matching, behavioral contracts — are all about what happens when the ecosystem starts using it at scale. Engine's routing strategy (#505, #577) is the first real consumer beyond the eval harness. That's where the next pressure will come from.
