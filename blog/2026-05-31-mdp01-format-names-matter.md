---
layout: post
title: "Format Names Matter"
date: 2026-05-31
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-eidos]
tags: [eval, validation, render-format, injection-surface]
---

# Format Names Matter

Three issues going in: extend validation to `AgentCapability` and `AgentDisposition` (eidos#20, #22), and expand the eval harness to cover all render formats (eidos#21). I expected the validation work to be mechanical. It was. The eval work had a surprise in it.

## A naming problem masquerading as a feature request

`RenderFormat` had four members: `CLAUDE_MD`, `OPENAI_SYSTEM`, `A2A_CARD`, `GEMINI`. All named after providers or a protocol. The question I couldn't answer during the design session: where does Grok go? Or Qwen? Every LLM that prefers dense prose over markdown would need its own member — except they'd all be structurally identical to `OPENAI_SYSTEM`.

I asked how `GEMINI` differed from `OPENAI_SYSTEM`. The answer was one space in a resource citation. `label (uri)` versus `label(uri)`. That's not a format. That's a micro-styling choice.

We renamed. `CLAUDE_MD` → `MARKDOWN`. `OPENAI_SYSTEM` and `GEMINI` collapsed into `PROSE`. Three members, named after structure. The eval harness went from four formats to three. `EidosRenderPipeline` lost `assembleGemini` and `assembleGeminiStructural` — two methods nearly identical to `assembleOpenAiSystem`. The deletion was the implementation.

## The completeness bug hiding in the JSON

The existing completeness check in `PromptJudge` does a substring search: does `rendered.content()` contain each capability name? For prose formats — correct and meaningful. For A2A_CARD — always true.

In a JSON card, every capability object contains a `name` field. `"sprint-planning"` is always present in the rendered string, embedded in `{"name":"sprint-planning","qualityHint":0.9}`, regardless of whether a `description` was generated. The check would always pass.

The fix makes the check format-aware: for JSON, parse the card and verify that each capability object has a non-empty `description` field. That's the actual quality signal for A2A — capability presence isn't the question, description quality is.

## One more surface at a time

`AgentCapability` — `name` is required by semantics: it's the capability identifier, used in the A2A card and the completeness check itself. Null would break the check. `costHint`, list items in `inputTypes`/`outputTypes`/`tags`, and `epistemicDomains` keys all get the character-set rules. They flow into the LLM payload or the card.

`AgentDisposition` — four open-String axes for behavioural description. All optional, but if provided: non-blank, ≤200 chars, no banned characters. Compact constructor validates itself; the record is self-consistent wherever it's constructed.

`AgentDescriptor` — 10 optional String fields, same rules. Vocabulary URIs get 500 chars, jurisdiction and data policy fields get 1000. The exception renamed: `AgentDescriptorValidationException` → `AgentValidationException`. The old name was wrong the moment `AgentCapability` started throwing it.

## Per-format summaries only

The original `EvalReport` had a flat `List<EvalResult>` and a single `EvalSummary`. That worked with one format. With MARKDOWN, PROSE, and A2A_CARD in play, averaging CONCISENESS across markdown prose and dense prose produces a number that means nothing. Averaging prose dimensions with `COMPLETENESS` — which only applies to A2A — is worse.

We replaced the flat shape with `Map<RenderFormat, List<EvalResult>>` and `Map<RenderFormat, EvalSummary>`. Each format gets its own summary. `EvalDimension.applicableFor(format)` — a static method on the enum — is the single source of truth for which dimensions apply to which format. Both the judge and the report builder call it independently; neither knows about the other.

The harness now covers 9 cases: 5 MARKDOWN, 2 PROSE, 2 A2A_CARD. The PROSE rubric scores any markdown headers or bullets at ≤2 for CONCISENESS. A2A cases use `COMPLETENESS` and `FACTUAL_FIDELITY` only; `SECOND_PERSON` and `TONE` don't mean anything for JSON output.
