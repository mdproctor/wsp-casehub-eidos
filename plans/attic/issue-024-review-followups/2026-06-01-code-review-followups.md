# Code Review Follow-ups (eidos#24) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Resolve five minor code-review findings from branch `issue-021-020-eval-coverage-validation` — clarify naming, add missing boundary tests, add a structural exception class, and fix a wrong comment.

**Architecture:** Six independent changes across two Maven modules (`api`, `eval`). Tasks 1–3 touch `casehub-eidos-api`; Tasks 4–6 touch `casehub-eidos-eval`. Each task is a self-contained commit. Tasks within a module are ordered to avoid merge conflicts on shared files.

**Tech Stack:** Java 21 (on Java 26 JVM), JUnit 5, AssertJ, LangChain4j inline stubs (no Mockito).

---

## File Map

| File | Change |
|------|--------|
| `api/src/main/java/io/casehub/eidos/api/AgentDescriptorValidator.java` | Add comment to `validateRequired`; add `MAX_MODEL_IDENTIFIER`; add `MAX_DATA_HANDLING_POLICY` |
| `api/src/main/java/io/casehub/eidos/api/AgentDescriptor.java` | Switch call sites to new constants |
| `api/src/test/java/io/casehub/eidos/api/AgentDescriptorValidatorTest.java` | Add `validateRequired_null_throws` |
| `api/src/test/java/io/casehub/eidos/api/AgentDescriptorTest.java` | Add 4 boundary tests |
| `eval/src/main/java/io/casehub/eidos/eval/MalformedJudgeResponseException.java` | New class |
| `eval/src/main/java/io/casehub/eidos/eval/PromptJudge.java` | Update `parseResponse()` throws; update catch block in `evaluate()` |
| `eval/src/main/java/io/casehub/eidos/eval/EvalResult.java` | Fix `overall` field comment |
| `eval/src/test/java/io/casehub/eidos/eval/PromptJudgeTest.java` | Update `parseResponse_throws_when_applicable_dimension_missing` to assert `MalformedJudgeResponseException` |
| `eval/src/test/java/io/casehub/eidos/eval/EvalReportWriterTest.java` | Refactor `sampleReport()` into `sampleMarkdownResult()` + `sampleReport()`; add `summaryTable_two_formats_have_separate_headers` |

---

## Task 1: validateRequired comment + null test

**Files:**
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentDescriptorValidator.java:31-33`
- Modify: `api/src/test/java/io/casehub/eidos/api/AgentDescriptorValidatorTest.java`

- [ ] **Step 1: Write the failing test**

Add this test to `AgentDescriptorValidatorTest.java` immediately after `validateOptional_null_is_allowed()`:

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

- [ ] **Step 2: Run the test**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=AgentDescriptorValidatorTest#validateRequired_null_throws
```

Expected: **PASS** — the test documents existing behaviour; null is already rejected by `validateField` which `validateRequired` delegates to.

- [ ] **Step 3: Add the comment to validateRequired**

In `AgentDescriptorValidator.java`, replace lines 31-33:

```java
static void validateRequired(final String fieldName, final String value, final int maxLength) {
    validateField(fieldName, value, maxLength);
}
```

with:

```java
// Unlike validateOptional, passes null to validateField — where it throws.
static void validateRequired(final String fieldName, final String value, final int maxLength) {
    validateField(fieldName, value, maxLength);
}
```

- [ ] **Step 4: Run the full api test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api
```

Expected: all tests PASS.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add api/src/main/java/io/casehub/eidos/api/AgentDescriptorValidator.java api/src/test/java/io/casehub/eidos/api/AgentDescriptorValidatorTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "docs(eidos#24): clarify validateRequired delegation; add null contract test"
```

---

## Task 2: MAX_MODEL_IDENTIFIER constant + boundary tests

**Files:**
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentDescriptorValidator.java:15-16`
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentDescriptor.java:35-36`
- Modify: `api/src/test/java/io/casehub/eidos/api/AgentDescriptorTest.java`

- [ ] **Step 1: Write the boundary tests**

Add these two tests to `AgentDescriptorTest.java`, after `vocabulary_uri_exceeds_500_chars_throws`:

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

- [ ] **Step 2: Run the new tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest="AgentDescriptorTest#model_family_exceeds_200_chars_throws+model_version_exceeds_200_chars_throws"
```

Expected: **PASS** — confirms existing behaviour before renaming the constant.

- [ ] **Step 3: Add the MAX_MODEL_IDENTIFIER constant**

In `AgentDescriptorValidator.java`, add one line after `MAX_PROVIDER`:

```java
// Optional field bounds — accessible within package for compact constructors
static final int MAX_VERSION             = 200;
static final int MAX_PROVIDER            = 200;
static final int MAX_MODEL_IDENTIFIER    = 200; // modelFamily, modelVersion
static final int MAX_WEIGHTS_FINGERPRINT = 255;
static final int MAX_VOCABULARY_URI      = 500;
static final int MAX_JURISDICTION        = 1000;
static final int MAX_DISPOSITION_AXIS    = 200;
static final int MAX_CAPABILITY_NAME     = 100;
static final int MAX_CAPABILITY_STRING   = 200;
```

- [ ] **Step 4: Update the call sites in AgentDescriptor.java**

Replace lines 35-36 of `AgentDescriptor.java`:

```java
        AgentDescriptorValidator.validateOptional("modelFamily",           modelFamily,           AgentDescriptorValidator.MAX_PROVIDER);
        AgentDescriptorValidator.validateOptional("modelVersion",          modelVersion,          AgentDescriptorValidator.MAX_PROVIDER);
```

with:

```java
        AgentDescriptorValidator.validateOptional("modelFamily",           modelFamily,           AgentDescriptorValidator.MAX_MODEL_IDENTIFIER);
        AgentDescriptorValidator.validateOptional("modelVersion",          modelVersion,          AgentDescriptorValidator.MAX_MODEL_IDENTIFIER);
```

- [ ] **Step 5: Run the full api test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api
```

Expected: all tests PASS.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add api/src/main/java/io/casehub/eidos/api/AgentDescriptorValidator.java api/src/main/java/io/casehub/eidos/api/AgentDescriptor.java api/src/test/java/io/casehub/eidos/api/AgentDescriptorTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "refactor(eidos#24): MAX_MODEL_IDENTIFIER for modelFamily/modelVersion; add boundary tests"
```

---

## Task 3: MAX_DATA_HANDLING_POLICY constant + boundary tests

**Files:**
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentDescriptorValidator.java` (after Task 2's state)
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentDescriptor.java:42`
- Modify: `api/src/test/java/io/casehub/eidos/api/AgentDescriptorTest.java`

- [ ] **Step 1: Write the boundary tests**

Add these two tests to `AgentDescriptorTest.java`, after `data_handling_policy_null_is_allowed`:

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

- [ ] **Step 2: Run the new tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest="AgentDescriptorTest#data_handling_policy_blank_throws+data_handling_policy_exceeds_1000_chars_throws"
```

Expected: **PASS** — confirms existing behaviour before renaming the constant.

- [ ] **Step 3: Add the MAX_DATA_HANDLING_POLICY constant**

In `AgentDescriptorValidator.java`, add one line after `MAX_JURISDICTION`:

```java
// Optional field bounds — accessible within package for compact constructors
static final int MAX_VERSION              = 200;
static final int MAX_PROVIDER             = 200;
static final int MAX_MODEL_IDENTIFIER     = 200; // modelFamily, modelVersion
static final int MAX_WEIGHTS_FINGERPRINT  = 255;
static final int MAX_VOCABULARY_URI       = 500;
static final int MAX_JURISDICTION         = 1000;
static final int MAX_DATA_HANDLING_POLICY = 1000; // compliance-text field; same bound as MAX_JURISDICTION
static final int MAX_DISPOSITION_AXIS     = 200;
static final int MAX_CAPABILITY_NAME      = 100;
static final int MAX_CAPABILITY_STRING    = 200;
```

- [ ] **Step 4: Update the call site in AgentDescriptor.java**

Replace line 42 of `AgentDescriptor.java`:

```java
        AgentDescriptorValidator.validateOptional("dataHandlingPolicy",    dataHandlingPolicy,    AgentDescriptorValidator.MAX_JURISDICTION);
```

with:

```java
        AgentDescriptorValidator.validateOptional("dataHandlingPolicy",    dataHandlingPolicy,    AgentDescriptorValidator.MAX_DATA_HANDLING_POLICY);
```

- [ ] **Step 5: Run the full api test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api
```

Expected: all tests PASS.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add api/src/main/java/io/casehub/eidos/api/AgentDescriptorValidator.java api/src/main/java/io/casehub/eidos/api/AgentDescriptor.java api/src/test/java/io/casehub/eidos/api/AgentDescriptorTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "refactor(eidos#24): MAX_DATA_HANDLING_POLICY constant; add boundary tests"
```

---

## Task 4: EvalReportWriterTest refactor + multi-format test

**Files:**
- Modify: `eval/src/test/java/io/casehub/eidos/eval/EvalReportWriterTest.java`

- [ ] **Step 1: Write the failing multi-format test**

Add this test to `EvalReportWriterTest.java`. It calls `sampleMarkdownResult()` which does not exist yet — the test will fail at compilation.

```java
@Test
void summaryTable_two_formats_have_separate_headers() {
    // A2A_CARD result
    final var desc = new AgentDescriptor(
        "b", "Bot", null, null, null, null, null,
        null, null, null, "worker", List.of(), null, null, null, "t");
    final var ctx = AgentPromptContext.forFormat(RenderFormat.A2A_CARD);
    final var rendered = new RenderedPrompt(
        "{\"name\":\"Bot\"}", RenderFormat.A2A_CARD, "dh", "ch");
    final Map<EvalDimension, EvalScore> a2aScores = new EnumMap<>(EvalDimension.class);
    a2aScores.put(EvalDimension.COMPLETENESS,     new EvalScore(4, "good"));
    a2aScores.put(EvalDimension.FACTUAL_FIDELITY, new EvalScore(4, "good"));
    final var a2aResult = new EvalResult(
        new EvalCase("case2", desc, ctx), rendered, true, List.of(), a2aScores, 4.0, List.of());

    final EvalReport report = EvalReport.build(
        List.of(sampleMarkdownResult(), a2aResult), "test-judge");
    final String table = EvalReportWriter.summaryTable(report);

    assertThat(table).contains("=== MARKDOWN");
    assertThat(table).contains("=== A2A_CARD");

    // "completeness" must appear in the A2A_CARD section, not just anywhere in the table
    final int a2aPos = table.indexOf("=== A2A_CARD");
    assertThat(a2aPos).isGreaterThan(-1);
    final String a2aSection = table.substring(a2aPos);
    assertThat(a2aSection).containsIgnoringCase("completeness");
}
```

- [ ] **Step 2: Run the test — expect compilation failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -pl eval 2>&1 | grep "cannot find symbol\|sampleMarkdownResult"
```

Expected: compilation error — `cannot find symbol: method sampleMarkdownResult()`.

- [ ] **Step 3: Refactor sampleReport() into sampleMarkdownResult() + sampleReport()**

In `EvalReportWriterTest.java`, replace the current `sampleReport()` method:

```java
static EvalReport sampleReport() {
    final var desc = new AgentDescriptor(
        "a", "Agent", null, null, null, null, null,
        null, null, null, "worker", List.of(), null, null, null, "t");
    final var ctx = AgentPromptContext.forFormat(RenderFormat.MARKDOWN);
    final var rendered = new RenderedPrompt("You are Agent.", RenderFormat.MARKDOWN, "dh", "ch");
    // MARKDOWN uses SECOND_PERSON, CONCISENESS, FACTUAL_FIDELITY, TONE
    final Map<EvalDimension, EvalScore> scores = new EnumMap<>(EvalDimension.class);
    scores.put(EvalDimension.SECOND_PERSON,    new EvalScore(4, "good"));
    scores.put(EvalDimension.CONCISENESS,      new EvalScore(4, "good"));
    scores.put(EvalDimension.FACTUAL_FIDELITY, new EvalScore(4, "good"));
    scores.put(EvalDimension.TONE,             new EvalScore(4, "good"));
    final var result = new EvalResult(new EvalCase("case1", desc, ctx), rendered,
        true, List.of(), scores, 4.0, List.of());
    return EvalReport.build(List.of(result), "test-judge");
}
```

with these two methods:

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

- [ ] **Step 4: Run the full eval test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval
```

Expected: all tests PASS, including the new `summaryTable_two_formats_have_separate_headers`.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add eval/src/test/java/io/casehub/eidos/eval/EvalReportWriterTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "test(eidos#24): multi-format summaryTable test; refactor sampleMarkdownResult helper"
```

---

## Task 5: MalformedJudgeResponseException

**Files:**
- Create: `eval/src/main/java/io/casehub/eidos/eval/MalformedJudgeResponseException.java`
- Modify: `eval/src/main/java/io/casehub/eidos/eval/PromptJudge.java`
- Modify: `eval/src/test/java/io/casehub/eidos/eval/PromptJudgeTest.java`

- [ ] **Step 1: Update the existing test to assert the new exception type**

In `PromptJudgeTest.java`, change the assertion in `parseResponse_throws_when_applicable_dimension_missing()`:

Replace:
```java
assertThatThrownBy(() -> new PromptJudge(stub, new ObjectMapper()).evaluate(evalCase, rendered))
    .isInstanceOf(IllegalStateException.class)
    .hasMessageContaining("missing dimension");
```

with:
```java
assertThatThrownBy(() -> new PromptJudge(stub, new ObjectMapper()).evaluate(evalCase, rendered))
    .isInstanceOf(MalformedJudgeResponseException.class)
    .hasMessageContaining("missing dimension");
```

- [ ] **Step 2: Run the test — expect compilation failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -pl eval 2>&1 | grep "cannot find symbol\|MalformedJudge"
```

Expected: compilation error — `cannot find symbol: class MalformedJudgeResponseException`.

- [ ] **Step 3: Create MalformedJudgeResponseException.java**

Create `eval/src/main/java/io/casehub/eidos/eval/MalformedJudgeResponseException.java`:

```java
package io.casehub.eidos.eval;

class MalformedJudgeResponseException extends RuntimeException {
    MalformedJudgeResponseException(final String message) {
        super(message);
    }
}
```

- [ ] **Step 4: Update parseResponse() in PromptJudge.java**

In `PromptJudge.java`, in the `parseResponse()` method (lines 252-270), change both `throw new IllegalStateException(...)` calls to `throw new MalformedJudgeResponseException(...)`.

Replace:
```java
        for (final EvalDimension d : applicable) {
            final JsonNode dimNode = root.get(d.name());
            if (dimNode == null) {
                throw new IllegalStateException("Judge response missing dimension: " + d.name());
            }
            final JsonNode scoreNode = dimNode.get("score");
            final JsonNode reasoningNode = dimNode.get("reasoning");
            if (scoreNode == null || reasoningNode == null) {
                throw new IllegalStateException(
                    "Judge response for " + d.name() + " missing 'score' or 'reasoning'");
            }
            scores.put(d, new EvalScore(scoreNode.asInt(), reasoningNode.asText()));
        }
```

with:
```java
        for (final EvalDimension d : applicable) {
            final JsonNode dimNode = root.get(d.name());
            if (dimNode == null) {
                throw new MalformedJudgeResponseException("Judge response missing dimension: " + d.name());
            }
            final JsonNode scoreNode = dimNode.get("score");
            final JsonNode reasoningNode = dimNode.get("reasoning");
            if (scoreNode == null || reasoningNode == null) {
                throw new MalformedJudgeResponseException(
                    "Judge response for " + d.name() + " missing 'score' or 'reasoning'");
            }
            scores.put(d, new EvalScore(scoreNode.asInt(), reasoningNode.asText()));
        }
```

- [ ] **Step 5: Update the catch block in evaluate()**

In `PromptJudge.java`, in the `evaluate()` method (lines 194-198), replace:

```java
        } catch (final IllegalStateException e) {
            throw e;
        } catch (final Exception e) {
            throw new IllegalStateException("Judge LLM call failed — check judge model configuration", e);
        }
```

with:

```java
        } catch (final MalformedJudgeResponseException e) {
            throw e;
        } catch (final Exception e) {
            throw new IllegalStateException("Judge LLM call failed — check judge model configuration", e);
        }
```

- [ ] **Step 6: Run the full eval test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval
```

Expected: all tests PASS, including `parseResponse_throws_when_applicable_dimension_missing`.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add eval/src/main/java/io/casehub/eidos/eval/MalformedJudgeResponseException.java eval/src/main/java/io/casehub/eidos/eval/PromptJudge.java eval/src/test/java/io/casehub/eidos/eval/PromptJudgeTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "refactor(eidos#24): MalformedJudgeResponseException — distinguish parse failure from LLM call failure"
```

---

## Task 6: Fix EvalResult.overall comment

**Files:**
- Modify: `eval/src/main/java/io/casehub/eidos/eval/EvalResult.java:6`

No test for a comment fix. Change only.

- [ ] **Step 1: Fix the comment**

In `EvalResult.java`, replace:

```java
        double overall,      // 0.0–5.0; mean of all four EvalDimension scores
```

with:

```java
        double overall,      // 0.0–5.0; mean of applicable EvalDimension scores for the result's format
```

- [ ] **Step 2: Run the full eval test suite to confirm no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval
```

Expected: all tests PASS.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add eval/src/main/java/io/casehub/eidos/eval/EvalResult.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "docs(eidos#24): fix EvalResult.overall comment — A2A_CARD has 2 applicable dimensions, not 4"
```

---

## Final Verification

- [ ] **Run the complete build across all modules**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean test
```

Expected: BUILD SUCCESS, all tests pass across `api`, `eval`, `runtime`, `deployment`, `vocab`, `persistence-memory`.
