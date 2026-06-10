# 0005 — Behavioral Evaluation Methodology: Pair-Contrast over Per-Profile Scoring

Date: 2026-06-10
Status: Accepted

## Context and Problem Statement

Phase 3 of the eval harness must test whether eidos-rendered system prompts actually
change agent behaviour — not just whether the rendered text looks good. The original
design used per-profile scoring: a judge receives the expected trait values (HIGH/LOW)
and rates how well an agent's responses match the expected profile. This was rejected
as circular.

## Decision Drivers

* The judge must not know the expected answer before reading agent responses — otherwise
  evaluation proves nothing about behavioural alignment
* The method must control for question bias (both agents answer the same question)
* The method should reuse existing infrastructure (VariantPair, PairContrastJudge pattern)

## Considered Options

* **Option A — Per-profile loaded scoring** — judge sees expected axis values and scores
  each agent's responses against them
* **Option B — Pair-contrast blind comparison** — judge sees both agents' responses to
  the same question and picks which expresses the axis more strongly, without knowing
  which profile is "higher"
* **Option C — Cross-model judging** — one model answers, a different model judges,
  avoiding same-model circularity while keeping loaded scoring

## Decision Outcome

Chosen option: **Option B (pair-contrast blind)**, because it eliminates both circularity
(judge has no expected values to anchor on) and question bias (both agents answer the
same question), and directly extends the Phase 2 PairContrastJudge pattern.

### Positive Consequences

* Judge is forced to detect behavioural signal from the text, not from expectations
* Controls for question framing — the question is held constant
* Reuses VariantPair, PairContrastJudge infrastructure

### Negative Consequences / Tradeoffs

* Requires variant pairs — profiles without a pair cannot be evaluated
* Position bias risk: A=higher, B=lower always; judge may favour "A" systematically
* 6 data points (2 pairs × 3 questions) is a sanity check, not a statistically meaningful threshold

## Pros and Cons of the Options

### Option A — Per-profile loaded scoring

* ✅ Works for any profile, not just paired ones
* ❌ Circular — judge knows expected values and scores against them
* ❌ Cannot distinguish "prompt works" from "any response on this topic sounds like the expected profile"

### Option B — Pair-contrast blind comparison

* ✅ Methodologically sound — judge cannot see expected values
* ✅ Controls for question bias
* ✅ Reuses existing variant pair infrastructure
* ❌ Requires paired profiles
* ❌ Position bias unmitigated (planned Phase 4 fix: randomise A/B assignment)

### Option C — Cross-model judging

* ✅ Removes same-model circularity even with loaded scoring
* ❌ Requires a second LLM provider
* ❌ Does not control for question bias
* ❌ The loaded-scoring circularity problem is structural, not model-specific

## Links

* [eval design spec](../superpowers/specs/2026-06-09-eval-baseline-behavioral-design.md)
* Phase 2 PairContrastJudge — `eval/src/main/java/io/casehub/eidos/eval/PairContrastJudge.java`
* Protocol PP-20260610-de090d — behavioral-judge-blind
