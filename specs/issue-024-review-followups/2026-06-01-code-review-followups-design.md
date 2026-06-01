# Design: Code Review Follow-ups — eidos#21/20/22 Branch
**Issue:** casehubio/eidos#24  
**Date:** 2026-06-01  
**Scope:** Five minor findings from final review of `issue-021-020-eval-coverage-validation`, plus one pre-existing documentation correctness issue in `EvalResult`. No correctness issues in runtime behaviour.

---

## Finding 1 — `validateRequired` comment + missing test

**File:** `api/src/main/java/io/casehub/eidos/api/AgentDescriptorValidator.java` line 31

`validateRequired` is a named entry point for single required-field validation and is called by `AgentCapability` for `capability.name`. It delegates to the private `validateField`, which is where null enforcement happens — `validateRequired` itself has no null check.

The comment must not claim "null is not accepted" at this method, as the null check is in `validateField`. Accurate wording:

```java
// Unlike validateOptional, passes null to validateField — where it throws.
static void validateRequired(final String fieldName, final String value, final int maxLength) {
    validateField(fieldName, value, maxLength);
}
```

**New test** in `AgentDescriptorValidatorTest`:
```java
@Test
void validateRequired_null_throws() {
    assertThatThrownBy(
        () -> AgentDescriptorValidator.validateRequired("capability.name", null, 100))
        .isInstanceOf(AgentValidationException.class)
        .satisfies(ex -> assertThat(((AgentValidationException) ex).fieldName())
            .isEqualTo("capability.name"));
}
```

*Note:* There is a structural improvement available — move validation logic into `validateRequired` and make `validateOptional` a guard-then-call wrapper. This eliminates the delegation indirection and removes the need for the comment. Out of scope here; captured in idea log.

---

## Finding 2 — `MAX_MODEL_IDENTIFIER` constant + boundary tests

**Files:** `AgentDescriptorValidator.java` (constant), `AgentDescriptor.java` lines 35–36 (call sites)

`modelFamily` and `modelVersion` are validated against `MAX_PROVIDER` (200). The bound is correct but the constant name describes the wrong field.

**Change:** Add constant:
```java
static final int MAX_MODEL_IDENTIFIER = 200; // modelFamily, modelVersion
```
Update `AgentDescriptor.java` lines 35–36 to reference `MAX_MODEL_IDENTIFIER`. `MAX_PROVIDER` stays at 200 for `provider` only.

**New tests** in `AgentDescriptorTest` (boundary safety net for the rename):
```java
@Test
void model_family_exceeds_200_chars_throws() {
    assertThatThrownBy(() -> new AgentDescriptor(
        "id", "Name", null, null, "f".repeat(201), null, null,
        null, null, null, "worker", List.of(), null, null, null, "tenant"))
        .isInstanceOf(AgentValidationException.class)
        .satisfies(ex -> assertThat(((AgentValidationException) ex).fieldName())
            .isEqualTo("modelFamily"));
}

@Test
void model_version_exceeds_200_chars_throws() {
    assertThatThrownBy(() -> new AgentDescriptor(
        "id", "Name", null, null, null, "v".repeat(201), null,
        null, null, null, "worker", List.of(), null, null, null, "tenant"))
        .isInstanceOf(AgentValidationException.class)
        .satisfies(ex -> assertThat(((AgentValidationException) ex).fieldName())
            .isEqualTo("modelVersion"));
}
```

---

## Finding 3 — `MAX_DATA_HANDLING_POLICY` constant + boundary test

**Files:** `AgentDescriptorValidator.java` (constant), `AgentDescriptor.java` line 42 (call site)

`dataHandlingPolicy` is validated against `MAX_JURISDICTION` (1000) — same problem class as Finding 2: a field is validated against a constant that describes a different field. Consistent fix: new constant (not a comment).

**Change:** Add constant:
```java
static final int MAX_DATA_HANDLING_POLICY = 1000; // compliance-text field; same bound as MAX_JURISDICTION
```
Update `AgentDescriptor.java` line 42 to reference `MAX_DATA_HANDLING_POLICY`. `MAX_JURISDICTION` stays at 1000 for `jurisdiction` only.

**New tests** in `AgentDescriptorTest`:
```java
@Test
void data_handling_policy_blank_throws() {
    assertThatThrownBy(() -> new AgentDescriptor(
        "id", "Name", null, null, null, null, null,
        null, null, null, "worker", List.of(), null, null, "  ", "tenant"))
        .isInstanceOf(AgentValidationException.class)
        .satisfies(ex -> assertThat(((AgentValidationException) ex).fieldName())
            .isEqualTo("dataHandlingPolicy"));
}

@Test
void data_handling_policy_exceeds_1000_chars_throws() {
    assertThatThrownBy(() -> new AgentDescriptor(
        "id", "Name", null, null, null, null, null,
        null, null, null, "worker", List.of(), null, null, "p".repeat(1001), "tenant"))
        .isInstanceOf(AgentValidationException.class)
        .satisfies(ex -> assertThat(((AgentValidationException) ex).fieldName())
            .isEqualTo("dataHandlingPolicy"));
}
```

---

## Finding 4 — Multi-format writer test + `sampleMarkdownResult()` refactor

**File:** `eval/src/test/java/io/casehub/eidos/eval/EvalReportWriterTest.java`

The existing `summaryTable_contains_format_header` test only verifies a single-format report. A multi-format test is needed to confirm that two independent format sections are emitted.

**Required refactor:** `sampleReport()` currently builds an `EvalReport` — there is no `EvalResult` directly accessible without a fragile map/index lookup. Refactor into:

```java
static EvalResult sampleMarkdownResult() {
    final var desc = new AgentDescriptor(
        "a", "Agent", null, null, null, null, null,
        null, null, null, "worker", List.of(), null, null, null, "t");
    final var ctx = AgentPromptContext.forFormat(RenderFormat.MARKDOWN);
    final var rendered = new RenderedPrompt("You are Agent.", RenderFormat.MARKDOWN, "dh", "ch");
    final Map<EvalDimension, EvalScore> scores = new EnumMap<>(EvalDimension.class);
    scores.put(EvalDimension.SECOND_PERSON,    new EvalScore(4, "good"));
    scores.put(EvalDimension.CONCISENESS,      new EvalScore(4, "good"));
    scores.put(EvalDimension.FACTUAL_FIDELITY, new EvalScore(4, "good"));
    scores.put(EvalDimension.TONE,             new EvalScore(4, "good"));
    return new EvalResult(new EvalCase("case1", desc, ctx), rendered,
        true, List.of(), scores, 4.0, List.of());
}

static EvalReport sampleReport() {
    return EvalReport.build(List.of(sampleMarkdownResult()), "test-judge");
}
```

**New test** `summaryTable_two_formats_have_separate_headers()`:

```java
@Test
void summaryTable_two_formats_have_separate_headers() {
    // A2A_CARD result
    final var desc = new AgentDescriptor(
        "b", "Bot", null, null, null, null, null,
        null, null, null, "worker", List.of(), null, null, null, "t");
    final var ctx = AgentPromptContext.forFormat(RenderFormat.A2A_CARD);
    final var rendered = new RenderedPrompt("{\"name\":\"Bot\"}", RenderFormat.A2A_CARD, "dh", "ch");
    final Map<EvalDimension, EvalScore> a2aScores = new EnumMap<>(EvalDimension.class);
    a2aScores.put(EvalDimension.COMPLETENESS,     new EvalScore(4, "good"));
    a2aScores.put(EvalDimension.FACTUAL_FIDELITY, new EvalScore(4, "good"));
    final var a2aResult = new EvalResult(new EvalCase("case2", desc, ctx), rendered,
        true, List.of(), a2aScores, 4.0, List.of());

    final EvalReport report = EvalReport.build(
        List.of(sampleMarkdownResult(), a2aResult), "test-judge");
    final String table = EvalReportWriter.summaryTable(report);

    assertThat(table).contains("=== MARKDOWN");
    assertThat(table).contains("=== A2A_CARD");

    // Verify isolation: "completeness" appears in the A2A_CARD section, not in MARKDOWN
    final int a2aPos = table.indexOf("=== A2A_CARD");
    assertThat(a2aPos).isGreaterThan(-1);
    final String a2aSection = table.substring(a2aPos);
    assertThat(a2aSection).containsIgnoringCase("completeness");
}
```

---

## Finding 5 — `MalformedJudgeResponseException` (structural fix)

**Files:** `eval/src/main/java/io/casehub/eidos/eval/MalformedJudgeResponseException.java` (new), `PromptJudge.java` lines 194–195 + `parseResponse()`

Two `IllegalStateException`s with different semantics (malformed LLM response vs. network/config failure) are currently distinguished only by catch-block position — the exception type does no semantic work. A comment compensating for this will drift.

**New exception class** (internal to eval, no API impact):

```java
package io.casehub.eidos.eval;

class MalformedJudgeResponseException extends RuntimeException {
    MalformedJudgeResponseException(final String message) {
        super(message);
    }
}
```

**Update `parseResponse()`:** replace `throw new IllegalStateException(...)` with `throw new MalformedJudgeResponseException(...)` for all response-parsing failures.

**Update catch block in `evaluate()`:**

```java
} catch (final MalformedJudgeResponseException e) {
    throw e;
} catch (final Exception e) {
    throw new IllegalStateException("Judge LLM call failed — check judge model configuration", e);
}
```

The catch block is now self-documenting. Callers can distinguish response parsing failure from infrastructure failure if needed.

---

## Pre-existing — `EvalResult.overall` comment

**File:** `eval/src/main/java/io/casehub/eidos/eval/EvalResult.java`

The `overall` field comment says `// 0.0–5.0; mean of all four EvalDimension scores`. A2A_CARD has two applicable dimensions, not four.

**Fix:**
```java
double overall,      // 0.0–5.0; mean of applicable EvalDimension scores for the result's format
```

---

## Out of scope

- `quarkus-junit5` deprecation across all modules — tracked in casehubio/platform#19.
- `validateRequired` structural refactor (move logic into `validateRequired`, make `validateOptional` guard-then-call) — captured in idea log.

---

## Platform coherence

All changes are internal to `casehub-eidos-api` and `casehub-eidos-eval`. No SPI changes, no public API breaks, no platform doc updates required.
