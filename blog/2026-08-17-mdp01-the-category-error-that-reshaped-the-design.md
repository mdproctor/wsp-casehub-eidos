---
layout: post
title: "The Category Error That Reshaped the Design"
date: 2026-08-17
entry_type: note
subtype: diary
projects: [eidos]
tags: [vocabulary, disposition, sarcasm, sarc7, design-review, personality]
---

We set out to add structured sarcasm types to the Eidos disposition model. The Sarc7 paper (Xiong et al., ACL WiNLP 2025) defines seven pragmatically distinct sarcasm types — self-deprecating, brooding, deadpan, polite, obnoxious, raging, manic — each with four evaluation dimensions: incongruity, shock value, context dependency, emotional tone. The question was where to put them.

The obvious answer was wrong.

DISC works as a disposition vocabulary because DISC types *are* behavioral claims. "Dominance" genuinely implies independence on social orientation, boldness on risk appetite, autonomy on the autonomy axis. The mappings have decades of psychometric research behind them — they correlate with Big Five Extraversion × Agreeableness. So the natural first design was: treat Sarc7 the same way. Create a `Sarc7Term` vocabulary, implement `axisExactMatch`, store sarcasm types across all five disposition axes.

But "deadpan" doesn't determine risk appetite. A deadpan communicator can be conservative or bold, strict or flexible, directed or autonomous. The cross-mappings were editorial associations — "deadpan feels like it maps to INDEPENDENT on social orientation" — not behavioral claims. Storing "deadpan" across five axes was semantically hollow. The spec's own rationale said "humor is a personality framework, not a behavioral axis" and then proceeded to map it to behavioral axes anyway.

Claude caught this during spec review. The finding was precise: the current render pipeline gates `responseStyleGuidance()` on the Jungian vocabulary URI. A Sarc7 vocab implementing those methods would never get called without pipeline changes. And more fundamentally, `AgentDescriptor` has one `dispositionVocabulary` field — you can't set it to both Jungian and Sarc7 simultaneously. The claimed "Additive" relationship between the two frameworks had no mechanism.

The fix required stepping back to first principles. Sarcasm style is a *communication style overlay*, not a behavioral model. It modifies how an agent talks, independent of how it behaves. A collaborative, rule-following agent can use deadpan sarcasm. An autonomous, bold agent can use polite sarcasm. The dimensions are orthogonal.

The descriptor had no concept of communication style as a first-class category. That was the gap. We added two new fields: `styleVocabulary` on `AgentDescriptor` and `styleProfile` on `AgentDisposition` — a second personality profile slot parallel to the existing `dispositionProfile`. The behavioral profile (Jungian function stacks) auto-derives to axes. The communication style profile (Sarc7 sarcasm types) does not. Coexistence is natural: `dispositionVocabulary=jungian` + `styleVocabulary=sarc7`.

The render pipeline gained a hybrid three-layer architecture. Framework-specific methods handle structural rendering — the Jungian method keeps all its function-stack logic, the Sarc7 method handles sarcasm type identification and dimension context. A shared `renderGuidanceBlock()` helper renders `responseStyleGuidance()` and `antiPatternWarning()` for any vocabulary term. And a generic fallback sweep catches future vocabularies that provide guidance but don't have a dedicated renderer. The Jungian method was refactored to use the shared helper — extracting what was already there into a reusable form.

The Sarc7 vocabulary itself carries each type's evaluation dimensions as read-only fields. They're intrinsic properties of each type — deadpan IS high-incongruity, low-shock — not per-agent tuning knobs. The `responseStyleGuidance()` for deadpan: "Deliver humor through understatement and matter-of-fact delivery. Never signal that you are being humorous. Let the incongruity between your flat tone and the content do the work." The `antiPatternWarning()`: "Do not use exclamation marks, emoji, 'lol', or any tonal marker that telegraphs humor."

The open question is whether any of this matters. LLMs already produce credible sarcasm from plain English instructions. The value of structured vocabulary — consistency across a fleet, behavioral monitoring against declared style, model-portable parameterization — scales with agent count and deployment complexity. For a single agent on a single model, the incremental value over "be sarcastic" in the system prompt is modest. The eval harness will answer whether the structured terms produce measurably different output. If they don't, the vocabulary sits unused. If they do, it's already wired in.
