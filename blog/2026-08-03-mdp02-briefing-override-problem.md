---
layout: post
title: "Is Your Personality Framework Actually Doing Anything?"
date: 2026-08-03
entry_type: article
projects: [casehub-eidos]
tags: [personality, eval, jungian, experiment-design]
---

When you give an LLM agent a structured personality — weighted cognitive function scores, disposition axes, type profiles — you also write a briefing. A paragraph of natural language telling the agent how to behave.

Take the Hooded Claw from our Wacky Manor testbed. His descriptor carries an ENTJ Jungian profile — Te dominant at 0.35, Ni auxiliary at 0.20, six shadow functions filling the tail:

```yaml
disposition:
  mbtiType: ENTJ
  delegation: false
dispositionVocabulary: urn:casehub:vocab:jungian
```

The rendering pipeline projects those weights onto five disposition axes — rule-following, risk appetite, autonomy, social orientation, conflict mode — and writes them into the system prompt as structured personality data. That's the framework doing its job.

But the descriptor also carries a briefing:

```yaml
briefing: >-
  You are The Hooded Claw, Penelope Pitstop's secret nemesis, disguised
  as the mild-mannered estate manager Sylvester Sneekly. You have TWO
  voices and you MUST use them correctly:

  AS SNEEKLY (when other characters are present): You are obsequious,
  overly helpful, and unctuous...

  AS THE HOODED CLAW (when alone, or in private asides): You are
  grandiose, theatrical, and magnificently villainous...
```

That briefing text already implies the ENTJ pattern — strategic, organised, commanding. The LLM reads it directly. So when the Hooded Claw responds to a scenario with Te-style systematic planning, which signal drove it? The 0.35 weight on Te, or the prose describing "grandiose schemes explained step by step in dramatic monologues"?

This is the briefing override problem, and it matters more than it looks.

## Why it matters

If the briefing is doing all the work, the framework is overhead. Structured personality data becomes cosmetic — it shapes the prompt's appearance without changing agent behavior. That's a problem, because the entire value of structured personality is in what prose *cannot* do: vocabulary-grounded matching across agents, quantitative disposition comparison, cross-framework composition. If the framework doesn't independently drive behavior, none of those capabilities rest on real ground.

## The experiment

I built a 2x2 factorial to isolate the two signals. For each character, we construct four descriptor variants:

|  | Minimal briefing | Rich briefing |
|---|---|---|
| **Baseline** (no framework) | Floor — zero personality signal | Briefing-only contribution |
| **Jungian** (framework active) | Framework-only contribution | Current state |

The Hooded Claw in the *Jungian + Minimal* condition keeps his full ENTJ dispositionProfile — the Te/Ni weights, the vocabulary grounding, the derived axes — but his briefing is stripped to *"You are an agent named The Hooded Claw."* No theatrical villainy, no Sneekly voice, no schemes. If the framework works, Te-dominant behavior should still emerge from the structured weights alone.

In *Baseline + Rich*, it's the reverse: no dispositionProfile, no vocabulary, no derived axes — but the full briefing text survives. The LLM gets the prose description of the personality but none of the structured data. If the briefing is carrying the signal, this condition should score nearly as well as the full descriptor.

We run this across all eight Jungian profiles — INTP, ENTJ, INFP, ENFJ, ISTJ, ESTP, INTJ, ENTP — covering all eight dominant functions. For each profile and condition, six function activation scenarios target the dominant and auxiliary functions. The FunctionActivationJudge asks: given this system prompt and this scenario, which cognitive function did the agent actually use? TAA (Target Activation Accuracy) is the proportion it got right.

## What the results will tell us

The key comparison is *Jung+Min - Base+Min* — the framework's independent contribution, stripped of briefing support. If that delta is large, the structured weights genuinely reach behavior. If it's near zero, the framework needs stronger integration mechanisms — the weights are rendered into the prompt but the LLM isn't using them to shape its responses.

*Base+Rich - Base+Min* gives the briefing's independent contribution — what prose alone achieves without any framework. Comparing the two deltas tells you which channel carries more signal for the same personality.

The *Base+Min* condition establishes the floor. With nothing but a name and a slot, TAA should approach random (~0.125 for eight functions). Anything above that floor, in any condition, is signal from somewhere.

This experiment doesn't tell you *how much* personality framework is enough. It tells you whether the framework you built actually reaches the behavior you're trying to shape. For anyone building personality systems for LLM agents — Jungian, Big Five, DISC, or custom — that's the first question worth answering.
