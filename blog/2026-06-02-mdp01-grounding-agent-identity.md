---
layout: post
title: "Grounding agent identity in the real world"
date: 2026-06-02
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-eidos]
tags: [eval, profiles, llm-evaluation, personality, attribution]
---

The existing eval dataset was entirely synthetic — devtown planners, ML researchers,
code reviewers built from scratch to stress-test the renderer. They did the structural
job fine. What they couldn't answer was whether the pipeline could take a real
human-authored system prompt, derive a descriptor from it, and render something that
still reads like the same agent. That's eidos#23.

The work split into two parts that ended up more tightly coupled than I expected: a
profile library of real-world agent definitions, and a measurement system capable of
telling you when and why personality is lost in the pipeline.

**Sealing EvalCase**

The first structural decision was converting `EvalCase` from a record to a sealed
interface. The original record had one form — now there are two: `SyntheticEvalCase`
(hand-crafted, no source) and `ProfiledEvalCase` (wraps a real profile with original
prose). I could have added `Optional<AgentProfile>` to the existing record, but
Optional in record components is a smell. It says "this type has two modes" — and when
that's true, a sealed interface says it more clearly, eliminates the `isPresent()` guards
everywhere, and lets the new judges take `ProfiledEvalCase` directly.

**Why ProximityJudge is not a new EvalDimension**

`EvalDimension.applicableFor(format)` is simple: format in, dimension set out. Semantic
proximity — how faithfully does the rendered prompt capture the original prose? — would
have added a second axis: profile availability. That breaks the invariant. Profile cases
and synthetic cases would produce `EvalResult.overall` scores computed over different
denominators, making cross-dataset comparisons meaningless.

ProximityJudge is a separate bean with its own result type and a separate floor. It runs
after the standard quality eval, on the same renders. The quality signal stays comparable;
proximity is additive.

**Three stages of personality preservation**

I wanted to be able to say more than "the pipeline is working" or "it isn't." The
three-stage system attributes failures to a specific locus.

Stage 1 asks: can a short open-string label capture what the prose says about this
personality axis? Four sequential LLM calls, one per axis. If the vocabulary can't
express it, fixing the renderer won't help — the gap is upstream.

Stage 2 blind-scores the rendered text. The judge sees only `rendered.content()`. No
descriptor, no profile. This was the constraint I most wanted to get right: if the judge
sees `riskAppetite: "conservative"` in the descriptor, it will score that axis high
regardless of whether the rendered text actually expresses conservatism. The test captures
the user message payload and asserts it doesn't contain `agentId` — that invariant is now
compile-tracked.

Stage 3 measures pairwise effect size between variant profiles. The profiles come in pairs
that differ on exactly one disposition axis: the sw-engineer pair on `riskAppetite`, the
security-analyst pair on `ruleFollowing`. Stage 3 answers "how distinctly different do they
read?" — a high effect size on the declared axis is the signal the pipeline is preserving
personality differentiation.

The attribution logic computes a diagnosis per (profile, axis): vocabulary gap, renderer
flattening, profile design gap, or working. All three stages must pass for working — and
critically, Stage 1 must score high (≥ 4) for any diagnosis beyond vocabulary gap to be
meaningful.

That last constraint I got wrong in the first implementation. The `PROFILE_DESIGN_GAP` and
`WORKING` branches were firing without the `s1 >= 4` guard. A profile with borderline
vocabulary (score 3, "approximates but loses nuance") and a high match rate would diagnose
as working. Claude's final code review caught both missing guards. We added
`PersonalityPreservationReportTest` to cover the boundary cases — those tests now fail if
the guards are removed.

**The profiles themselves**

Eight profiles grounded in Belbin team roles and DISC behavioural profiles. The variant
pairs satisfy Stage 0 validation — a deterministic check that runs at load time and throws
if any non-primary axis differs between pair members. The careful and bold engineers are
identical on `socialOrient`, `ruleFollowing`, `autonomy`, and `delegation`. Only
`riskAppetite` — `conservative` vs. `bold` — changes.

Most profiles came from O*NET occupational data. The Anthropic Prompt Library had better
software engineering prose, but for clinical researcher, customer support, and technical
writer, O*NET provided structured capability data that made the descriptor derivation
principled. The trade-off is that O*NET-synthesised prose isn't a real published system
prompt — proximity scores for those profiles carry lower confidence, which is why
`SourceType.ONET_SYNTHESISED` is a distinct enum value.

**81 tests, no results yet**

The 81 tests are unit tests using mock models. They verify the infrastructure: parsing,
attribution logic, Stage 0 validation, payload isolation. The integration test that would
produce actual scores — `evaluateRealWorldScenarios()` — requires a configured judge model
and is excluded from CI. We've set the proximity floor at 3.0 and left the personality
preservation thresholds for after the first real run.

Whether the variant pairs produce distinguishable effect sizes — whether a bold engineer
and a careful one actually read differently enough for the judge to tell them apart — is
the question the system was built to answer. We'll find out when the credentials are
configured.

The next step from here is whether the vocabulary gaps surfaced in the profiles feed
anything. They should — that's what eidos#26 is for, once the framework mapping document
(eidos#29) exists to tell us how Belbin, DISC, Big Five, Thomas-Kilmann, and Situational
Leadership actually map to each other and to `AgentDisposition`.
