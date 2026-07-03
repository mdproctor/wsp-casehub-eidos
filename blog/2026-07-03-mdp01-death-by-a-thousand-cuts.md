---
title: "Death by a Thousand Cuts"
date: 2026-07-03
author: mdp
tags: [eidos, behavioral-contracts, aggregate-threshold, design-review]
---

The per-dimension compliance threshold from yesterday's behavioral contracts work had a known blind spot: an agent with 2 violations in each of 10 dimensions — 20 total, all individually below the threshold of 3 — looked perfectly compliant. The design review caught the gap and filed it as #88.

The fix is an aggregate safety valve. Sum all VIOLATED counts across all compliance dimensions; if the total exceeds a second threshold (default 5), fire. The key insight: the aggregate only fires when per-dimension hasn't — every dimension's count is below the per-dimension threshold. So summing counts directly measures "total behavioral drift below the per-dimension radar."

The design needed a discriminator on `BehavioralViolation` — `ViolationKind { PER_DIMENSION, AGGREGATE }` — because the violations map has different semantics per kind. PER_DIMENSION contains only the exceeding dimensions. AGGREGATE contains all dimensions with any violations. Without the discriminator, consumers can't interpret the map correctly.

The co-tuning guidance between the two thresholds was the trickiest part. The design review caught two mathematical errors in my initial formula. The correct general formula for ensuring at least K dimensions must contribute: `aggregate >= (K-1) * (per_dimension - 1) + 1`. At the defaults (per-dimension=3, K=3), that gives aggregate >= 5.

Also closed #86 as an ADR: no compliance status in A2A_CARD. The card represents what an agent declares; compliance is runtime state about whether it lives up to those declarations. Different lifecycles, caching would break, and `probe()` already gives consumers the same information. The right architectural answer is that these are separate concerns accessed through separate APIs.

The rest was cleanup from the #85 design review: unified `InMemoryAgentRegistry.matchesCapability()` to delegate to `CapabilityResolver.match()` (#83), renamed stale test inner classes (#91), and updated ARC42STORIES.MD and PLATFORM.md for the behavioral contracts framework.
