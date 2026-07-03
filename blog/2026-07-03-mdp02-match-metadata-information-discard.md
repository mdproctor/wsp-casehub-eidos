---
layout: post
title: "Match Metadata and the Information-Discard Anti-Pattern"
date: 2026-07-03
type: phase-update
entry_type: note
subtype: diary
projects: [eidos]
tags: [api-design, owls-mx, capability-matching]
---

Three layers of the eidos stack computed match information — which capability matched, how well it matched — and then threw it away. `CapabilityResolver.resolve()` iterated each declared capability, computed an OWLS-MX `MatchDegree`, picked the best, and returned only the bare `AgentCapability`. The degree was gone. `AgentRegistry.find()` used that same matching logic as a boolean filter — yes or no, you matched — and returned a flat `List<AgentDescriptor>`. The engine's `AgentCandidateFactory` then had to call `resolve()` *again* just to log which capability had matched in the first place.

I'd noticed the problem during the #76 brainstorming — the issue was filed, parked, and now it was time to fix it.

The design started at the root. `CapabilityResolver.resolve()` already does the hard work; it just doesn't tell you the result. A new `ResolvedCapability(capability, degree)` record captures what `resolve()` already computes. `AgentMatch(descriptor, resolvedCapability)` wraps that for registry results. `MatchDegree` becomes `Comparable` — Exact beats Plugin beats Specialization beats None — which is definitional to OWLS-MX, not a policy choice.

The design review surfaced something I hadn't noticed: the existing `resolve()` loop had a latent bug. Plugin and Specialization competed on the same `int bestDepth` variable, so a `Specialization(1)` could incorrectly beat a `Plugin(2)`. The `Comparable`-based selection fixes this as a side effect — the `compareTo()` on `MatchDegree` encodes the full type-then-depth ordering, so the old `instanceof` cascade for Plugin and Specialization collapses into a single comparison.

The implementation was 8 commits across 27 files. The change is a breaking SPI update to `AgentRegistry.find()` — every caller now gets `List<AgentMatch>` instead of `List<AgentDescriptor>`. There are no external consumers, so the break costs nothing. The subsumption scenario test that previously commented "ranking is a future enhancement" now asserts exact OWLS-MX ordering with degree verification.

What the engine gets from this is the ability to prefer exact matches over subsumption matches at dispatch time without re-deriving the match. `AgentCandidateFactory` currently calls `CapabilityResolver.resolve()` to log the match — with `ResolvedCapability` as the return type, it can carry the degree into `AgentCandidate` and let routing strategies rank on it directly. That's engine#638 and #639.
