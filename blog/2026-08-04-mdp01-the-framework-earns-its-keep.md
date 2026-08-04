---
layout: post
title: "The Framework Earns Its Keep"
date: 2026-08-04
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-eidos]
tags: [personality, eval, jungian, experiment-results, vertex-ai, cross-model-validation]
series: issue-136-eval-reactive-calibration
---

*Part of a series on [#136 — Run minimal briefing experiment](https://github.com/casehubio/eidos/issues/136). Previous: [Is Your Personality Framework Actually Doing Anything?](2026-08-03-mdp02-briefing-override-problem.md).*

Yesterday I laid out the briefing override problem: when a structured personality framework and a prose briefing both shape an LLM agent's behaviour, which one is doing the work? The experiment design was a 2×2 factorial — framework on/off crossed with briefing minimal/rich — across eight Jungian profiles.

Today I ran it. Four times, across two models, with a split-model validation pass. The answer is clear enough to be useful: the framework contributes independently, and it contributes more when the model is smaller.

## Fixing the instrument first

The experiment almost didn't produce anything. The first run came back with all zeros — every scenario scored TAA 0.00 across all eight profiles. The FunctionActivationJudge was being injected by CDI, but the underlying `AgentProvider` bean wasn't discoverable because the JAR lacked a `beans.xml` and Quarkus doesn't scan test-scoped dependencies without explicit `quarkus.index-dependency` configuration. Every LLM call was silently failing, caught by the exception handler, and recorded as `activatedFunction: "error"`.

That was the easy fix. The harder one: even after the infrastructure worked, the judge had a systematic blind spot. Introverted Intuition (Ni) — the convergent insight function — was being classified as Extraverted Intuition (Ne) 100% of the time. Both involve pattern recognition; the judge prompt's one-line descriptions didn't give the LLM enough to distinguish convergent synthesis ("the answer is X") from divergent exploration ("the options are A, B, C").

I rewrote the judge prompt with explicit contrastive discriminators — Ni CONVERGES, Ne DIVERGES — and replaced the Ni scenarios with prompts that structurally force singular answers: "What is the ONE underlying cause?" instead of "What deeper pattern connects these?" The fix took Ni accuracy from 0% to 72%. The remaining 28% get misclassified as Te, not Ne — different error, and one that tells us something about how dominant Te in profiles like ENTJ drowns out auxiliary Ni signals.

## The numbers

Four experiment runs, 192 LLM-evaluated scenarios each:

| | Sonnet/Sonnet R1 | Sonnet/Sonnet R2 | Haiku/Haiku | Haiku/Sonnet |
|---|---|---|---|---|
| Baseline + Minimal | 0.67 | 0.63 | 0.63 | 0.56 |
| Baseline + Rich | 0.75 | 0.81 | 0.73 | 0.67 |
| Jungian + Minimal | 0.73 | 0.71 | 0.73 | 0.65 |
| Jungian + Rich | 0.71 | 0.77 | 0.71 | 0.67 |
| **Framework Δ** | **+0.06** | **+0.08** | **+0.10** | **+0.08** |
| **Briefing Δ** | **+0.08** | **+0.19** | **+0.10** | **+0.10** |

Framework contribution is positive across every run. Modest — 6 to 10 percentage points — but consistent. The structured Jungian disposition axes, rendered into the system prompt with no personality briefing at all, shift agent behaviour toward the specified cognitive function profile.

The baseline isn't random. Even Baseline+Minimal scores 0.56–0.67, not the theoretical 0.125 (1-in-8 chance). The agent's role and slot alone carry some signal — a "Systems Analyst" agent responds analytically regardless of whether you specify Ti-dominance. The framework's contribution sits on top of that already-strong baseline.

## Smaller models benefit more

The finding I didn't expect: the framework contribution is larger with Haiku (+0.10) than with Sonnet (+0.06/+0.08).

The explanation fits once you see it. A larger model can compensate for missing structure through stronger reasoning — Sonnet extracts personality signal from a minimal briefing because it has the capacity to infer what's implied. Haiku doesn't. It needs the explicit structured guidance, and the framework provides exactly that.

This matters for cost-sensitive deployments. If you're running agents on cheaper models to manage API costs, structured personality specification isn't a luxury — it's the mechanism that keeps personality differentiation working when the model can't do it on its own.

## The judge was grading its own homework

When I ran the Haiku experiment, the framework contribution jumped to +0.10 — higher than Sonnet. But both the agent and the judge were Haiku. A weaker model generating responses and then evaluating them creates a self-evaluation bias: the judge might recognise patterns from its own generation style, or it might see "Ti" in the system prompt and rubber-stamp the classification.

To test this, we built split-model support — Haiku generates the agent responses, Sonnet judges them. The framework contribution with Sonnet judging: +0.08. It survived. The delta held up under a stricter, independent evaluator.

The absolute scores dropped — Sonnet-as-judge rates Haiku's responses more harshly (0.56 baseline vs 0.63). That's expected. But the framework delta staying at +0.08 means the signal is real agent behaviour, not judge self-recognition.

## What the profiles tell us

Not all personality types are equally detectable. The per-profile data reveals a clear hierarchy:

**INTJ and ENTJ** score 0.83 across nearly every condition and every run. Their dominant functions (Ni and Te) are distinctive enough that even a baseline rendering captures them. The framework doesn't help much because the personality is already strongly expressed — there's not much room to improve.

**INTP** is where the framework shows its largest lift — 0.50 baseline to 0.83 with framework, a +0.33 jump in one run. Ti (Introverted Thinking) is the analytical function that looks superficially like Te. Without the framework's explicit specification, the agent defaults to Te-style systematic planning. With it, the response shifts toward Ti's internal logical consistency — building frameworks from first principles rather than organising external outcomes.

**ESTP** is the hardest type to capture. Se (Extraverted Sensation) — the present-moment, concrete-action function — scores 0.17 to 0.50 across all conditions. In text, "react to what's in front of you right now" looks a lot like "execute a systematic plan." The judge can't reliably tell them apart, which means we can't tell whether the framework is failing or the measurement is.

## What this doesn't tell us

The experiment measures whether the framework changes behaviour. It doesn't measure whether the changed behaviour is *good* — whether an INTP agent's Ti-shifted responses are actually more useful for the tasks an INTP should excel at. TAA measures accuracy against a Jungian classification target, not downstream task performance.

The judge is also a bottleneck. An 8-way open classifier is the hardest possible task for an LLM judge — every other judge in the eval module uses pairwise comparison or anchored scoring. The TAA ceiling is constrained by judge accuracy. Ti/Te confusion persists across all runs, and Se is effectively unmeasurable in text. Moving to pairwise judging for confusable pairs would sharpen the measurement substantially.

And the variance is real. Briefing contribution ranges from +0.08 to +0.19 across two identical Sonnet runs. That's a wide band for what should be the same experiment. More repetitions would tighten the confidence intervals, but the direction is consistent.

## Where this leads

The framework earns its keep. Not dramatically — this isn't a "10x improvement" result. It's a consistent, measurable, independently-validated contribution to personality-differentiated agent behaviour. The structured axes add something that prose alone doesn't, and they add more when the model has less reasoning capacity to compensate.

The next question is whether pairwise judging — "is this response more Ti or Te?" — can sharpen the measurement enough to see what the current classifier misses. The framework may already be producing correct behaviour that the judge can't detect. The interesting work now is building an instrument that can tell the difference.
