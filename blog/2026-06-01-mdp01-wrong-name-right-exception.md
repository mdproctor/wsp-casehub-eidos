---
layout: post
title: "Wrong Name, Right Exception"
date: 2026-06-01
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-eidos]
tags: [eval, validation, naming, exceptions]
---

# Wrong Name, Right Exception

The previous branch closed three issues: validation across the descriptor family,
and a multi-format eval harness. A code review pass afterwards found five things
worth fixing. Claude and I worked through them.

## When the constant describes the wrong field

`modelFamily` and `modelVersion` were both validated against `MAX_PROVIDER` (200).
The bound is right; the name is wrong. A reader checking the limit for `modelFamily`
finds a constant named after `provider` and either trusts the value without
understanding it, or traces it manually to confirm.

Same problem one field down: `dataHandlingPolicy` validated against `MAX_JURISDICTION`
(1000). Compliance text gets 1000 characters — that's intentional, and the grouping
with jurisdiction makes sense. The name makes that invisible.

We added `MAX_MODEL_IDENTIFIER = 200` and `MAX_DATA_HANDLING_POLICY = 1000`, both
with inline comments noting the shared bound. The fix is mechanical. The code now
reads as intended.

## A comment that misattributed null enforcement

`validateRequired` delegates to the private `validateField` — it has no null check
of its own. The proposed comment read "null is not accepted." Accurate about the
outcome; wrong about where enforcement happens. A reader following the call chain
sees `validateRequired` pass null straight through and has to open `validateField`
to confirm the comment isn't lying.

The accurate version: `// Unlike validateOptional, passes null to validateField — where it throws.`

There was also no test directly named `validateRequired_null_throws` — only indirect
coverage via `AgentCapabilityTest`. We added one.

## The test that couldn't access its own helper

`EvalReportWriterTest` had a `sampleReport()` helper that built an `EvalResult` and
immediately wrapped it in `EvalReport.build()`. No way to get at the `EvalResult`
without a fragile map lookup on the returned report.

Writing a multi-format test required two EvalResults. We split the helper:
`sampleMarkdownResult()` returns the `EvalResult`, and `sampleReport()` delegates
to it. The new test assembles both results directly:

```java
final EvalReport report = EvalReport.build(
    List.of(sampleMarkdownResult(), a2aResult), "test-judge");
```

The isolation assertion is deliberate: `assertThat(table.substring(a2aPos)).containsIgnoringCase("completeness")` — not checking that "completeness" appears somewhere in the table, but that it appears in the A2A_CARD section. `COMPLETENESS` is not a MARKDOWN dimension; a cross-section appearance would be wrong.

## Two IllegalStateExceptions doing different jobs

`PromptJudge.evaluate()` had this catch structure:

```java
} catch (final IllegalStateException e) {
    throw e;
} catch (final Exception e) {
    throw new IllegalStateException("Judge LLM call failed — ...", e);
}
```

The first catch re-throws because `parseResponse()` throws `IllegalStateException`
for malformed responses — missing dimensions, absent `score` or `reasoning` fields.
Those should propagate as-is, not be wrapped as "LLM call failed", which implies a
network problem.

Two `IllegalStateException`s with different semantics, distinguished only by which
catch block catches them. We replaced with `MalformedJudgeResponseException extends
RuntimeException`. The catch block explains itself without a comment. Callers can
distinguish parsing failure from infrastructure failure if they need to.

The refactor surfaced a gap: `mapper.readTree(json)` throwing `JsonProcessingException`
(non-JSON from the judge) still falls through to "LLM call failed". Same wrong message
for a different cause. Filed as casehubio/eidos#25.

## Four dimensions, except when it's two

`EvalResult.overall` carried the comment `// 0.0–5.0; mean of all four EvalDimension scores`. A2A_CARD uses two: `COMPLETENESS` and `FACTUAL_FIDELITY`. The comment was
wrong for every A2A result.

Fixed to `// 0.0–5.0; mean of applicable EvalDimension scores for the result's format`.
