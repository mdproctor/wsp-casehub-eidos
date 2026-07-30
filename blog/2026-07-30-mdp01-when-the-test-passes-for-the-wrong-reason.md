---
layout: post
title: "When the Test Passes for the Wrong Reason"
date: 2026-07-30
type: phase-update
entry_type: note
subtype: diary
projects: [eidos]
tags: [personality, jungian, eval, mbti, testing, llm-agents]
series: issue-122-vocab-imbue-verify
---

## The Test Suite That Found a Bug in Itself

The eval judges from the last few entries — MbtiAlignmentJudge, FunctionActivationJudge, PersonalityEvolutionJudge — were designed to score whether a rendered agent prompt actually produces personality-aligned behaviour. The obvious next step: build a test suite that proves each vocabulary framework (Jungian, Belbin, DISC, Conscientiousness) works independently when an LLM is imbued with it, and that additive pairs compose without contradiction.

The test suite has two layers. The structural layer runs in CI without an LLM — it builds descriptors for each vocabulary, runs them through axis derivation and prompt rendering, and asserts deterministic outcomes. Profile weights, axis values, prompt content. This layer caught something immediately: the `composite()` fixture was silently dropping derived axes when merging two descriptors, because it rebuilt the disposition from only the profile weights. A regression that would never surface in an LLM eval — you'd just see a weaker prompt, not a clear failure.

The structural tests also verified the compatibility table empirically. Jungian + Belbin: both signals survive composition (Jungian axes derived, Belbin slot label present). Jungian + DISC: conflicting axis values (ENTJ derives `ruleFollowing=strict`, DISC Dominance derives `ruleFollowing=flexible`). The table was a design assertion until now. It's a tested fact with specific conflicts identified.

## Four Dimensions, Two Problems

The eval layer uses a live LLM to score rendered prompts. I ran an ENTJ descriptor through the MbtiAlignmentJudge — 12 personality questions across four MBTI dimensions. TF and JP scored 1.00 immediately. EI scored 0.00. SN scored 0.33.

EI was a genuine rendering gap. The Jungian function stack encodes E/I through the dominant function's attitude — Te is *extraverted* thinking. But the cross-vocabulary projection maps Te to `socialOrient=independent` and `autonomy=semi-autonomous`. Those axis values are behaviourally correct (Te IS systematic and self-directed), but they read as introverted to the MBTI questionnaire. The 5 Conscientiousness axes don't have an energy-direction dimension, so E/I information was being lost in the projection.

The fix: derive the dominant function's attitude directly from the function stack and add an explicit orientation paragraph to the Cognitive Style section. For extraverted-dominant types: "Your natural orientation is outward. You think out loud, actively seek input from others..." For introverted-dominant: "Your natural orientation is inward. You prefer to work through problems internally..." EI went from 0.00 to 0.67–1.00 across runs.

## The Dimension That Wouldn't Move

SN stayed at 0.33 — one out of three questions correct, deterministic across every run. I went through five iterations of rendering changes: perception hints as prose paragraphs, as structured bold lines, in the enrichment payload so the narrative LLM would weave them in. The enrichment LLM *did* incorporate the language — the rendered prompt explicitly said "intuitive perception — you gravitate toward underlying patterns, big-picture implications, and novel possibilities." But the score wouldn't move.

Each iteration made the prompt more explicitly N-type. The rendering was working. The perception hints were landing in the right section with the right structural weight. And the score stayed frozen at 0.33.

I handed the full context to a fresh session — the rendered prompt, all 12 questionnaire questions, the scoring logic, the axis mappings, the five iterations — and asked it to re-architect the solution from first principles.

Claude came back in minutes with a two-character fix in MbtiAlignmentJudge.java.

## The Answer Key Was Wrong

The MBTI questionnaire's `aIsPole` field declares which MBTI pole option A represents. Two of the three SN questions had it inverted. Q4 asks "concrete, established methods (A) or abstract patterns and novel possibilities (B)?" — option A is S-type behaviour. But `aIsPole` was set to `"N"`. The scoring logic expected a correct N-type answer to be A — the opposite of what the question actually tests.

Q6 had the same inversion: "proven approaches (A) or innovative solutions (B)?" with `aIsPole="N"`.

A judge that correctly read the ENTJ as intuitive — answering B on both questions — was penalised twice. Score: 1/3. A judge that incorrectly read S-type would score 2/3. The harder I worked to strengthen the N-type signal, the worse the score got, because each improvement helped the judge answer *correctly* — and correct answers were being marked wrong.

Two characters changed: `"N"` to `"S"` on Q4 and Q6. SN went from 0.33 to 1.00. All four MBTI dimensions now score 1.00 on both standalone ENTJ and composite ENTJ+Shaper tests.

## What the Five Iterations Actually Produced

The rendering pipeline improvements — the E/I orientation hint and the S/N perception hint — turned out to be independently necessary. I traced the scores through the inverted key to recover what the correctly-keyed scores would have been at each stage:

| Stage | EI (correct key) | SN (correct key) |
|---|---|---|
| Baseline (no hints) | 0.00 | 0.00 |
| E/I orientation hint | 0.67–1.00 | 0.00 |
| E/I + S/N perception hint | 0.67–1.00 | 1.00 |

Without the perception hint, the baseline prompt scores SN=0.00 even with the correct answer key — the axis values (`strict`, `measured`) dominate the prompt and the judge reads S-type. The hint is what flips it. Five iterations of prompt engineering, undertaken for the wrong reason, produced two genuine architectural improvements to the rendering pipeline.

## What This Means for Structured Personality

The Conscientiousness disposition axes map cleanly to two MBTI dimensions — TF and JP score 1.00 with no help. They fail on the other two — EI and SN — because those dimensions describe aspects of personality that don't have corresponding axes. E/I is energy direction (external vs internal interaction). S/N is perception style (concrete vs abstract). Neither maps to socialOrient, ruleFollowing, riskAppetite, autonomy, or conflictMode.

The fix is architectural: when a Jungian function stack is present, derive E/I and S/N from the stack directly and render them as explicit hints alongside the axes. The function stack already contains this information — the dominant function's attitude gives E/I, the perceiving function in the dom/aux pair gives S/N. The rendering pipeline was discarding two dimensions worth of personality data because the intermediary vocabulary (Conscientiousness) couldn't represent them.

The broader pattern: cross-vocabulary projection is lossy by design. Any target vocabulary that has fewer dimensions than the source loses information in the mapping. Conscientiousness has 5 axes. MBTI has 4 dimensions. The two systems overlap on 2 dimensions (TF, JP) and diverge on 2 (EI, SN). Knowing where the loss happens — and compensating for it at the rendering layer rather than trying to force more information through the projection — is how you build a pipeline that produces correct prompts from structured personality data.

The test suite that discovered all of this is now a permanent fixture. Sixteen structural tests run in CI. Eleven LLM eval tests verify behavioural alignment under `-Peval`. A new DispositionPresenceJudge handles non-Jungian vocabularies — Belbin Shaper scores 1.00, DISC Dominance scores 1.00, both survive pairwise composition. The compatibility table between vocabulary pairs is empirically verified, not assumed.
