---
layout: post
title: "A language the renderer understands"
date: 2026-06-09
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-eidos]
tags: [vocabulary, llm, renderer, disposition]
---

`AgentDescriptor` has had disposition axes for a while — five open strings for
how an agent operates. The Belbin and DISC vocabularies landed last week. What it
didn't have was any way to tell the renderer that a descriptor with
`socialOrient: "independent"` was instantiating a specific concept from a specific
framework. The LLM saw raw strings and produced generic prose.

The fix sounds simple: include the vocabulary name alongside the value. In practice
it required decisions at every layer.

I'd thought about adding an explicit `theoreticalFrameworks` field to
`AgentDescriptor`. Two reasons I didn't. First, it duplicates what's already in the
vocabulary URI — `urn:casehub:vocab:belbin`, "Belbin Team Roles", both already
registered. Second, the renderer still can't produce canonical language without
knowing what the framework says, and that lives in `@VocabularyMetadata.description()`.
A new descriptor field doesn't solve the renderer problem; it just moves it one
step later.

What needed to change was the annotation. `@VocabularyMetadata` had `uri`, `name`,
and `version`. Adding `description` — a short empirical basis note — gives the
pipeline enough context to tell the LLM not just which framework but what kind of
thing it is. The Conscientiousness vocabulary now reads: "An operational axis
vocabulary for agent behavioral disposition, grounded in Big Five Conscientiousness
research. Covers four of the five disposition axes. Does not cover CONFLICT_MODE —
use Thomas-Kilmann for that axis." That last sentence came from a spec review that
caught that the original description made the axis split invisible.

The pipeline change was bigger. Disposition axes had been flat strings in the LLM
payload — `"socialOrient": "independent"`. They're now nested objects per axis:
`{"value": "independent", "label": "Independent", "vocabularyName": "Conscientiousness
Disposition Axes"}`. The loop over `DispositionAxis.values()` replaces four
hand-written if-blocks and fixes a pre-existing bug I found in the process:
`conflictMode` had never been serialised. It was in `AgentDisposition` and
`DispositionAxis.CONFLICT_MODE`, but the renderer silently skipped it. The prose
fallback was worse — it only rendered `ruleFollowing` and `autonomy`, dropping the
other three.

The helpers that go with the loop — `axisJsonKey` and `axisLabel` — both use
exhaustive switches with no default branch. Adding a sixth axis to `DispositionAxis`
causes a compile error at every mapping site simultaneously. No grep, no
pull-request review scan. The type system does the impact analysis.

The spec review cycles surfaced a helper naming distinction worth keeping.
`addIfPresent` checks `!= null` — right for nullable descriptor fields.
`addIfNonBlank` checks `!= null && !isEmpty()` — right for annotation attributes,
because `@VocabularyMetadata.name()` with `default ""` returns an empty string, never
null. Using `addIfPresent` for annotation-sourced fields produces
`"vocabularyName": ""` in the JSON payload. Looks correct. Wrong.

The integration tests confirmed the output. An agent with Conscientiousness
disposition vocabulary now renders `"Facilitative (Conscientiousness Disposition Axes)"`
in structural markdown. Thomas-Kilmann conflict mode shows up in rendered output for
the first time. The hash-uniqueness test — two descriptors, same disposition value,
different vocabulary URI — verified the cache key separates them correctly.

The renderer now knows what the frameworks are. Whether the LLM produces noticeably
better prompts is a claim that needs a real LLM in the loop, which the test
environment doesn't have. The structure is right; the quality will show in production.
