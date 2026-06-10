---
layout: post
title: "The Evaluation That Tests Itself"
date: 2026-06-10
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-eidos]
tags: [eval, behavioral-validation, disposition-axis, type-migration]
---

The eval harness has been sitting in the repo since eidos#16 — designed, built, never run. The five judges exist, the real-world profiles exist, the pair contrast infrastructure exists. What's been missing is: first, a `ChatModel` bean so the judges have something to call; second, any evidence that the scores the harness produces mean anything at all.

That's what eidos#46 was for. First principles. Run the thing and find out.

Before I got to running anything, I had to get the type system in order. `DispositionAxis` is an enum — `RISK_APPETITE`, `RULE_FOLLOWING`, `SOCIAL_ORIENTATION`, `AUTONOMY`, `CONFLICT_MODE`. But the eval module had been treating it as a string since the beginning: `List.of("riskAppetite", "ruleFollowing", ...)`, `Map<String, Integer>` for scores, `Map<String, TraitPolarity>` for expected traits. Nothing was wrong exactly — it all compiled and ran. It was just that the axes existed as typed enum constants in the API and as camelCase strings in the eval, with nothing to make the two correspond except convention.

I migrated them: typed `AXES` and `NUMERIC_AXES` constants in every judge, enum keys in `AgentProfile.expectedTraits`, `DispositionAxis.jsonKey()` to bridge back to camelCase for the JSON output that the LLM produces and the result records still carry. The result records deliberately keep `Map<String, Integer>` — Jackson serializes enum keys as `RISK_APPETITE` by default, which would change the report format and break nothing of consequence but was unnecessary churn. The computation is typed; the output is stable.

One thing I hit during the migration: `c.primaryAxis().equals(axis)` — where `c.primaryAxis()` had been migrated to `DispositionAxis` but `axis` was still a `String` from the old constant list. This compiles. Java's `Object.equals(Object)` accepts anything. At runtime, `DispositionAxis.equals("riskAppetite")` returns false — as required by the equals contract when comparing incompatible types. So the streaming filter matched nothing, every attribution in `PersonalityPreservationReport.computeDiagnoses()` collapsed to `INSUFFICIENT_DATA`, and no exception was raised to explain why. That took a moment to find. The fix is `c.primaryAxis() == axis` once both are `DispositionAxis` — enum identity comparison, which is what you should be using for enums anyway.

For Phase 3 — the behavioral validation loop — I had originally designed something that seemed obvious: render the system prompt, invoke the agent, then judge whether the responses match the expected profile. `riskAppetite: HIGH` → judge scores each response for boldness. If score ≥ 4, pass.

This is circular. The judge sees the expected values. It knows the answer before reading the responses. Even if the system prompt had zero behavioral effect, you'd still score high on the "does this response match bold?" rubric if you ask bold-sounding questions of a capable model. There's no control.

The right structure already existed in Phase 2 — `PairContrastJudge`. Give the judge two prompts, ask which expresses the axis more strongly, check whether it picks the right one. The judge is blind: it doesn't see `riskAppetite: HIGH`, it just sees Prompt A and Prompt B and describes what it observes. For behavioral validation, you do the same thing one level down: render both system prompts, invoke both agents on the same question, give the judge both responses, ask which is bolder. Compare the judge's answer to ground truth.

Six data points — two variant pairs, three questions each — is not statistically powered. It's a sanity check. If the system prompt is doing nothing, we'd expect random chance (three correct out of six). If the signal is real, we'd expect most of them correct. We won't know the floor until we run it.

The bridge between the platform's `AgentProvider` SPI and LangChain4j's `ChatModel` is a single class: `AgentProviderChatModel`. `@DefaultBean @ApplicationScoped` in the eval test sources, so Jlama's own `ChatModel @ApplicationScoped` displaces it automatically if Jlama is on the classpath. `ResponseFormat` is ignored — the Claude CLI doesn't expose structured output parameters. The judge prompts are written to produce JSON without schema enforcement, which works reliably enough that I'm not worried about it for a first run.

The 0.8 accuracy target in the issue description is aspirational. The actual `ACCURACY_FLOOR` constant is `0.0` until we run the harness and observe what's achievable. If pair contrast can detect behavioral differentiation at all — if the RISK_APPETITE difference between `sw-engineer-bold` and `sw-engineer-careful` actually shows up in their responses to the same code review question — that's the real result. Whether 80% is achievable or whether the signal is weaker than that, I don't know yet. The harness exists to find out.
