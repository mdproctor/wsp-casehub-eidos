# Enrichment Rethink — Design Spec

**Date:** 2026-06-16  
**Status:** Approved  
**Context:** Emerged from Phase 2 eval analysis — independent judge run, proximity scoring investigation, and systematic audit of the enrichment pipeline against its original design intent.

---

## Problem

Three separate issues were found during eval analysis. They compound each other but have independent fixes.

### 1. Enrichment scope crept beyond its original intent

The original spec (`eidos.md`, Phase 3 section) said:

> *"LLM prose layer: disposition section only — renders socialOrient + ruleFollowing + situationalContext into 2–3 natural language sentences."*

What was implemented: 6 narrative fields covering identity, role, capability, disposition, constraint, and goal. The scope expansion is undocumented — no ADR, no spec revision.

The eval shows the extra scope produces no value: identity, role, and capability sections scored 4.97–5.0 structurally. The LLM is rephrasing schema fields that already render well without it.

### 2. Enrichment mechanics are broken for Claude Opus 4.6

`SemanticEnrichmentStep.stripCodeFences()` only handles the outer fence. It does not extract JSON from within prose preamble. Claude Opus 4.6 frequently returns responses starting with "Project...", "The...", "CLAUDE..." — prose that has no JSON at all — or wraps JSON in code fences with a preamble. Both fail silently as structural fallbacks.

Result: enrichment never runs in practice. All renders are structural.

### 3. Proximity eval measures the wrong target

Proximity compares renders against `originalProse`. But `originalProse` was the source used to *create* the descriptor — not the rendering target. Every profile's `vocabularyGaps` field explicitly documents what the descriptor *cannot* express:

- `velocity-as-feature` → FULL LOSS  
- `six-month maintainability horizon` → FULL LOSS  
- `trust-in-pipeline` → FULL LOSS

These are intentional losses. No enrichment can recover them — they are not in the descriptor. The proximity metric is testing fidelity to information that was deliberately excluded.

---

## Quality Target

**The renderer's job is: produce the best system prompt expressible from the `AgentDescriptor` as given.**

The descriptor is the truth. If `riskAppetite: bold` is in the descriptor and "90% elegant beats perfect" is not, the render correctly omits that principle. The renderer is not a reconstruction engine.

`originalProse` remains in eval profiles as research material and as a source for `briefing` content when apps choose to use it — but it is not the rendering target and must not be the eval target.

---

## Changes

### Change 1 — Narrow enrichment scope to disposition + goal

Remove `identityNarrative`, `roleNarrative`, `capabilityNarrative`, `constraintNarrative` from the enrichment response schema. Keep only `dispositionNarrative` and `goalNarrative`.

**Rationale:** The four removed sections render correctly and completely from structured fields alone (eval score 4.97–5.0). The enrichment call adds no measurable value. Disposition and goal are the sections where natural language elaboration matters:

- Disposition: open strings (`bold`, `conservative`, `strict`) benefit from contextual elaboration. "riskAppetite: bold" → "comfortable approving changes that push boundaries when fundamentals are sound" is genuine value. Structural fallback: list axes as plain prose.
- Goal: sub-goals benefit from flowing prose rather than bullet enumeration. Structural fallback: join sub-goals with semicolons.

Identity, role, capability, and constraint sections become deterministic structural assembly — no LLM involvement.

**Updated `SemanticEnrichment` record:**

```java
record SemanticEnrichment(
    Optional<String> dispositionNarrative,
    Optional<String> goalNarrative
) {}
```

**Updated `RESPONSE_FORMAT` schema:** two optional string fields only.

**Updated `PROMPT_TEMPLATE`:** rewritten to describe only disposition and goal fields.

**Updated `TEMPLATE_HASH`:** recomputed from new template — invalidates all existing cached renders.

### Change 2 — Fix enrichment mechanics

Apply the same resilience fixes used for the eval judges:

1. In `SemanticEnrichmentStep.parse()`: replace `stripCodeFences()` with `PromptJudge.extractJson()` — or extract `extractJson()` to a shared utility class. The shared utility handles both code fences and prose preamble by extracting between the outermost `{` and `}`.

2. In `SemanticEnrichmentStep.enrich()`: add one retry on `JsonProcessingException` before returning `Optional.empty()`. Log the retry at WARN level. The second failure falls back to structural — no change to existing fallback behaviour.

**Note:** The enrichment prompt already says "Return ONLY the JSON object. No explanation, no preamble, no code fences." The mechanical fix is applied first. If the retry still produces non-JSON at measurable rate after validation, the prompt will be strengthened — but this is deferred until the eval shows whether it is needed.

### Change 3 — Redefine proximity eval target

Replace `originalProse` as the proximity comparison target with **descriptor fidelity**: does the rendered disposition section correctly express the disposition axes that are in the descriptor?

**New proximity question per case:**  
For each disposition axis present in the descriptor, does the render correctly convey that axis value? Scored 0–5 as before, but the judge prompt changes to evaluate fidelity to the structured descriptor, not similarity to the original prose.

**Updated `ProximityJudge` system prompt:** rewritten to instruct the judge to compare the render against the descriptor's disposition fields, not against any prose original.

**Updated `ProfiledEvalCase` / `RealWorldEvalDataset`:** proximity cases now pass the descriptor's disposition fields as the reference, not `originalProse`.

`originalProse` remains in profile YAML files — it is not removed. It is simply no longer used as the proximity eval reference.

### Change 4 — Add `AgentDescriptor.briefing` (separate issue)

A nullable free-text field for behavioral principles the structured axes cannot express.

```java
record AgentDescriptor(
    // ... existing fields ...
    String briefing   // nullable — "90% elegant beats perfect. Speed is a feature."
)
```

**Behaviour:**
- If present, renders as a final sentence in the disposition section after the enriched (or structural) axis prose.
- No LLM involvement — direct pass-through.
- Consuming apps set this at registration time.
- Null/absent: no change to current behaviour.

**Eval profiles:** `vocabularyGap: FULL` entries become candidates for `briefing` values. Profile YAML gains an optional `briefing` field that maps to `AgentDescriptor.briefing` in the descriptor.

**Scope:** this change is separable from Changes 1–3. Implement after the enrichment fix is validated by the eval harness.

---

## What Does Not Change

- `SystemPromptRenderer` SPI and `RenderedPrompt` record — unchanged externally
- `RenderedPromptCache` SPI and cache key structure — unchanged (TEMPLATE_HASH changes, which correctly invalidates existing cache entries)
- Format assembly (Stage 3) — structural paths for identity, role, capability, constraint are unchanged; only the disposition and goal assembly changes to accept two-field `SemanticEnrichment`
- `A2ASemanticEnrichmentStep` — out of scope; handles per-capability descriptions, separate concern
- Structural fallback contract — any enrichment failure still produces correct structural output

---

## Issues

| # | Scope | Depends on |
|---|-------|-----------|
| New: enrichment-mechanics | Changes 1 + 2: narrow scope + fix extractJson/retry | — |
| New: proximity-eval-redesign | Change 3: redefine proximity target | enrichment-mechanics (so eval runs against fixed enrichment) |
| New: briefing-field | Change 4: `AgentDescriptor.briefing` | proximity-eval-redesign (to validate briefing improves score) |

---

## Validation

After implementing Changes 1–2, re-run `evaluateRealWorldScenarios` with Claude. Enrichment should now succeed for disposition sections. Compare:

- Enrichment success rate (currently 0%; target: >80%)  
- Quality scores — should be unchanged or improved (structural sections already 4.97–5.0)
- Disposition section prose — spot-check for naturalness and axis accuracy

After implementing Change 3, re-run proximity eval. Proximity scores should improve from 2.81 (current) — because the descriptor fields ARE correctly expressed in renders, the gap was in the originalProse target.

After implementing Change 4, re-run with `briefing` populated in profiles. Proximity scores for `FULL LOSS` gap cases should improve further.
