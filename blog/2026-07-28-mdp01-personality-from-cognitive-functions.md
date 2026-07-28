---
layout: post
title: "Personality from Cognitive Functions, Not Labels"
date: 2026-07-28
type: phase-update
entry_type: note
subtype: diary
projects: [eidos]
tags: [personality, jungian, mbti, jpaf, vocabulary, disposition]
series: issue-107-jungian-personality-framework
---

Six months ago I rejected MBTI as a vocabulary source for eidos. The rejection was sound: ~50% of people receive a different type when retested a month later. Any vocabulary built on that instability is unreliable. I wrote it up in `personality-frameworks.md` and moved on to DISC and Conscientiousness, which have better psychometric properties.

Then I read the JPAF paper.

## The Insight That Changes the Framing

JPAF (Structured Personality Control and Adaptation for LLM Agents, arXiv:2601.10025) demonstrates something I hadn't considered: the test-retest critique assumes personality is *measured*. For LLM agents, personality is *specified*. You declare "this agent uses Ti-dominant, Ne-auxiliary" and inject it via structured prompting. There is no measurement error because there is no measurement. The instability is in the assessment instrument, not in the type system itself.

More importantly, JPAF doesn't operate at the MBTI level at all. It operates at the Jungian cognitive function level — the eight functions (Ti, Te, Fi, Fe, Si, Se, Ni, Ne) that underpin Jung's original typology. Each function is a cognitive processing mode: Ti builds internal logical frameworks, Fe harmonises group values, Ni synthesises patterns into singular insights, Se focuses on immediate sensory data.

MBTI's four-letter codes (INTJ, ENFP, etc.) are emergent properties of how these functions are stacked. An INTJ is Ni-dominant, Te-auxiliary — the letters fall out of the function stack, not the other way around. The JPAF approach assigns continuous weights to all eight functions (dominant ~0.35, auxiliary ~0.20, remaining six sharing the rest) and lets the MBTI type emerge dynamically. This achieves 100% MBTI personality alignment across GPT-4, Llama, and Qwen — significantly stronger than injecting the four-letter code directly.

A separate paper published this month — "The Geometry of Personality" (arXiv:2607.20803) — independently validates the same eight-function model via activation steering rather than prompting. Two different research groups, two different approaches, same underlying model. That's strong validation.

## What Eidos Already Had

Eidos's disposition model uses five behavioural axes — social orientation, rule following, risk appetite, autonomy, and conflict mode — each with a single categorical value from a vocabulary. An agent might be `socialOrient: independent, ruleFollowing: strict, riskAppetite: conservative`. DISC, Conscientiousness, Belbin, and Thomas-Kilmann all provide values on these axes.

This works well for expressing *what* an agent does. It doesn't express *how* the agent thinks. An INTJ and an ISTP both map to `independent` on social orientation, but they arrive there through completely different cognitive paths — Ni (abstract pattern synthesis) versus Ti (concrete logical analysis). The axes are a behavioural surface; the cognitive functions are the underlying model.

## The Design: String + Weight

The existing model gives the LLM a string per axis. The evolution: give it a string plus a degree. Every disposition value becomes `DispositionValue(String term, double weight)`. This is backward compatible in spirit — existing profiles just use weight 1.0 — and opens three new capabilities:

**Weighted axes.** A DISC agent that's 60% Dominance and 30% Influence can now express that directly: `socialOrient: [(dominance, 0.6), (influence, 0.3)]`. The renderer communicates the nuance: "primarily independent, with collaborative tendencies."

**Cognitive function profiles.** Jungian functions don't fit on individual axes — Ti has implications across *all five*. So the descriptor carries a `dispositionProfile` field alongside the axes. The profile holds weighted function terms: `[(ti, 0.45), (ne, 0.20), (fi, 0.08), ...]`. At registration time, the system projects the function weights onto the axes automatically via the vocabulary's cross-vocabulary resolution.

**Cross-vocabulary projection.** Each `JungianFunctionTerm` implements `axisExactMatch` — the same mechanism DISC already uses. Ti maps to `independent` on social orientation, `principled` on rule following, `measured` on risk appetite, `autonomous` on autonomy, `avoiding` on conflict mode. The projection aggregates these across all eight functions, weighted by their profile weights, to produce multi-valued axes. An INTP's social orientation becomes `[(independent, 0.65), (collaborative, 0.31), (facilitative, 0.04)]` — a richer signal than a single label.

## Where This Goes Beyond JPAF

JPAF did the hard part — the research, the validation, the proof that cognitive function-level specification works. The open-source implementation at `agent-topia/evolving_personality` includes working weight tracking, all four reflection mechanisms, and LLM-adjudicated personality transitions. That's a complete system.

What eidos adds is the infrastructure layer — integrating JPAF's personality model into a structured agent identity framework with vocabulary grounding, persistence, and multi-format rendering:

**Vocabulary grounding.** The eight functions are a registered vocabulary (`urn:casehub:vocab:jungian`) with the full VocabularyTerm contract — value, label, description, aliases, axisExactMatch, specializes. The sixteen MBTI types are a second vocabulary (`urn:casehub:vocab:mbti`) where each type specializes its dominant and auxiliary functions. All of this composes with the existing vocabularies — DISC, Belbin, Conscientiousness, Thomas-Kilmann — through the same cross-vocabulary resolution layer.

**Personality health probing.** A new `DispositionHealth` SPI parallels the existing `CapabilityHealth`. The probe reads accumulated activation signals from a dedicated `DispositionSignalStore`, computes effective weights (base + accumulated changes), and checks JPAF's four threshold conditions — dominant-auxiliary swap, dominant replacement (shadow takeover), auxiliary replacement, and structural reorganisation. The result is a sealed type: `Aligned`, `Drifted`, or `EvolutionPending`.

**Personality evolution.** When the probe detects `EvolutionPending`, a separate `DispositionEvolution` SPI evaluates whether the transition should occur — using LLM-adjudicated judgment (matching JPAF's approach) with a rule-based fallback. Evolution creates a new descriptor version; the old one is preserved in the registry's history.

**Multi-format rendering.** The renderer detects the Jungian vocabulary and switches to function-level prompting — dominant and auxiliary functions described with their cognitive processing mode, compensation instructions for when the agent needs to draw on underused functions. MARKDOWN, PROSE, and A2A_CARD each get format-appropriate output.

## The Cross-Vocabulary Consistency Question

The question I needed to answer before building: does Jungian fit coherently with every other personality model eidos supports?

The answer is yes, cleanly:

| Combination | Rating |
|-------------|--------|
| Jungian + Belbin | Additive — cognitive style + team role are orthogonal |
| Jungian + DISC | Redundant — both describe behavioural style; Jungian is deeper |
| Jungian + Conscientiousness | Redundant — Jungian projects onto Conscientiousness axes |
| MBTI + Jungian | Hierarchical — MBTI emerges from Jungian function stacks |

The `dispositionProfile` field isn't Jungian-specific — we renamed it from `cognitiveProfile` during the design because DISC weighted profiles use the same mechanism. Any holistic disposition vocabulary that projects onto axes can use it. The field name reflects that generality.

## What's Built, What's Left

The API evolution is complete — `DispositionValue`, weighted `AgentDisposition`, the `dispositionProfile` field, all callers updated across ten modules. The Jungian and MBTI vocabularies are in with full cross-vocabulary resolution and structural rules (shadow functions, compatible auxiliaries, category/attitude metadata). The adversarial design review ran five rounds, raised 21 issues, resolved all of them — the spec is tight.

Eight implementation tasks remain: the signal store, health and evolution SPIs with implementations, auto-derivation, rendering, documentation, and the eval judges (MBTI alignment, function activation accuracy, personality shift accuracy). The vocabulary and API foundation is solid; the remaining work is runtime mechanics and validation.

What I find most interesting is the future direction this opens: contextual weights. The base weights are static declarations. But in practice, personality is situational — I know I'm introverted in some contexts and extraverted in others. The architecture already accommodates this: the `DispositionValue(term, weight)` primitive is the right structure for a runtime overlay that adjusts weights based on context. The signal store provides the observation layer; a future context-aware renderer could modulate weights before prompting. That's not in scope for this epic, but the foundation doesn't need to change.
