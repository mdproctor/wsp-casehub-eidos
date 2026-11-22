---
layout: post
title: "The Unit That Vouches Too Much"
date: 2026-11-22
entry_type: note
subtype: diary
projects: [casehubio/eidos]
tags: [trust, anti-collusion, attestation-relevance, independence]
series: issue-257-independence-aware-anti-collusion-factor
---

# The Unit That Vouches Too Much

The org-proximity factor we shipped last session gives same-unit attestors full weight — 1.0 — because they share operational context with the agent they're attesting about. That's correct on accuracy grounds. An engineer on your team has seen your work up close; someone three divisions away hasn't.

But accuracy and independence are in tension. If eight out of ten attestations for an agent come from its own unit, the trust score is dominated by correlated observations. Those eight attestors share the same context, see the same outputs, and form similar opinions. The effective information content isn't eight independent signals — it's closer to two or three, inflated by repetition.

This is the anti-gaming gap the ledger ARC42 flags under Active Risks: a unit of agents whose positive attestations correlate will produce artificially high trust scores. The identity layer handles Sybil attacks (synthetic agents rating each other), but social gaming — real agents with correlated viewpoints — has been open.

## The Cap Formula

The fix is a share cap. If same-unit attestors exceed a configured threshold (default 60%) of the known attestation group, their weights are reduced proportionally. The formula falls out of basic algebra.

Given `S` same-unit attestations weighted at `w` and `C` cross-unit attestations at 1.0, the target is:

```
S × w / (S × w + C) = cap
```

Solving for `w`:

```
w = cap × C / ((1 - cap) × S)
```

With 8 same-unit and 2 cross-unit at a 60% cap: `w = 0.6 × 2 / (0.4 × 8) = 0.375`. After weighting, same-unit mass is 3.0, cross-unit is 2.0, and the share hits 60% exactly. The existing composer floor (0.5) bounds the composed product when multiple factors stack, so extreme imbalance doesn't zero out same-unit evidence entirely.

Three edge cases shaped the design. When all attestations are same-unit, there's no cross-unit evidence to protect — the factor returns NEUTRAL, because penalizing without a redistribution target just destroys mass. When the share is at or under the cap, everything passes through unchanged. And attestations without org-distance in their snapshot (unknown relationship) are excluded from the share calculation entirely — they can't be classified either way.

## The Batch Problem

The existing factor SPI has two entry points: `assess()` for per-attestation evaluation, and `assessBatch()` as a default method that loops `assess()`. The org-proximity factor uses `assess()` — it only needs the individual attestation's snapshot to look up its weight.

Independence is different. You can't compute "this group is 80% same-unit" when you only see one attestation at a time. The factor needs `assessBatch()` to see the full group and compute share ratios.

The catch: the `RelevanceComposer` in ledger-core was calling `assess()` per attestation, never `assessBatch()`. The default method existed but nothing invoked it. We refactored the composer to call `assessBatch()` per factor instead — a ~10-line change that's fully backward-compatible because the default `assessBatch()` loops `assess()`. Existing factors work identically; factors that override `assessBatch()` now get the batch they need.

## What the Distance Means

I spent time investigating the org-distance algorithm from first principles before deciding what "same-unit" means for the cap. The algorithm is LCA-based — it finds the Lowest Common Ancestor of two agents' organizational units and sums the edge distances.

Distance 1 is the crisp boundary: two distinct agents in the same `OrganizationalUnit`. They share operational context directly — same team, same work, same observations. Distance 2 means sibling units under the same parent, structurally separate vantage points. The correlation drops sharply at that boundary.

Multi-unit membership takes the minimum distance across all pairings. An agent in both "eng" and "sales" has distance 1 to any "eng" member. That correctly counts them as same-unit for anti-collusion purposes — their shared "eng" context is real regardless of their other memberships.

## Where This Sits

The independence factor and the proximity factor operate on different concerns through the same SPI. Proximity handles accuracy: same-unit attestors have better context, so their individual attestations weigh more. Independence handles diversity: when too many attestations come from the same unit, their collective contribution is capped. Both weights multiply in the composer, and the trust computer's mass-preserving normalisation redistributes mass toward cross-unit attestors.

Qhorus has a separate anti-collusion mechanism at the credibility level — `CollusionAwareCredibilityPolicy` flags mutual-endorsement pairs. That's about individual attestor suspicion. This is about group concentration. The two are complementary: credibility asks "is this attestor suspicious?", independence asks "is this group of attestors too homogeneous?"

The factor went in as 19 tests covering the full edge-case matrix: under cap, over cap, at cap exactly, all-same-unit, all-cross-unit, unknowns excluded from share, extreme imbalance, NaN rejection. Claude caught the NaN guard gap during review — `Double.NaN` passes range comparisons because NaN is unordered in IEEE 754, so the constructor needed an explicit `isNaN()` check to match the existing pattern in `OrgProximityRelevanceFactor`.

The anti-gaming gap isn't closed. This is the first structural countermeasure — it caps concentration but doesn't detect coordination. A unit whose members deliberately inflate each other's scores would still benefit up to the cap threshold. But 60% is a lot less than 100%, and the cap composes with credibility-level collusion detection to narrow the opening further.
