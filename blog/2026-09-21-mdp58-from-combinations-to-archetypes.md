---
layout: post
title: "From 663,552 Personality Combinations to 48 Archetypes"
date: 2026-09-21
entry_type: article
subtype: diary
projects: [casehubio/eidos]
tags: [personality, archetypes, vocabulary, rendering, avatar, design]
---

# From 663,552 Personality Combinations to 48 Archetypes

Eidos has a rich vocabulary system — 85 personality terms across 11 frameworks. Belbin team roles, MBTI cognitive types, DISC behavioural styles, Big Five trait poles, Enneagram motivations, Jungian cognitive functions, SDI conflict styles. Each term carries a description with real semantic content. Each implements `axisExactMatch()` to translate itself into five canonical disposition axes.

The problem: that translation is lossy, and the loss happens before the two places it matters most.

## The fidelity gap

Every personality framework maps to the same five canonical axes: social orientation, rule following, risk appetite, autonomy, and conflict mode. A Belbin Plant and a DISC Dominance both resolve to `{independent, flexible, bold, autonomous}` — differing only on conflict mode (avoiding vs competing). The rendered prompt says:

```yaml
## How You Operate
- Social orientation: Independent
- Rule adherence: Flexible
- Risk appetite: Bold
- Autonomy level: Autonomous
- Conflict approach: Avoiding
```

That's it. No "Belbin." No "Plant." No "Creative, unorthodox problem-solver; generates novel ideas independently." The 85-term vocabulary with its cross-mapping matrices collapses to 17 coarse values before reaching the LLM.

The Jungian path is the exception. When the vocabulary is `urn:casehub:vocab:jungian`, the renderer produces a full Cognitive Style section — dominant and auxiliary function labels, descriptions, orientation guidance, response style, anti-pattern warnings. An INTJ gets two paragraphs of rich characterisation. A Belbin Plant gets five axis labels.

This was a design gap, not a bug. The Jungian rendering was built as a proof of concept. It was never generalised.

## Why this matters for LLMs

An LLM has deep training data on what a "Belbin Plant" is. Telling it `riskAppetite: bold` activates almost nothing. Telling it "You are a Plant — creative, unorthodox problem-solver who generates novel ideas independently" activates a rich behavioural pattern the model already understands.

The same applies to avatars. A visual identity system receiving `{independent, flexible, bold, autonomous}` can generate a face with certain mouth curves and eye openness values. But it can't distinguish between a Plant and a Dominance — two fundamentally different archetypes — because the distinguishing information was discarded upstream.

## The combinatorial problem

The vocabulary system has six personality frameworks an agent can be characterised through:

| Framework | Terms |
|---|---|
| MBTI / Jungian | 16 types |
| Enneagram | 9 types |
| DISC | 4 styles |
| Big Five | 32 (2^5 pole combinations) |
| Belbin | 9 roles |
| SDI | 4 styles |

The raw cross-product is 9 × 16 × 4 × 32 × 9 × 4 = **663,552** combinations. Most are noise — personality research shows strong correlations across frameworks. An INTJ clusters with Enneagram 5, DISC C, high Openness, low Agreeableness. The practical archetype space is much smaller.

But the data model constrained this further. `dispositionVocabulary` is a single URI — one framework per agent. An agent is Jungian OR DISC, not both. The combination that produces the richest characterisation (INTJ + Enneagram 5 + Belbin Monitor Evaluator) wasn't expressible.

## Archetypes as the abstraction layer

The insight: these frameworks are complementary, not competing. MBTI measures how you process information. DISC measures how you interact. Enneagram measures what drives you. Big Five measures broad traits. They're orthogonal dimensions of the same underlying personality.

The Hartwell & Chen archetype model (from *Archetypes in Branding*, 2012) gives us the abstraction layer that was missing. Twelve families, each with four sub-archetypes, grouped by four motivation quadrants:

| Quadrant | Families |
|---|---|
| Independence | Innocent, Explorer, Sage |
| Mastery | Hero, Rebel, Magician |
| Belonging | Everyman, Lover, Jester |
| Stability | Caregiver, Creator, Sovereign |

Each sub-archetype is a universally understood personality label: Detective, Pioneer, Warrior, Alchemist, Maverick, Storyteller. LLMs know what these are. Humans know what these are. Avatar systems can generate distinctive visual identities from them.

The archetype sits between the framework combination and the canonical axes:

```
Framework values (INTJ + Enneagram 5 + DISC C)
    ↓ set intersection
Archetype (Detective)
    ↓ derivation
Canonical axes (independent, strict, cautious, autonomous, avoiding)
```

## The Venn diagram algorithm

Each framework value maps to a set of compatible archetype families. Specifying values across multiple frameworks intersects the sets. What survives is the archetype.

```
compatible("INTJ")       = {Sage, Magician, Sovereign}
compatible("Enneagram 5") = {Sage, Explorer, Magician}
compatible("DISC C")      = {Sage, Sovereign, Creator, Explorer}

INTJ ∩ E5              = {Sage, Magician}
INTJ ∩ E5 ∩ DISC-C     = {Sage}
```

Three framework values, one family. Sub-archetype affinity (Detective has INTJ + Enneagram 5 affinity) narrows further within the family. The whole thing is a `retainAll()` call on precomputed sets.

The algorithm handles every entry point. Start from MBTI — narrows to 3-4 families. Start from an archetype — shows compatible framework values. Start from Belbin — narrows archetypes AND compatible MBTI types. Every selection is commutative. Conflicts are detectable: if the intersection is empty, the specified values are contradictory.

## What the prompt looks like now

With the archetype system, the same agent descriptor produces:

```markdown
## Personality
**Detective** (Sage family) — meticulous, persistent

Uncovers truth through systematic investigation and evidence.

## How You Operate
- Social orientation: Independent
- Rule adherence: Strict
- Risk appetite: Cautious
- Autonomy: Autonomous
- Conflict approach: Avoiding
```

"Detective — meticulous, persistent" carries more behavioural signal than five axis labels ever could. The LLM activates its training on detective archetypes: investigative, evidence-driven, methodical, sceptical. The canonical axes still exist for operational matching — `AgentRegistry.find()` queries against them. But the generative output uses the richer representation.

For avatars, the archetype provides immediate visual identity. A Detective has a scholarly, scrutinising quality. A Pioneer has an adventurous, forward-leaning quality. A Warrior has determination and strategic intensity. These are visual concepts that a generic `{bold, autonomous}` decomposition can't express.

## Adjectives as the refinement layer

The archetype constrains what's reachable. A Detective can be meticulous, analytical, persistent, methodical, observant, evidence-driven, sceptical. It cannot be impulsive, gullible, reckless, or superficial — those belong to other archetypes. The adjectives refine within the archetype's valid space.

This replaces per-framework configuration with a two-step selection: pick an archetype, add 1-2 adjectives. The framework values are derived, not specified. A user who thinks in MBTI terms can still start there — the Venn diagram narrows to the archetype. But the archetype is the concept that flows through to the prompt and the avatar.

The vocabulary system's 85 terms and cross-mapping matrices aren't wasted. They're the foundation the archetype layer sits on — the compatibility tables that power the set intersection. Every `axisExactMatch()` implementation feeds the derivation pipeline. The work done building eleven vocabulary enums with bidirectional cross-references is what makes the archetype resolution accurate.

The difference: a skilled developer adding a new personality framework no longer needs to worry about whether the renderer surfaces their descriptions. The archetype carries the semantic signal. The framework values are the plumbing.
