---
layout: post
title: "The A2A card needed more than routing signals"
date: 2026-06-13
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-eidos]
tags: [renderer, a2a, caching, design-review]
---

The issue going in felt clear: `qualityHint`, `latencyHintP50Ms`, `epistemicDomains`, and `costHint` were appearing in PROSE and MARKDOWN renders as raw numbers and "strong expertise" labels. None of that information is useful to an LLM — it's engine dispatch metadata, not behavioural instruction. The fix was obvious: suppress it from LLM-facing formats, keep it in A2A_CARD.

What wasn't obvious was how incomplete the A2A_CARD had become. It carried `qualityHint` but was missing `latencyHintP50Ms`, `costHint`, `epistemicDomains`, `inputTypes`, and `outputTypes`. The capability block in the card was a quarter of what it should be. The design review caught this and pushed the right argument: A2A_CARD is a machine-readable capability card. `slot` and `disposition` are structured objects with vocabulary context precisely because machines benefit from structured data over parsed prose. `inputTypes` and `outputTypes` are capability type schema — an engine routing on type compatibility shouldn't be parsing the prose `description` field to find out what a capability accepts.

That argument was good enough to include the full type set in this branch rather than filing a separate issue for a four-line change.

**The cache trap**

The design went through four rounds of review. Most of the feedback was clear and right: `RESPONSE_FORMAT` schema descriptions were missing from Layer 3's analysis of cache invalidation, the `buildStage1` change wasn't shown explicitly, a test was named after the full cache key when it actually asserted on `descriptorHash`. All fixed.

The interesting moment came in round three. The reviewer noted that `costHint` was dead data to the A2A enrichment LLM — `A2A_PROMPT_TEMPLATE` gives no instruction to use it, the schema constrains output to `{name, description}`. The proposed fix: exclude `costHint` from ALL descriptor payload formats.

I pushed back. `assembleA2aCard()` reads `costHint` directly from the `AgentDescriptor` record — not from `s1.descriptorNode()`. The descriptor hash is computed from `buildDescriptorPayload(descriptor, format)`. If `costHint` is excluded from the A2A_CARD descriptor payload, changing `costHint` produces no change in the hash, the cache key is identical, and a cache hit serves a stale A2A card with the old value. The fix would have been correct for LLM noise reduction and wrong for cache correctness.

The structural assembly bypass is the trap: the payload builder looks authoritative, but any field that any assembly step reads from the source record must be in the hash — regardless of whether the LLM enrichment path cares about it. The rule went into the garden as GE-20260613-3fa95a and into the project protocols as PP-20260613-608684.

**The format-discriminated payload**

The main design choice was where to apply the format discrimination. The instinct is to strip at the LLM call boundary — build one format-agnostic payload, filter before sending to the LLM. The problem: the hash is computed from the full payload, so format-irrelevant field changes still invalidate all format caches. Changing `qualityHint` would flush PROSE and MARKDOWN cache entries even though those formats don't render it.

Making `buildDescriptorPayload(descriptor, format)` format-aware solves both at once. The hash input is format-specific, so PROSE and MARKDOWN cache entries are unaffected by numeric field changes. The LLM payload is already correct — there's nothing to strip at the call boundary. `buildStage1()` passes `context.format()` through, and the rest composes cleanly.

The implementation touched four layers: the payload builder, the structural MARKDOWN fallback, the `PROMPT_TEMPLATE` and `RESPONSE_FORMAT` schema description, and `assembleA2aCard()`. All four compiled and passed on the first run. The interesting work had already been done in the spec.
