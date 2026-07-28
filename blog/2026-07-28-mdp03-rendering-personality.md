---
layout: post
title: "Rendering Personality"
date: 2026-07-28
type: phase-update
entry_type: note
subtype: diary
projects: [eidos]
tags: [personality, jungian, rendering, a2a, cognitive-profile, llm-agents]
series: issue-107-jungian-personality-framework
---

*Part of a series on [#107 — Jungian personality framework](https://github.com/casehubio/eidos/issues/107). Previous: [Wiring the Disposition Layer](2026-07-28-mdp02-wiring-the-disposition-layer.md).*

## Why Agent Personality Needs Rendering

Most LLM agent frameworks treat personality as a paragraph in a system prompt — "You are a helpful, analytical assistant who values precision." Prose. Handwritten. Static. The moment you need five agents in a team, each with a distinct cognitive style, you're copy-pasting paragraphs and hoping the model pays attention to the differences.

Eidos treats personality as structured data. An agent's disposition is a set of weighted values across defined axes — social orientation, rule following, risk appetite, autonomy, conflict mode — grounded in registered vocabularies. That structure is machine-readable, queryable, comparable, and evolvable. But structured data doesn't go in a system prompt. A model needs prose. Something has to bridge the gap.

That bridge is the rendering pipeline: take the structured disposition, resolve terms through vocabularies, and produce format-specific output. MARKDOWN for system prompts. PROSE for LLM enrichment. A2A_CARD for machine-to-machine agent discovery. Same data, three views, each optimised for its consumer.

## Jungian Functions: The Research That Changed the Approach

The [previous entry](2026-07-28-mdp01-personality-from-cognitive-functions.md) covers the research in detail, but the short version: JPAF (arXiv:2601.10025) demonstrated that specifying personality through Jungian cognitive functions — not trait labels, not MBTI types — achieves 100% MBTI alignment in LLM responses. Every other approach they tested fell short.

The eight Jungian functions (Ti, Te, Fi, Fe, Si, Se, Ni, Ne) are the building blocks. Each function describes a mode of cognition — Introverted Thinking builds internal logical frameworks, Extraverted Intuition explores external patterns and possibilities. Every person (and now every agent) uses all eight, but in a specific priority order. An INTP leads with Ti (dominant) and Ne (auxiliary). An ENFJ leads with Fe and Ni. The dominant function shapes how the agent approaches problems. The auxiliary complements it.

This matters for multi-agent systems because cognitive style determines collaboration dynamics. An INTP code reviewer (Ti-dominant: analytical, internally consistent, precision-focused) behaves differently from an ENFJ team coordinator (Fe-dominant: group harmony, interpersonal awareness). Not just in what they say — in how they reason about problems, what they notice, what they deprioritise.

## Weighted Axes: Personality Isn't Binary

Axes used to be single strings: `socialOrient: "independent"`. Now they carry weighted lists: `[("independent", 0.7), ("collaborative", 0.3)]`. An agent can be primarily independent but with collaborative tendencies — which is exactly what an INTP's cognitive profile produces when projected onto the social orientation axis.

The rendering has to make this readable. For single values at weight 1.0, nothing changes — `Social orientation: independent`. For two weighted values: `Social orientation: primarily independent (0.7), with collaborative tendencies (0.3)`. Three or more list by weight without the "tendencies" phrasing. Vocabulary resolution applies throughout — registered vocab labels replace raw terms.

This isn't cosmetic. The weight distribution IS the personality signal. An LLM receiving "primarily independent with collaborative tendencies" behaves measurably differently from one told "independent." The weights create a gradient the model can work with — not a binary categorisation but a spectrum that the model navigates per-response.

## Cognitive Style: Function-Level Prompting

When an agent has a Jungian disposition profile, the renderer produces a Cognitive Style section before the disposition axes:

```markdown
## Cognitive Style

Your personality is structured around Jungian cognitive functions:

**Dominant — Introverted Thinking (Ti):** Builds internal logical
frameworks; analytical, precision-focused, seeking internal consistency.
This is your primary mode of engagement.

**Auxiliary — Extraverted Intuition (Ne):** Explores external patterns,
possibilities, and connections. This complements your analytical core.

When your dominant and auxiliary functions cannot effectively address
a situation, draw on other cognitive functions. Recognize that
compensatory function use produces less controlled but potentially
valuable responses.
```

The structure mirrors JPAF's prompt design: identify the dominant and auxiliary functions explicitly, describe what they do, then add compensation instructions. The compensation paragraph matters — it gives the model permission to use non-dominant functions when the situation demands it, while signalling that this should feel like a stretch. The result is an agent that can handle situations outside its comfort zone without losing its primary cognitive style.

The dominant and auxiliary are identified purely by weight ordering — highest weight is dominant, second is auxiliary. No metadata stored, no explicit designation. This means if the weights shift through evolution (disposition health detects reinforcement patterns and adjusts), the rendering automatically reflects the new cognitive structure.

## The Renderer Can't Import the Vocabulary

An interesting boundary constraint: `JungianFunctionTerm` lives in `casehub-eidos-vocab`, and the runtime module only has vocab as a test dependency. This keeps vocabularies optional — consumers who don't need Jungian personality don't pull in the vocab module.

The renderer uses the same pattern as auto-derivation and disposition health — resolve through the vocabulary registry, never import vocab types. A URI string constant (`"urn:casehub:vocab:jungian"`) drives template dispatch. The dominant function's first letter determines the complement text: `t` → "analytical core", `f` → "values-driven core", `s` → "experiential core", `n` → "intuitive core." A string convention instead of an enum import.

## A2A Cards: Machine-Readable Personality

The A2A_CARD is how agents discover each other. The disposition section evolved from single-value objects to weighted arrays:

```json
"dispositionProfile": {
  "vocabulary": "urn:casehub:vocab:jungian",
  "functions": [
    {"term": "ti", "weight": 0.35, "role": "dominant"},
    {"term": "ne", "weight": 0.20, "role": "auxiliary"}
  ],
  "derivedMbtiType": "INTP"
}
```

The `role` and `derivedMbtiType` are computed at render time, not stored. `derivedMbtiType` matches the profile's dominant+auxiliary against `MbtiTypeTerm.specializes()` through the registry. A different weight distribution after evolution would produce a different type without touching the descriptor's identity.

This is the machinery that makes personality-aware dispatch possible. An orchestrating engine can read the A2A card, see that Agent A is Ti-dominant (analytical, precision-focused) and Agent B is Fe-dominant (group harmony, interpersonal), and route a code review to A and a stakeholder debrief to B — not because someone hardcoded a routing table, but because the personality structure is machine-readable at the protocol level.

## What's Still Coming

The rendering pipeline is complete for all three formats, but two pieces remain before the framework is end-to-end. `DefaultDispositionEvolution` handles what happens when the probe detects that an agent's personality has drifted far enough to warrant a structural change — the four JPAF reflection types (dominant-auxiliary swap, dominant replacement, auxiliary replacement, structural reorganization). And the eval judges will validate that rendered prompts actually produce personality-aligned responses using MBTI questionnaire items and function activation scenarios. The rendering proved out in this session is what those judges will be scoring.
