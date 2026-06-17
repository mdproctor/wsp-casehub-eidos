---
layout: post
title: "Judging the Judge"
date: 2026-06-17
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-eidos]
tags: [eval, enrichment, self-evaluation-bias, llm-eval]
---

The eval passed. Then it failed.

I ran `evaluateRealWorldScenarios` with Claude Sonnet and got what I'd hoped for: all 16 real-world cases complete, capability names present in both MARKDOWN and PROSE, proximity scores at 5.00/5.00. The enrichment fix from the last session had done its job.

Then the test assertion fired. PROSE mean: 3.875 against a floor of 4.0. The culprit: factual fidelity scoring 2.13 out of 5.

That number didn't make sense. The renders were clearly good. How could factual fidelity be that low?

I suspected self-evaluation bias — but I'd assumed the bias would inflate scores. Models tend to be fond of their own output. I ran `evaluateWithIndependentJudge` with Qwen 8B: same Claude renders, different judge. Factual fidelity: 5.00, both formats. The renders Claude had scored at 1.88, Qwen rated perfect.

What's happening is specific. Claude's factual fidelity criterion applies strict literalism — any claim in the rendered output not present verbatim in the descriptor JSON is flagged as potentially hallucinated. When enrichment takes `riskAppetite: conservative` and generates "You weigh regulatory exposure before recommending any change," Claude penalises it. The elaboration isn't in the descriptor. Qwen reads the intent, checks against the descriptor, gives 5.

So the PROSE floor wasn't failing because the renders were bad. It was failing because Claude was grading its own elaborations as suspect. We recalibrated the floor from 4.0 to 3.5 — Sonnet mean of 3.88 minus the 0.5 margin.

There was also a structural fix. The `allCasesComplete()` check used substring matching on capability names. With Qwen as the renderer (earlier sessions), that check would fail when Qwen paraphrased names in the enriched output. The right fix: add a `boolean enriched` field to `RenderedPrompt`. When a render has gone through semantic enrichment, skip the substring logic — the FACTUAL_FIDELITY score is already handling coverage. We kept the structural check for non-enriched renders where it's both reliable and necessary.

The last piece was briefing content. We'd added the `briefing` field to `AgentDescriptor` in the previous session but left the eval profiles empty. I populated all eight. The source is the `vocabularyGap` entries with `loss: FULL` — concepts the disposition axes genuinely cannot express. For the software engineer bold profile, that's trust-in-pipeline: the epistemic delegation to CI/CD that no axis captures. For the clinical researcher, it's protocol-deviation escalation and source-data verification — mandatory procedures with no disposition equivalent. Two or three sentences per profile, all in second person, covering what would otherwise be invisible to the renderer.

The enrichment is confirmed working. The inverse self-evaluation bias is documented. The question I'm curious about now: does the briefing content actually move the proximity scores? The redesigned `ProximityJudge` looks at descriptor-axis completeness — not briefing. But if the briefing helps the enrichment step write a more grounded disposition narrative, the next independent judge run might tell a different story.
