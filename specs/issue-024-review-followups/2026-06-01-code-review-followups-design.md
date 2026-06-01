# Design: Code Review Follow-ups — eidos#21/20/22 Branch
**Issue:** casehubio/eidos#24  
**Date:** 2026-06-01  
**Scope:** Five minor findings from final review of `issue-021-020-eval-coverage-validation`. No correctness issues; all are naming clarity, comment gaps, or test coverage.

---

## Finding 1 — `validateRequired` comment

**File:** `api/src/main/java/io/casehub/eidos/api/AgentDescriptorValidator.java` line 31

`validateRequired` is a named entry point for single required-field validation. It delegates to the private `validateField` by design — callers cannot call `validateField` directly. The pair `validateRequired / validateOptional` is a reader contract: required throws on null, optional skips it. The method is not dead code: `AgentCapability` calls it for `capability.name`.

**Change:** Add one comment line above the method body explaining the semantic distinction from `validateOptional`.

```java
// Named entry point for required fields — unlike validateOptional, null is not accepted.
static void validateRequired(final String fieldName, final String value, final int maxLength) {
    validateField(fieldName, value, maxLength);
}
```

---

## Finding 2 — `MAX_MODEL_IDENTIFIER` constant

**Files:** `AgentDescriptorValidator.java` (constant), `AgentDescriptor.java` lines 35–36 (call sites)

`modelFamily` and `modelVersion` are validated against `MAX_PROVIDER` (200). The bound is numerically correct but the constant name describes the wrong field.

**Change:** Add `static final int MAX_MODEL_IDENTIFIER = 200;` to `AgentDescriptorValidator`. Update `AgentDescriptor` lines 35–36 to reference `MAX_MODEL_IDENTIFIER`. `MAX_PROVIDER` stays at 200 and remains the bound for `provider` only. No behavioural change.

---

## Finding 3 — `MAX_JURISDICTION` comment

**File:** `AgentDescriptorValidator.java` line 18

`dataHandlingPolicy` is validated against `MAX_JURISDICTION` (1000). The grouping is intentional — both are compliance-text fields that need generous space — but is not visible at the declaration site.

**Change:** Add an inline comment to the constant:

```java
static final int MAX_JURISDICTION = 1000; // compliance-text fields; shared with dataHandlingPolicy
```

No change to call sites in `AgentDescriptor.java`.

---

## Finding 4 — Multi-format writer test

**File:** `eval/src/test/java/io/casehub/eidos/eval/EvalReportWriterTest.java`

The existing `summaryTable_contains_format_header` test only verifies a single-format report (`=== MARKDOWN`). There is no test confirming that two independent format sections appear when a report spans multiple formats.

**Change:** Add `summaryTable_two_formats_have_separate_headers()`. The test:
1. Reuses the existing MARKDOWN `EvalResult` from `sampleReport()`.
2. Constructs a second `EvalResult` for `RenderFormat.A2A_CARD` with `COMPLETENESS` and `FACTUAL_FIDELITY` scores.
3. Calls `EvalReport.build(List.of(markdownResult, a2aResult), "test-judge")`.
4. Asserts the summary table contains both `=== MARKDOWN` and `=== A2A_CARD`.
5. Asserts `completeness` appears in the table (A2A_CARD dimension, absent from MARKDOWN section), confirming format isolation.

---

## Finding 5 — PromptJudge re-throw comment

**File:** `eval/src/main/java/io/casehub/eidos/eval/PromptJudge.java` lines 194–195

The `catch (IllegalStateException e) { throw e; }` block before the generic `catch (Exception e)` is unusual but deliberate: `parseResponse` throws `IllegalStateException` when the judge response is malformed, and those exceptions must propagate as-is rather than being wrapped as "Judge LLM call failed" (which implies a network/call failure, not a response parsing failure).

**Change:** Add one comment:

```java
} catch (final IllegalStateException e) {
    throw e; // parseResponse throws ISE for malformed LLM responses — don't wrap as "LLM call failed"
}
```

---

## Out of scope

`quarkus-junit5` deprecation (all modules still use the deprecated artifact ID — tracked in casehubio/platform#19). Not addressed here.

---

## Platform coherence

All five changes are internal to `casehub-eidos-api` and `casehub-eidos-eval`. No SPI changes, no public API breaks, no platform doc updates required.
