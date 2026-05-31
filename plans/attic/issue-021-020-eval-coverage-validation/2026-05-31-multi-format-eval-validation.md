# Multi-Format Eval Coverage + String Field Validation

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rename RenderFormat to structure-based names (MARKDOWN/PROSE/A2A_CARD), extend string validation to all AgentDescriptor/AgentCapability/AgentDisposition fields, and add multi-format eval coverage.

**Architecture:** Three sequenced issues on one branch. Part 1 (Tasks 1-2) renames RenderFormat — foundational, must land first. Part 2 (Tasks 3-6) extends validation infrastructure and records. Part 3 (Tasks 7-13) adds multi-format eval coverage in the eval module.

**Tech Stack:** Java 21, Quarkus 3.32.2, JUnit 5, AssertJ, LangChain4j, Jackson

**Build command:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
**Module test:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl <module>`
**Single test:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl <module> -Dtest=ClassName`

---

## File Map

**Modified — api/:**
- `api/SystemPromptRenderer.java` — rename RenderFormat members (CLAUDE_MD→MARKDOWN, OPENAI_SYSTEM→PROSE, remove GEMINI)
- `api/AgentDescriptorValidationException.java` → renamed to `AgentValidationException.java`
- `api/AgentDescriptorValidator.java` — add validateRequired, validateOptional, validateItems, validateMapKeys + constants
- `api/AgentDescriptor.java` — add optional field validation in compact constructor
- `api/AgentDisposition.java` — add compact constructor
- `api/AgentCapability.java` — add compact constructor

**Modified — runtime/:**
- `runtime/renderer/EidosRenderPipeline.java` — update switch statements, rename assembleClaudeMarkdown→assembleMarkdown, collapse assembleOpenAiSystem+assembleGemini→assembleProse
- `runtime/test/cache/BlockingToReactiveRenderedPromptCacheAdapterTest.java` — CLAUDE_MD→MARKDOWN
- `runtime/test/renderer/DefaultReactiveSystemPromptRendererStreamingTest.java` — CLAUDE_MD→MARKDOWN
- `runtime/test/renderer/EidosRenderPipelineTest.java` — CLAUDE_MD→MARKDOWN, update cache key assertion
- `runtime/test/renderer/EidosSystemPromptRendererTest.java` — CLAUDE_MD→MARKDOWN, OPENAI_SYSTEM→PROSE, delete GEMINI tests

**Modified — eval/:**
- `eval/EvalDimension.java` — add COMPLETENESS, add applicableFor(RenderFormat)
- `eval/PromptJudge.java` — format-aware evaluation, 3 system prompts, 2 schemas, format-aware completeness, parameterised parseResponse
- `eval/EvalDataset.java` — add 4 new cases (2 PROSE, 2 A2A_CARD), update all() + CLAUDE_MD→MARKDOWN
- `eval/EvalReport.java` — clean break to Map<RenderFormat,…> shape
- `eval/EvalReportWriter.java` — format-grouped summaryTable

**New — api/:**
- `api/AgentValidationException.java` — renamed from AgentDescriptorValidationException
- `api/test/AgentCapabilityTest.java`
- `api/test/AgentDispositionTest.java`

**New — eval/:**
- `eval/test/EvalReportTest.java`

**Modified — eval/test/:**
- `eval/test/PromptJudgeTest.java` — CLAUDE_MD→MARKDOWN, add A2A tests
- `eval/test/EvalDatasetTest.java` — update case count
- `eval/test/PromptEvalTest.java` — per-format floor assertions

**Modified — examples/:**
- `examples/agent-scenarios/test/SystemPromptRendererTest.java` — CLAUDE_MD→MARKDOWN

---

## Part 1 — RenderFormat Redesign

### Task 1: Rename RenderFormat + update EidosRenderPipeline

**Files:**
- Modify: `api/src/main/java/io/casehub/eidos/api/SystemPromptRenderer.java`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosRenderPipeline.java`

- [ ] **Step 1: Rename enum members in SystemPromptRenderer.java**

Replace the `RenderFormat` enum body:

```java
// Before:
enum RenderFormat {
    CLAUDE_MD, OPENAI_SYSTEM, A2A_CARD, GEMINI
}

// After:
enum RenderFormat {
    MARKDOWN,   // rich markdown — Claude and other markdown-capable models
    PROSE,      // flowing paragraphs — OpenAI, Gemini, Grok, Qwen, Mistral, Llama, ...
    A2A_CARD    // JSON — machine-readable agent identity card
}
```

- [ ] **Step 2: Update EidosRenderPipeline — usesEnrichment**

```java
// Before:
static boolean usesEnrichment(final RenderFormat format) {
    return switch (format) {
        case CLAUDE_MD, OPENAI_SYSTEM, GEMINI -> true;
        case A2A_CARD                          -> false;
    };
}

// After:
static boolean usesEnrichment(final RenderFormat format) {
    return switch (format) {
        case MARKDOWN, PROSE -> true;
        case A2A_CARD        -> false;
    };
}
```

- [ ] **Step 3: Update EidosRenderPipeline — assemble() switch**

```java
// Before:
final String content = switch (context.format()) {
    case CLAUDE_MD     -> assembleClaudeMarkdown(enrichment, descriptor, context);
    case OPENAI_SYSTEM -> assembleOpenAiSystem(enrichment, descriptor, context);
    case A2A_CARD      -> assembleA2aCard(a2aEnrichment, descriptor);
    case GEMINI        -> assembleGemini(enrichment, descriptor, context);
};

// After:
final String content = switch (context.format()) {
    case MARKDOWN  -> assembleMarkdown(enrichment, descriptor, context);
    case PROSE     -> assembleProse(enrichment, descriptor, context);
    case A2A_CARD  -> assembleA2aCard(a2aEnrichment, descriptor);
};
```

- [ ] **Step 4: Rename assembleClaudeMarkdown → assembleMarkdown**

In EidosRenderPipeline: rename the private method `assembleClaudeMarkdown` to `assembleMarkdown` and rename `assembleClaudeMarkdownStructural` to `assembleMarkdownStructural`. No logic change.

- [ ] **Step 5: Collapse assembleOpenAiSystem + assembleGemini → assembleProse**

Delete `assembleGemini` and `assembleGeminiStructural`. Rename `assembleOpenAiSystem` to `assembleProse`. In `assembleProse` (was `assembleOpenAiSystem`): the only change is removing the comment "// Structural OPENAI_SYSTEM — dense prose, no headers". The resource format `label (uri)` (with space) is already the standard. No logic change needed — the old OPENAI_SYSTEM implementation IS the PROSE implementation.

Delete `assembleGeminiStructural` entirely. Its caller (`assembleGemini`) is also deleted.

- [ ] **Step 6: Verify the compile — run api + runtime builds**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api,runtime
```

Expected: BUILD SUCCESS. If compile fails on GEMINI/CLAUDE_MD/OPENAI_SYSTEM references in other files — that's expected, will be fixed in Task 2.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add api/src/main/java/io/casehub/eidos/api/SystemPromptRenderer.java runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosRenderPipeline.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "refactor(eidos#21): rename RenderFormat CLAUDE_MD→MARKDOWN, OPENAI_SYSTEM+GEMINI→PROSE

Structure-named formats: MARKDOWN, PROSE, A2A_CARD. Provider-named formats
(OPENAI_SYSTEM, GEMINI) were structurally identical except for one space in
resource formatting — not sufficient to justify separate formats. Collapse
assembleOpenAiSystem+assembleGemini into assembleProse (label (uri) standard).

Refs #21"
```

---

### Task 2: Update all callers of old format names

**Files:**
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/cache/BlockingToReactiveRenderedPromptCacheAdapterTest.java`
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/renderer/DefaultReactiveSystemPromptRendererStreamingTest.java`
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/renderer/EidosRenderPipelineTest.java`
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/renderer/EidosSystemPromptRendererTest.java`
- Modify: `examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/SystemPromptRendererTest.java`

- [ ] **Step 1: BlockingToReactiveRenderedPromptCacheAdapterTest**

```java
// Before:
static final RenderedPrompt PROMPT = new RenderedPrompt("content", RenderFormat.CLAUDE_MD, "dh", "ch");

// After:
static final RenderedPrompt PROMPT = new RenderedPrompt("content", RenderFormat.MARKDOWN, "dh", "ch");
```

- [ ] **Step 2: DefaultReactiveSystemPromptRendererStreamingTest**

Replace all occurrences: `CLAUDE_MD` → `MARKDOWN` (5 occurrences: 4 `forFormat(CLAUDE_MD)` calls and 1 `CLAUDE_MD` in a RenderedPrompt constructor, 1 import, 1 assertion `isEqualTo(CLAUDE_MD)`).

```java
// Import line — no change needed if using static import of RenderFormat.*
// Change all: CLAUDE_MD → MARKDOWN
```

- [ ] **Step 3: EidosRenderPipelineTest**

Two changes:
```java
// 1. All forFormat(CLAUDE_MD) → forFormat(MARKDOWN)
// 2. Cache key assertion:
assertThat(s1.lookupKey()).contains("CLAUDE_MD");  // before
assertThat(s1.lookupKey()).contains("MARKDOWN");    // after

// 3. usesEnrichment test name update (optional — test still passes):
void uses_enrichment_true_for_claude_md() // → keep or rename to uses_enrichment_true_for_markdown
```

- [ ] **Step 4: EidosSystemPromptRendererTest — renames**

Replace all occurrences:
- `CLAUDE_MD` → `MARKDOWN` (all uses of `forFormat(CLAUDE_MD)`, assertions `isEqualTo(CLAUDE_MD)`)
- `OPENAI_SYSTEM` → `PROSE` (all uses of `forFormat(OPENAI_SYSTEM)`, assertions)
- Comment `// ── OPENAI_SYSTEM path` → `// ── PROSE path`
- Comment `// ── Structural CLAUDE_MD path` → `// ── Structural MARKDOWN path`

- [ ] **Step 5: EidosSystemPromptRendererTest — delete GEMINI tests**

Delete the entire `// ── GEMINI path` section (lines 320–353):
- `gemini_structural_has_no_markdown_headers()`
- `gemini_enriched_has_no_markdown_headers()`
- `gemini_enriched_contains_identity_and_role_narrative()`
- `gemini_enriched_resources_format_uses_no_space_before_paren()`

These are covered by: MARKDOWN tests (no-headers check) and PROSE tests (resource format with space). The Gemini-specific no-space-before-paren assertion is deleted because PROSE uses space-before-paren.

Also update the cache-key-distinguishes-formats test (around line 385):
```java
// Before:
final var claudeCtx  = fullContext();  // format = CLAUDE_MD
final var openaiCtx  = AgentPromptContext.forFormat(OPENAI_SYSTEM)...
assertThat(claudeResult.format()).isEqualTo(CLAUDE_MD);
assertThat(openaiResult.format()).isEqualTo(OPENAI_SYSTEM);

// After:
final var markdownCtx = fullContext();  // format = MARKDOWN (fullContext() uses CLAUDE_MD → update)
final var proseCtx    = AgentPromptContext.forFormat(PROSE)...
assertThat(markdownResult.format()).isEqualTo(MARKDOWN);
assertThat(proseResult.format()).isEqualTo(PROSE);
```

- [ ] **Step 6: SystemPromptRendererTest in examples**

Replace `RenderFormat.CLAUDE_MD` → `RenderFormat.MARKDOWN` (3 occurrences).

- [ ] **Step 7: Run full test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api,runtime,examples/agent-scenarios
```

Expected: BUILD SUCCESS, all tests pass. Fix any remaining compilation errors.

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add \
  runtime/src/test \
  examples/agent-scenarios/src/test
git -C /Users/mdproctor/claude/casehub/eidos commit -m "refactor(eidos#21): update all call sites to MARKDOWN/PROSE — delete GEMINI tests

Refs #21"
```

---

## Part 2 — Validation Infrastructure

### Task 3: Rename AgentDescriptorValidationException → AgentValidationException

**Files:**
- Rename: `api/src/main/java/io/casehub/eidos/api/AgentDescriptorValidationException.java` → `AgentValidationException.java`
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentDescriptorValidator.java`
- Modify: `api/src/test/java/io/casehub/eidos/api/AgentDescriptorValidatorTest.java`

- [ ] **Step 1: Create AgentValidationException.java**

```java
package io.casehub.eidos.api;

public final class AgentValidationException extends IllegalArgumentException {

    private final String fieldName;

    public AgentValidationException(final String fieldName, final String message) {
        super("field '" + fieldName + "': " + message);
        this.fieldName = fieldName;
    }

    public String fieldName() {
        return fieldName;
    }
}
```

- [ ] **Step 2: Update AgentDescriptorValidator — replace exception type**

Replace all `new AgentDescriptorValidationException(` → `new AgentValidationException(` (4 occurrences in `validateField`).

- [ ] **Step 3: Delete AgentDescriptorValidationException.java**

```bash
git -C /Users/mdproctor/claude/casehub/eidos rm api/src/main/java/io/casehub/eidos/api/AgentDescriptorValidationException.java
```

- [ ] **Step 4: Update AgentDescriptorValidatorTest**

Replace:
```java
// Before:
.isInstanceOf(AgentDescriptorValidationException.class)
.satisfies(ex -> assertThat(((AgentDescriptorValidationException) ex).fieldName())

// After:
.isInstanceOf(AgentValidationException.class)
.satisfies(ex -> assertThat(((AgentValidationException) ex).fieldName())
```

Remove import of `AgentDescriptorValidationException`, add import of `AgentValidationException`.

- [ ] **Step 5: Run api tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api
```

Expected: BUILD SUCCESS, all tests pass.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add api/src/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "refactor(eidos#22): rename AgentDescriptorValidationException → AgentValidationException

Name was misleading once AgentCapability throws it. Generic name covers
all agent domain record validation.

Refs #22"
```

---

### Task 4: Extend AgentDescriptorValidator

**Files:**
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentDescriptorValidator.java`
- Modify: `api/src/test/java/io/casehub/eidos/api/AgentDescriptorValidatorTest.java`

- [ ] **Step 1: Write failing tests for new validator methods**

Add to `AgentDescriptorValidatorTest.java`:

```java
// ── validateOptional ──────────────────────────────────────────────────────

@Test
void validateOptional_null_is_allowed() {
    assertThatNoException().isThrownBy(
        () -> AgentDescriptorValidator.validateOptional("field", null, 200));
}

@Test
void validateOptional_blank_throws() {
    assertThatThrownBy(
        () -> AgentDescriptorValidator.validateOptional("provider", "   ", 200))
        .isInstanceOf(AgentValidationException.class)
        .satisfies(ex -> assertThat(((AgentValidationException) ex).fieldName())
            .isEqualTo("provider"));
}

@Test
void validateOptional_exceeds_length_throws() {
    assertThatThrownBy(
        () -> AgentDescriptorValidator.validateOptional("version", "v".repeat(201), 200))
        .isInstanceOf(AgentValidationException.class);
}

@Test
void validateOptional_bidi_control_throws() {
    assertThatThrownBy(
        () -> AgentDescriptorValidator.validateOptional("jurisdiction", "EU‮EVILCONTENT", 1000))
        .isInstanceOf(AgentValidationException.class);
}

@Test
void validateOptional_valid_value_does_not_throw() {
    assertThatNoException().isThrownBy(
        () -> AgentDescriptorValidator.validateOptional("provider", "anthropic", 200));
}

// ── validateItems ──────────────────────────────────────────────────────────

@Test
void validateItems_null_list_is_allowed() {
    assertThatNoException().isThrownBy(
        () -> AgentDescriptorValidator.validateItems("inputTypes", null, 200));
}

@Test
void validateItems_blank_item_throws_with_index() {
    assertThatThrownBy(
        () -> AgentDescriptorValidator.validateItems("inputTypes", List.of("ok", ""), 200))
        .isInstanceOf(AgentValidationException.class)
        .satisfies(ex -> assertThat(((AgentValidationException) ex).fieldName())
            .isEqualTo("inputTypes[1]"));
}

@Test
void validateItems_injection_item_throws() {
    assertThatThrownBy(
        () -> AgentDescriptorValidator.validateItems("tags", List.of("valid", "bad​zero"), 200))
        .isInstanceOf(AgentValidationException.class);
}

// ── validateMapKeys ────────────────────────────────────────────────────────

@Test
void validateMapKeys_null_set_is_allowed() {
    assertThatNoException().isThrownBy(
        () -> AgentDescriptorValidator.validateMapKeys("epistemicDomains", null, 200));
}

@Test
void validateMapKeys_bidi_key_throws() {
    assertThatThrownBy(
        () -> AgentDescriptorValidator.validateMapKeys("epistemicDomains",
            Set.of("java؜injection"), 200))
        .isInstanceOf(AgentValidationException.class);
}
```

Add required imports at top of the test file:
```java
import java.util.List;
import java.util.Set;
```

- [ ] **Step 2: Run to confirm tests fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=AgentDescriptorValidatorTest
```

Expected: compilation failure or test failure — `validateOptional`, `validateItems`, `validateMapKeys` do not exist yet.

- [ ] **Step 3: Add constants and new methods to AgentDescriptorValidator**

Full updated `AgentDescriptorValidator.java`:

```java
package io.casehub.eidos.api;

import java.util.Set;

class AgentDescriptorValidator {

    // Required field bounds
    private static final int MAX_AGENT_ID   = 255;
    private static final int MAX_NAME       = 200;
    private static final int MAX_SLOT       = 100;
    private static final int MAX_TENANCY_ID = 255;

    // Optional field bounds — accessible within package for compact constructors
    static final int MAX_VERSION             = 200;
    static final int MAX_PROVIDER            = 200;
    static final int MAX_WEIGHTS_FINGERPRINT = 255;
    static final int MAX_VOCABULARY_URI      = 500;
    static final int MAX_JURISDICTION        = 1000;
    static final int MAX_DISPOSITION_AXIS    = 200;
    static final int MAX_CAPABILITY_NAME     = 100;
    static final int MAX_CAPABILITY_STRING   = 200;

    static void validate(final String agentId, final String name,
                          final String slot, final String tenancyId) {
        validateField("agentId",   agentId,   MAX_AGENT_ID);
        validateField("name",      name,      MAX_NAME);
        validateField("slot",      slot,      MAX_SLOT);
        validateField("tenancyId", tenancyId, MAX_TENANCY_ID);
    }

    static void validateRequired(final String fieldName, final String value, final int maxLength) {
        validateField(fieldName, value, maxLength);
    }

    static void validateOptional(final String fieldName, final String value, final int maxLength) {
        if (value == null) return;
        validateField(fieldName, value, maxLength);
    }

    static void validateItems(final String fieldName, final Iterable<String> items,
                               final int maxLength) {
        if (items == null) return;
        int index = 0;
        for (final String item : items) {
            validateOptional(fieldName + "[" + index + "]", item, maxLength);
            index++;
        }
    }

    static void validateMapKeys(final String fieldName, final Set<String> keys,
                                 final int maxLength) {
        if (keys == null) return;
        for (final String key : keys) {
            validateOptional(fieldName + ".key", key, maxLength);
        }
    }

    private static void validateField(final String fieldName, final String value,
                                       final int maxLength) {
        if (value == null) {
            throw new AgentValidationException(fieldName, "must not be null");
        }
        if (value.isBlank()) {
            throw new AgentValidationException(fieldName, "must not be blank");
        }
        if (value.length() > maxLength) {
            throw new AgentValidationException(fieldName,
                "exceeds maximum length " + maxLength + " (was " + value.length() + ")");
        }
        for (int i = 0; i < value.length(); ) {
            final int cp = value.codePointAt(i);
            if (isBanned(cp)) {
                throw new AgentValidationException(fieldName,
                    "contains banned character U+" + String.format("%04X", cp));
            }
            i += Character.charCount(cp);
        }
    }

    private static boolean isBanned(final int cp) {
        if (cp <= 0x001F) return true;
        if (cp >= 0x007F && cp <= 0x009F) return true;
        if (cp == 0x061C) return true;
        if (cp == 0x200E || cp == 0x200F) return true;
        if (cp >= 0x202A && cp <= 0x202E) return true;
        if (cp >= 0x2066 && cp <= 0x2069) return true;
        if (cp == 0x200B || cp == 0xFEFF) return true;
        if (cp == 0x200C || cp == 0x200D) return true;
        if (cp == 0x2028 || cp == 0x2029) return true;
        return false;
    }
}
```

- [ ] **Step 4: Run tests — confirm all pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=AgentDescriptorValidatorTest
```

Expected: BUILD SUCCESS, all tests pass (including the new ones).

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add api/src/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#22,#20): extend AgentDescriptorValidator with optional/items/mapKeys

Adds validateOptional (null-permissive, blank-rejecting), validateItems
(per-element validation with index in field name), validateMapKeys (key
validation). Adds length constants for all optional field categories.

Refs #22, #20"
```

---

### Task 5: AgentCapability validation (TDD)

**Files:**
- Create: `api/src/test/java/io/casehub/eidos/api/AgentCapabilityTest.java`
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentCapability.java`

- [ ] **Step 1: Write all failing tests**

Create `api/src/test/java/io/casehub/eidos/api/AgentCapabilityTest.java`:

```java
package io.casehub.eidos.api;

import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThatNoException;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.assertj.core.api.Assertions.assertThat;

class AgentCapabilityTest {

    static AgentCapability valid() {
        return new AgentCapability("code-review", 0.9, 100L, "low",
            List.of("pull-request"), List.of("review-comment"), List.of("java"),
            Map.of("java", 0.95));
    }

    // ── name (required) ────────────────────────────────────────────────────────

    @Test
    void name_null_throws() {
        assertThatThrownBy(() ->
            new AgentCapability(null, null, null, null, List.of(), List.of(), List.of(), Map.of()))
            .isInstanceOf(AgentValidationException.class)
            .satisfies(ex -> assertThat(((AgentValidationException) ex).fieldName())
                .isEqualTo("capability.name"));
    }

    @Test
    void name_blank_throws() {
        assertThatThrownBy(() ->
            new AgentCapability("  ", null, null, null, List.of(), List.of(), List.of(), Map.of()))
            .isInstanceOf(AgentValidationException.class);
    }

    @Test
    void name_exceeds_100_chars_throws() {
        assertThatThrownBy(() ->
            new AgentCapability("n".repeat(101), null, null, null, List.of(), List.of(), List.of(), Map.of()))
            .isInstanceOf(AgentValidationException.class);
    }

    @Test
    void name_with_bidi_control_throws() {
        assertThatThrownBy(() ->
            new AgentCapability("code‮review", null, null, null, List.of(), List.of(), List.of(), Map.of()))
            .isInstanceOf(AgentValidationException.class);
    }

    @Test
    void name_at_exactly_100_chars_is_valid() {
        assertThatNoException().isThrownBy(() ->
            new AgentCapability("n".repeat(100), null, null, null, List.of(), List.of(), List.of(), Map.of()));
    }

    // ── costHint (optional) ────────────────────────────────────────────────────

    @Test
    void cost_hint_null_is_allowed() {
        assertThatNoException().isThrownBy(() ->
            new AgentCapability("review", null, null, null, List.of(), List.of(), List.of(), Map.of()));
    }

    @Test
    void cost_hint_blank_throws() {
        assertThatThrownBy(() ->
            new AgentCapability("review", null, null, "  ", List.of(), List.of(), List.of(), Map.of()))
            .isInstanceOf(AgentValidationException.class)
            .satisfies(ex -> assertThat(((AgentValidationException) ex).fieldName())
                .isEqualTo("costHint"));
    }

    @Test
    void cost_hint_with_injection_char_throws() {
        assertThatThrownBy(() ->
            new AgentCapability("review", null, null, "low​cost", List.of(), List.of(), List.of(), Map.of()))
            .isInstanceOf(AgentValidationException.class);
    }

    // ── list fields ────────────────────────────────────────────────────────────

    @Test
    void input_types_null_list_is_allowed() {
        assertThatNoException().isThrownBy(() ->
            new AgentCapability("review", null, null, null, null, List.of(), List.of(), Map.of()));
    }

    @Test
    void input_types_blank_item_throws_with_index() {
        assertThatThrownBy(() ->
            new AgentCapability("review", null, null, null,
                List.of("pr", ""), List.of(), List.of(), Map.of()))
            .isInstanceOf(AgentValidationException.class)
            .satisfies(ex -> assertThat(((AgentValidationException) ex).fieldName())
                .isEqualTo("inputTypes[1]"));
    }

    @Test
    void output_types_item_with_c0_control_char_throws() {  // U+0001 SOH embedded in string literal
        assertThatThrownBy(() ->
            new AgentCapability("review", null, null, null,
                List.of(), List.of("commentinject"), List.of(), Map.of()))
            .isInstanceOf(AgentValidationException.class);
    }

    @Test
    void tags_item_exceeds_200_chars_throws() {
        assertThatThrownBy(() ->
            new AgentCapability("review", null, null, null,
                List.of(), List.of(), List.of("t".repeat(201)), Map.of()))
            .isInstanceOf(AgentValidationException.class);
    }

    // ── epistemicDomains keys ──────────────────────────────────────────────────

    @Test
    void epistemic_domain_key_with_alm_throws() {
        assertThatThrownBy(() ->
            new AgentCapability("review", null, null, null,
                List.of(), List.of(), List.of(), Map.of("java؜domain", 0.9)))
            .isInstanceOf(AgentValidationException.class);
    }

    @Test
    void epistemic_domain_null_map_is_allowed() {
        assertThatNoException().isThrownBy(() ->
            new AgentCapability("review", null, null, null, List.of(), List.of(), List.of(), null));
    }

    // ── full valid construction ────────────────────────────────────────────────

    @Test
    void valid_capability_constructs_cleanly() {
        assertThatNoException().isThrownBy(AgentCapabilityTest::valid);
    }
}
```

- [ ] **Step 2: Run to confirm tests fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=AgentCapabilityTest
```

Expected: tests fail — no compact constructor exists yet.

- [ ] **Step 3: Add compact constructor to AgentCapability.java**

```java
public record AgentCapability(
        String name,
        Double qualityHint,
        Long latencyHintP50Ms,
        String costHint,
        List<String> inputTypes,
        List<String> outputTypes,
        List<String> tags,
        Map<String, Double> epistemicDomains
) {
    public AgentCapability {
        AgentDescriptorValidator.validateRequired("capability.name", name,
            AgentDescriptorValidator.MAX_CAPABILITY_NAME);
        AgentDescriptorValidator.validateOptional("costHint", costHint,
            AgentDescriptorValidator.MAX_CAPABILITY_STRING);
        AgentDescriptorValidator.validateItems("inputTypes", inputTypes,
            AgentDescriptorValidator.MAX_CAPABILITY_STRING);
        AgentDescriptorValidator.validateItems("outputTypes", outputTypes,
            AgentDescriptorValidator.MAX_CAPABILITY_STRING);
        AgentDescriptorValidator.validateItems("tags", tags,
            AgentDescriptorValidator.MAX_CAPABILITY_STRING);
        if (epistemicDomains != null) {
            AgentDescriptorValidator.validateMapKeys("epistemicDomains",
                epistemicDomains.keySet(), AgentDescriptorValidator.MAX_CAPABILITY_STRING);
        }
    }
}
```

- [ ] **Step 4: Run api tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api
```

Expected: BUILD SUCCESS, all tests pass. Note: existing tests may break if they construct `AgentCapability` with null name — check `AgentDescriptorTest.java` and fix any such usages (use `"worker"` or another valid name).

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add api/src/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#22): AgentCapability compact constructor — validate all string fields

Validates name (required), costHint, inputTypes/outputTypes/tags items, and
epistemicDomains keys against the same character-set rules as AgentDescriptor.
Capability names and string items cap at 100 and 200 chars respectively.

Closes #22"
```

---

### Task 6: AgentDisposition and AgentDescriptor validation (TDD)

**Files:**
- Create: `api/src/test/java/io/casehub/eidos/api/AgentDispositionTest.java`
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentDisposition.java`
- Modify: `api/src/test/java/io/casehub/eidos/api/AgentDescriptorTest.java`
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentDescriptor.java`

- [ ] **Step 1: Write AgentDispositionTest**

Create `api/src/test/java/io/casehub/eidos/api/AgentDispositionTest.java`:

```java
package io.casehub.eidos.api;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThatNoException;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.assertj.core.api.Assertions.assertThat;

class AgentDispositionTest {

    @Test
    void all_null_axes_are_allowed() {
        assertThatNoException().isThrownBy(
            () -> new AgentDisposition(null, null, null, null, false));
    }

    @Test
    void blank_social_orient_throws() {
        assertThatThrownBy(() -> new AgentDisposition("  ", "strict", "low", "autonomous", false))
            .isInstanceOf(AgentValidationException.class)
            .satisfies(ex -> assertThat(((AgentValidationException) ex).fieldName())
                .isEqualTo("socialOrient"));
    }

    @Test
    void rule_following_with_bidi_throws() {
        assertThatThrownBy(
            () -> new AgentDisposition("collaborative", "strict‮injection", "low", "autonomous", false))
            .isInstanceOf(AgentValidationException.class)
            .satisfies(ex -> assertThat(((AgentValidationException) ex).fieldName())
                .isEqualTo("ruleFollowing"));
    }

    @Test
    void risk_appetite_exceeds_200_chars_throws() {
        assertThatThrownBy(
            () -> new AgentDisposition("collaborative", "strict", "r".repeat(201), "autonomous", false))
            .isInstanceOf(AgentValidationException.class);
    }

    @Test
    void autonomy_with_zero_width_throws() {
        assertThatThrownBy(
            () -> new AgentDisposition("collaborative", "strict", "low", "auto​nomous", false))
            .isInstanceOf(AgentValidationException.class)
            .satisfies(ex -> assertThat(((AgentValidationException) ex).fieldName())
                .isEqualTo("autonomy"));
    }

    @Test
    void valid_disposition_constructs_cleanly() {
        assertThatNoException().isThrownBy(
            () -> new AgentDisposition("collaborative", "adaptive", "moderate", "assisted", true));
    }

    @Test
    void delegation_is_not_validated_it_is_boolean() {
        assertThatNoException().isThrownBy(
            () -> new AgentDisposition(null, null, null, null, true));
    }
}
```

- [ ] **Step 2: Run to confirm AgentDispositionTest fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=AgentDispositionTest
```

Expected: tests fail — no compact constructor in AgentDisposition.

- [ ] **Step 3: Add compact constructor to AgentDisposition.java**

```java
public record AgentDisposition(
        String socialOrient,
        String ruleFollowing,
        String riskAppetite,
        String autonomy,
        boolean delegation
) {
    public AgentDisposition {
        AgentDescriptorValidator.validateOptional("socialOrient", socialOrient,
            AgentDescriptorValidator.MAX_DISPOSITION_AXIS);
        AgentDescriptorValidator.validateOptional("ruleFollowing", ruleFollowing,
            AgentDescriptorValidator.MAX_DISPOSITION_AXIS);
        AgentDescriptorValidator.validateOptional("riskAppetite", riskAppetite,
            AgentDescriptorValidator.MAX_DISPOSITION_AXIS);
        AgentDescriptorValidator.validateOptional("autonomy", autonomy,
            AgentDescriptorValidator.MAX_DISPOSITION_AXIS);
    }
}
```

- [ ] **Step 4: Run AgentDispositionTest — confirm pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=AgentDispositionTest
```

Expected: BUILD SUCCESS, all tests pass.

- [ ] **Step 5: Write failing tests for AgentDescriptor optional fields**

Add to `api/src/test/java/io/casehub/eidos/api/AgentDescriptorTest.java`:

```java
// ── Optional field validation ──────────────────────────────────────────────

@Test
void version_with_bidi_throws() {
    assertThatThrownBy(() -> new AgentDescriptor(
        "id", "Name", "1.0‮", null, null, null, null,
        null, null, null, "worker", List.of(), null, null, null, "tenant"))
        .isInstanceOf(AgentValidationException.class)
        .satisfies(ex -> assertThat(((AgentValidationException) ex).fieldName())
            .isEqualTo("version"));
}

@Test
void provider_blank_throws() {
    assertThatThrownBy(() -> new AgentDescriptor(
        "id", "Name", null, "  ", null, null, null,
        null, null, null, "worker", List.of(), null, null, null, "tenant"))
        .isInstanceOf(AgentValidationException.class)
        .satisfies(ex -> assertThat(((AgentValidationException) ex).fieldName())
            .isEqualTo("provider"));
}

@Test
void vocabulary_uri_exceeds_500_chars_throws() {
    assertThatThrownBy(() -> new AgentDescriptor(
        "id", "Name", null, null, null, null, null,
        "https://vocab.io/" + "x".repeat(490), null, null,
        "worker", List.of(), null, null, null, "tenant"))
        .isInstanceOf(AgentValidationException.class)
        .satisfies(ex -> assertThat(((AgentValidationException) ex).fieldName())
            .isEqualTo("domainVocabulary"));
}

@Test
void jurisdiction_with_c0_throws() {
    assertThatThrownBy(() -> new AgentDescriptor(
        "id", "Name", null, null, null, null, null,
        null, null, null, "worker", List.of(), null,
        "EUinject", null, "tenant"))
        .isInstanceOf(AgentValidationException.class)
        .satisfies(ex -> assertThat(((AgentValidationException) ex).fieldName())
            .isEqualTo("jurisdiction"));
}

@Test
void data_handling_policy_null_is_allowed() {
    assertThatNoException().isThrownBy(() -> new AgentDescriptor(
        "id", "Name", null, null, null, null, null,
        null, null, null, "worker", List.of(), null, null, null, "tenant"));
}

@Test
void weights_fingerprint_at_255_chars_is_valid() {
    assertThatNoException().isThrownBy(() -> new AgentDescriptor(
        "id", "Name", null, null, null, null, "f".repeat(255),
        null, null, null, "worker", List.of(), null, null, null, "tenant"));
}

@Test
void weights_fingerprint_at_256_chars_throws() {
    assertThatThrownBy(() -> new AgentDescriptor(
        "id", "Name", null, null, null, null, "f".repeat(256),
        null, null, null, "worker", List.of(), null, null, null, "tenant"))
        .isInstanceOf(AgentValidationException.class)
        .satisfies(ex -> assertThat(((AgentValidationException) ex).fieldName())
            .isEqualTo("weightsFingerprint"));
}
```

Add import if missing: `import static org.assertj.core.api.Assertions.assertThat;`

- [ ] **Step 6: Run to confirm new tests fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=AgentDescriptorTest
```

Expected: new tests fail — AgentDescriptor compact constructor doesn't validate optional fields yet.

- [ ] **Step 7: Add optional field validation to AgentDescriptor compact constructor**

```java
public record AgentDescriptor(
        String agentId,
        String name,
        String version,
        String provider,
        String modelFamily,
        String modelVersion,
        String weightsFingerprint,
        String domainVocabulary,
        String slotVocabulary,
        String dispositionVocabulary,
        String slot,
        List<AgentCapability> capabilities,
        AgentDisposition disposition,
        String jurisdiction,
        String dataHandlingPolicy,
        String tenancyId
) {
    public AgentDescriptor {
        AgentDescriptorValidator.validate(agentId, name, slot, tenancyId);
        AgentDescriptorValidator.validateOptional("version",              version,              AgentDescriptorValidator.MAX_VERSION);
        AgentDescriptorValidator.validateOptional("provider",             provider,             AgentDescriptorValidator.MAX_PROVIDER);
        AgentDescriptorValidator.validateOptional("modelFamily",          modelFamily,          AgentDescriptorValidator.MAX_PROVIDER);
        AgentDescriptorValidator.validateOptional("modelVersion",         modelVersion,         AgentDescriptorValidator.MAX_PROVIDER);
        AgentDescriptorValidator.validateOptional("weightsFingerprint",   weightsFingerprint,   AgentDescriptorValidator.MAX_WEIGHTS_FINGERPRINT);
        AgentDescriptorValidator.validateOptional("domainVocabulary",     domainVocabulary,     AgentDescriptorValidator.MAX_VOCABULARY_URI);
        AgentDescriptorValidator.validateOptional("slotVocabulary",       slotVocabulary,       AgentDescriptorValidator.MAX_VOCABULARY_URI);
        AgentDescriptorValidator.validateOptional("dispositionVocabulary",dispositionVocabulary,AgentDescriptorValidator.MAX_VOCABULARY_URI);
        AgentDescriptorValidator.validateOptional("jurisdiction",         jurisdiction,         AgentDescriptorValidator.MAX_JURISDICTION);
        AgentDescriptorValidator.validateOptional("dataHandlingPolicy",   dataHandlingPolicy,   AgentDescriptorValidator.MAX_JURISDICTION);
    }
}
```

- [ ] **Step 8: Run full api tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api
```

Expected: BUILD SUCCESS, all tests pass.

- [ ] **Step 9: Run runtime + examples to check no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime,examples/agent-scenarios
```

Expected: BUILD SUCCESS.

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add api/src/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#20): validate optional string fields on AgentDescriptor and AgentDisposition

Extends AgentDescriptor compact constructor to validate version, provider,
modelFamily, modelVersion, weightsFingerprint (≤255), vocabulary URIs (≤500),
jurisdiction and dataHandlingPolicy (≤1000). AgentDisposition validates its
own axes via a new compact constructor (≤200 each, null-permissive).

Closes #20"
```

---

## Part 3 — Multi-Format Eval (#21)

### Task 7: EvalDimension + EvalReport clean break

**Files:**
- Modify: `eval/src/main/java/io/casehub/eidos/eval/EvalDimension.java`
- Modify: `eval/src/main/java/io/casehub/eidos/eval/EvalReport.java`
- Create: `eval/src/test/java/io/casehub/eidos/eval/EvalReportTest.java`

- [ ] **Step 1: Write EvalReportTest — all failing**

Create `eval/src/test/java/io/casehub/eidos/eval/EvalReportTest.java`:

```java
package io.casehub.eidos.eval;

import io.casehub.eidos.api.*;
import io.casehub.eidos.api.SystemPromptRenderer.RenderFormat;
import io.casehub.eidos.api.SystemPromptRenderer.RenderedPrompt;
import org.junit.jupiter.api.Test;

import java.util.EnumMap;
import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class EvalReportTest {

    static EvalResult resultFor(final RenderFormat format,
                                 final Map<EvalDimension, EvalScore> scores,
                                 final boolean complete) {
        final var desc = new AgentDescriptor(
            "id", "Name", null, null, null, null, null,
            null, null, null, "worker", List.of(), null, null, null, "tenant");
        final var evalCase = new EvalCase("test", desc, AgentPromptContext.forFormat(format));
        final var rendered = new RenderedPrompt("content", format, "dh", "ch");
        final double overall = scores.values().stream()
            .mapToInt(EvalScore::score).average().orElse(0.0);
        return new EvalResult(evalCase, rendered, complete, List.of(), scores, overall, List.of());
    }

    static Map<EvalDimension, EvalScore> proseScores() {
        final Map<EvalDimension, EvalScore> scores = new EnumMap<>(EvalDimension.class);
        scores.put(EvalDimension.SECOND_PERSON,    new EvalScore(4, "ok"));
        scores.put(EvalDimension.CONCISENESS,      new EvalScore(3, "ok"));
        scores.put(EvalDimension.FACTUAL_FIDELITY, new EvalScore(5, "ok"));
        scores.put(EvalDimension.TONE,             new EvalScore(4, "ok"));
        return scores;
    }

    static Map<EvalDimension, EvalScore> a2aScores() {
        final Map<EvalDimension, EvalScore> scores = new EnumMap<>(EvalDimension.class);
        scores.put(EvalDimension.COMPLETENESS,     new EvalScore(5, "ok"));
        scores.put(EvalDimension.FACTUAL_FIDELITY, new EvalScore(4, "ok"));
        return scores;
    }

    @Test
    void build_groups_results_by_format() {
        final List<EvalResult> results = List.of(
            resultFor(RenderFormat.MARKDOWN, proseScores(), true),
            resultFor(RenderFormat.MARKDOWN, proseScores(), true),
            resultFor(RenderFormat.A2A_CARD, a2aScores(),  true));

        final var report = EvalReport.build(results, "test-judge");

        assertThat(report.resultsByFormat()).containsOnlyKeys(RenderFormat.MARKDOWN, RenderFormat.A2A_CARD);
        assertThat(report.resultsByFormat().get(RenderFormat.MARKDOWN)).hasSize(2);
        assertThat(report.resultsByFormat().get(RenderFormat.A2A_CARD)).hasSize(1);
    }

    @Test
    void a2a_summary_has_exactly_two_dimensions() {
        final var report = EvalReport.build(
            List.of(resultFor(RenderFormat.A2A_CARD, a2aScores(), true)), "judge");

        final var summary = report.summaryByFormat().get(RenderFormat.A2A_CARD);
        assertThat(summary.meanByDimension())
            .containsOnlyKeys(EvalDimension.COMPLETENESS, EvalDimension.FACTUAL_FIDELITY);
    }

    @Test
    void markdown_summary_has_four_dimensions() {
        final var report = EvalReport.build(
            List.of(resultFor(RenderFormat.MARKDOWN, proseScores(), true)), "judge");

        final var summary = report.summaryByFormat().get(RenderFormat.MARKDOWN);
        assertThat(summary.meanByDimension())
            .containsOnlyKeys(EvalDimension.SECOND_PERSON, EvalDimension.CONCISENESS,
                              EvalDimension.FACTUAL_FIDELITY, EvalDimension.TONE);
    }

    @Test
    void completeness_is_per_format_independent() {
        final Map<EvalDimension, EvalScore> scores = proseScores();
        final List<EvalResult> results = List.of(
            resultFor(RenderFormat.MARKDOWN, scores, true),
            resultFor(RenderFormat.A2A_CARD, a2aScores(), false));

        final var report = EvalReport.build(results, "judge");

        assertThat(report.summaryByFormat().get(RenderFormat.MARKDOWN).allCasesComplete()).isTrue();
        assertThat(report.summaryByFormat().get(RenderFormat.A2A_CARD).allCasesComplete()).isFalse();
    }

    @Test
    void mean_overall_is_average_of_applicable_dimension_scores() {
        final var report = EvalReport.build(
            List.of(resultFor(RenderFormat.A2A_CARD, a2aScores(), true)), "judge");

        // a2aScores: COMPLETENESS=5, FACTUAL_FIDELITY=4 → mean = 4.5
        assertThat(report.summaryByFormat().get(RenderFormat.A2A_CARD).meanOverall())
            .isEqualTo(4.5);
    }
}
```

- [ ] **Step 2: Run to confirm EvalReportTest fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -Dtest=EvalReportTest
```

Expected: compilation failure — `resultsByFormat()`, `summaryByFormat()` don't exist yet; `EvalDimension.COMPLETENESS` doesn't exist.

- [ ] **Step 3: Update EvalDimension.java**

```java
package io.casehub.eidos.eval;

import io.casehub.eidos.api.SystemPromptRenderer.RenderFormat;

import java.util.EnumSet;
import java.util.Set;

public enum EvalDimension {
    SECOND_PERSON,    // "you"/"your" throughout; no third-person — prose formats only
    CONCISENESS,      // no redundancy, no filler — prose formats only
    FACTUAL_FIDELITY, // nothing claimed absent from descriptor + context — all formats
    TONE,             // reads as instructions to an AI agent — prose formats only
    COMPLETENESS;     // all capabilities present with quality descriptions — A2A_CARD only

    public static Set<EvalDimension> applicableFor(final RenderFormat format) {
        return switch (format) {
            case MARKDOWN, PROSE -> EnumSet.of(SECOND_PERSON, CONCISENESS, FACTUAL_FIDELITY, TONE);
            case A2A_CARD        -> EnumSet.of(COMPLETENESS, FACTUAL_FIDELITY);
        };
    }
}
```

- [ ] **Step 4: Replace EvalReport.java**

```java
package io.casehub.eidos.eval;

import io.casehub.eidos.api.SystemPromptRenderer.RenderFormat;

import java.time.Instant;
import java.util.*;
import java.util.stream.Collectors;

public record EvalReport(
        Instant timestamp,
        String judgeModel,
        Map<RenderFormat, List<EvalResult>> resultsByFormat,
        Map<RenderFormat, EvalSummary> summaryByFormat
) {
    public static EvalReport build(final List<EvalResult> results, final String judgeModel) {
        final Map<RenderFormat, List<EvalResult>> byFormat = results.stream()
            .collect(Collectors.groupingBy(r -> r.evalCase().context().format()));

        final Map<RenderFormat, EvalSummary> summaries = byFormat.entrySet().stream()
            .collect(Collectors.toMap(
                Map.Entry::getKey,
                e -> buildSummary(e.getKey(), e.getValue())));

        return new EvalReport(Instant.now(), judgeModel, byFormat, summaries);
    }

    private static EvalSummary buildSummary(final RenderFormat format,
                                             final List<EvalResult> results) {
        final Set<EvalDimension> applicable = EvalDimension.applicableFor(format);

        final boolean allComplete = results.stream().allMatch(EvalResult::completenessPass);

        final Map<EvalDimension, Double> meanByDim = new EnumMap<>(EvalDimension.class);
        for (final EvalDimension d : applicable) {
            final double mean = results.stream()
                .filter(r -> r.scores().containsKey(d))
                .mapToInt(r -> r.scores().get(d).score())
                .average()
                .orElse(0.0);
            meanByDim.put(d, mean);
        }

        final EvalDimension lowest = meanByDim.entrySet().stream()
            .min(Map.Entry.comparingByValue())
            .map(Map.Entry::getKey)
            .orElse(applicable.iterator().next());

        final double meanOverall = results.stream()
            .mapToDouble(EvalResult::overall)
            .average()
            .orElse(0.0);

        return new EvalSummary(allComplete, meanByDim, lowest, meanOverall);
    }
}
```

- [ ] **Step 5: Run EvalReportTest — confirm pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -Dtest=EvalReportTest
```

Expected: BUILD SUCCESS, all tests pass.

- [ ] **Step 6: Update EvalDatasetTest case count (quick fix)**

In `EvalDatasetTest.java`, find and update:
```java
// Before (whatever asserts the count of 5):
assertThat(EvalDataset.all()).hasSize(5);
// After:
assertThat(EvalDataset.all()).hasSize(9);
```

Also add:
```java
@Test
void all_cases_cover_all_three_formats() {
    final var formats = EvalDataset.all().stream()
        .map(c -> c.context().format())
        .collect(java.util.stream.Collectors.toSet());
    assertThat(formats).containsExactlyInAnyOrder(
        RenderFormat.MARKDOWN, RenderFormat.PROSE, RenderFormat.A2A_CARD);
}
```

(This test will fail until Task 9 adds the new cases — that's expected.)

- [ ] **Step 7: Run eval tests (partial pass expected)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -Dtest=EvalReportTest,EvalDatasetTest
```

Expected: EvalReportTest passes; EvalDatasetTest fails on `hasSize(9)` and format coverage test — that's correct at this stage.

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add eval/src/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#21): EvalDimension adds COMPLETENESS + applicableFor; EvalReport clean break

EvalDimension.applicableFor(RenderFormat) maps prose formats to 4 dimensions,
A2A_CARD to {COMPLETENESS, FACTUAL_FIDELITY}. EvalReport redesigned to
Map<RenderFormat, ...> — no cross-format summary, per-format EvalSummary only.

Refs #21"
```

---

### Task 8: PromptJudge format-aware evaluation

**Files:**
- Modify: `eval/src/main/java/io/casehub/eidos/eval/PromptJudge.java`
- Modify: `eval/src/test/java/io/casehub/eidos/eval/PromptJudgeTest.java`

- [ ] **Step 1: Write new PromptJudgeTest tests**

Add to `PromptJudgeTest.java`:

```java
static final String VALID_A2A_JUDGE_JSON = """
    {
      "COMPLETENESS":     { "score": 5, "reasoning": "All capabilities have descriptions." },
      "FACTUAL_FIDELITY": { "score": 4, "reasoning": "All claims grounded in descriptor." },
      "issues": []
    }""";

// ── A2A evaluation ────────────────────────────────────────────────────────

@Test
void evaluate_a2a_scores_only_completeness_and_factual_fidelity() {
    final ChatModel a2aStub = new ChatModel() {
        @Override
        public ChatResponse doChat(final ChatRequest request) {
            return ChatResponse.builder().aiMessage(AiMessage.from(VALID_A2A_JUDGE_JSON)).build();
        }
    };
    final var desc = new AgentDescriptor(
        "id", "Name", null, null, null, null, null, null, null, null,
        "worker",
        List.of(new AgentCapability("code-review", null, null, null,
            List.of(), List.of(), List.of(), Map.of())),
        null, null, null, "tenant");
    final var a2aCase = new EvalCase("a2a-test", desc,
        AgentPromptContext.forFormat(RenderFormat.A2A_CARD));
    final var a2aRendered = new RenderedPrompt(
        "{\"name\":\"Name\",\"agentId\":\"id\",\"capabilities\":[{\"name\":\"code-review\",\"description\":\"You can review code.\"}]}",
        RenderFormat.A2A_CARD, "dh", "ch");

    final EvalResult result = new PromptJudge(a2aStub, new ObjectMapper()).evaluate(a2aCase, a2aRendered);

    assertThat(result.scores()).containsOnlyKeys(EvalDimension.COMPLETENESS, EvalDimension.FACTUAL_FIDELITY);
    assertThat(result.scores().get(EvalDimension.COMPLETENESS).score()).isEqualTo(5);
    assertThat(result.scores().get(EvalDimension.FACTUAL_FIDELITY).score()).isEqualTo(4);
    assertThat(result.scores()).doesNotContainKey(EvalDimension.SECOND_PERSON);
    assertThat(result.scores()).doesNotContainKey(EvalDimension.TONE);
}

@Test
void evaluate_a2a_completeness_pass_when_all_descriptions_present() {
    final ChatModel stub = new ChatModel() {
        @Override
        public ChatResponse doChat(final ChatRequest request) {
            return ChatResponse.builder().aiMessage(AiMessage.from(VALID_A2A_JUDGE_JSON)).build();
        }
    };
    final var desc = new AgentDescriptor(
        "id", "Name", null, null, null, null, null, null, null, null,
        "worker",
        List.of(new AgentCapability("sprint-planning", null, null, null,
            List.of(), List.of(), List.of(), Map.of())),
        null, null, null, "tenant");
    final var a2aCase = new EvalCase("a2a", desc, AgentPromptContext.forFormat(RenderFormat.A2A_CARD));
    final String cardWithDesc =
        "{\"capabilities\":[{\"name\":\"sprint-planning\",\"description\":\"You plan sprints.\"}]}";
    final var rendered = new RenderedPrompt(cardWithDesc, RenderFormat.A2A_CARD, "dh", "ch");

    final EvalResult result = new PromptJudge(stub, new ObjectMapper()).evaluate(a2aCase, rendered);

    assertThat(result.completenessPass()).isTrue();
    assertThat(result.missingCapabilities()).isEmpty();
}

@Test
void evaluate_a2a_completeness_fail_when_description_absent() {
    final ChatModel stub = new ChatModel() {
        @Override
        public ChatResponse doChat(final ChatRequest request) {
            return ChatResponse.builder().aiMessage(AiMessage.from(VALID_A2A_JUDGE_JSON)).build();
        }
    };
    final var desc = new AgentDescriptor(
        "id", "Name", null, null, null, null, null, null, null, null,
        "worker",
        List.of(new AgentCapability("sprint-planning", null, null, null,
            List.of(), List.of(), List.of(), Map.of())),
        null, null, null, "tenant");
    final var a2aCase = new EvalCase("a2a", desc, AgentPromptContext.forFormat(RenderFormat.A2A_CARD));
    // No description field in the capability object
    final String cardNoDesc = "{\"capabilities\":[{\"name\":\"sprint-planning\"}]}";
    final var rendered = new RenderedPrompt(cardNoDesc, RenderFormat.A2A_CARD, "dh", "ch");

    final EvalResult result = new PromptJudge(stub, new ObjectMapper()).evaluate(a2aCase, rendered);

    assertThat(result.completenessPass()).isFalse();
    assertThat(result.missingCapabilities()).containsExactly("sprint-planning");
}

@Test
void evaluate_a2a_no_capabilities_is_trivially_complete() {
    final ChatModel stub = new ChatModel() {
        @Override
        public ChatResponse doChat(final ChatRequest request) {
            return ChatResponse.builder().aiMessage(AiMessage.from(VALID_A2A_JUDGE_JSON)).build();
        }
    };
    final var desc = new AgentDescriptor(
        "id", "Name", null, null, null, null, null, null, null, null,
        "worker", List.of(), null, null, null, "tenant");
    final var a2aCase = new EvalCase("a2a", desc, AgentPromptContext.forFormat(RenderFormat.A2A_CARD));
    final var rendered = new RenderedPrompt(
        "{\"name\":\"Name\",\"agentId\":\"id\"}", RenderFormat.A2A_CARD, "dh", "ch");

    final EvalResult result = new PromptJudge(stub, new ObjectMapper()).evaluate(a2aCase, rendered);

    assertThat(result.completenessPass()).isTrue();
    assertThat(result.missingCapabilities()).isEmpty();
}
```

Also update the `setUp()` method in existing tests: change `RenderFormat.CLAUDE_MD` → `RenderFormat.MARKDOWN` (2 places in the existing `setUp()` and `rendered` field).

- [ ] **Step 2: Run to confirm new tests fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -Dtest=PromptJudgeTest
```

Expected: new A2A tests fail (scores contain all 4 dimensions, not just 2).

- [ ] **Step 3: Replace PromptJudge.java**

```java
package io.casehub.eidos.eval;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import dev.langchain4j.data.message.SystemMessage;
import dev.langchain4j.data.message.UserMessage;
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.model.chat.request.ChatRequest;
import dev.langchain4j.model.chat.request.ResponseFormat;
import dev.langchain4j.model.chat.request.ResponseFormatType;
import dev.langchain4j.model.chat.request.json.*;
import io.casehub.eidos.api.AgentCapability;
import io.casehub.eidos.api.SystemPromptRenderer.RenderFormat;
import io.casehub.eidos.api.SystemPromptRenderer.RenderedPrompt;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Any;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;

import java.util.*;

@ApplicationScoped
public class PromptJudge {

    static final String MARKDOWN_SYSTEM_PROMPT = """
        You are evaluating the quality of an AI agent's system prompt in markdown format.

        Given the agent's descriptor (JSON) and the rendered system prompt text,
        score each dimension from 0 to 5 and provide brief reasoning.

        Dimensions:
        - SECOND_PERSON (0-5): Uses "you"/"your" consistently. 5 = every sentence second person.
        - CONCISENESS (0-5): Every sentence carries unique information. No filler.
          5 = dense and efficient. 0 = heavily padded.
        - FACTUAL_FIDELITY (0-5): Nothing claimed absent from descriptor or context.
          5 = every claim grounded. 0 = significant hallucinations.
        - TONE (0-5): Reads as instructions to an AI agent, not documentation about one.
          5 = imperative, action-oriented. 0 = reads like a bio or README.

        Return JSON with keys SECOND_PERSON, CONCISENESS, FACTUAL_FIDELITY, TONE
        (each with "score" int 0-5 and "reasoning" string) and an "issues" string array.
        """;

    static final String PROSE_SYSTEM_PROMPT = """
        You are evaluating the quality of an AI agent's system prompt in dense prose format.

        Given the agent's descriptor (JSON) and the rendered system prompt text,
        score each dimension from 0 to 5 and provide brief reasoning.

        Dimensions:
        - SECOND_PERSON (0-5): Uses "you"/"your" consistently. 5 = every sentence second person.
        - CONCISENESS (0-5): Dense prose required — no markdown headers or bullets.
          5 = dense, efficient prose with no markdown. 2 or less if any markdown header (#)
          or bullet point (- or *) is found anywhere in the text.
        - FACTUAL_FIDELITY (0-5): Nothing claimed absent from descriptor or context.
          5 = every claim grounded. 0 = significant hallucinations.
        - TONE (0-5): Reads as instructions to an AI agent, not documentation about one.
          5 = imperative, action-oriented. 0 = reads like a bio or README.

        Return JSON with keys SECOND_PERSON, CONCISENESS, FACTUAL_FIDELITY, TONE
        (each with "score" int 0-5 and "reasoning" string) and an "issues" string array.
        """;

    static final String A2A_SYSTEM_PROMPT = """
        You are evaluating the quality of an AI agent's A2A (agent-to-agent) identity card in JSON format.

        Given the agent's descriptor (JSON) and the rendered A2A card (JSON),
        score each dimension from 0 to 5 and provide brief reasoning.

        Dimensions:
        - COMPLETENESS (0-5): Every declared capability has a non-empty "description" field
          that meaningfully describes what the agent can do with that capability.
          5 = all capabilities have clear, accurate descriptions. 0 = no descriptions.
        - FACTUAL_FIDELITY (0-5): Nothing in the card is absent from the descriptor.
          No hallucinated capabilities, no fabricated names or versions.
          5 = every field grounded in descriptor data. 0 = significant hallucinations.

        Return JSON with keys COMPLETENESS, FACTUAL_FIDELITY
        (each with "score" int 0-5 and "reasoning" string) and an "issues" string array.
        """;

    static final ResponseFormat STANDARD_JUDGE_RESPONSE_FORMAT = ResponseFormat.builder()
        .type(ResponseFormatType.JSON)
        .jsonSchema(JsonSchema.builder()
            .name("EvalJudgment")
            .rootElement(JsonObjectSchema.builder()
                .addProperty("SECOND_PERSON",    scoreSchema())
                .addProperty("CONCISENESS",      scoreSchema())
                .addProperty("FACTUAL_FIDELITY", scoreSchema())
                .addProperty("TONE",             scoreSchema())
                .addProperty("issues", JsonArraySchema.builder()
                    .description("Quality issues found.")
                    .items(JsonStringSchema.builder().build())
                    .build())
                .required("SECOND_PERSON", "CONCISENESS", "FACTUAL_FIDELITY", "TONE", "issues")
                .build())
            .build())
        .build();

    static final ResponseFormat A2A_JUDGE_RESPONSE_FORMAT = ResponseFormat.builder()
        .type(ResponseFormatType.JSON)
        .jsonSchema(JsonSchema.builder()
            .name("A2AEvalJudgment")
            .rootElement(JsonObjectSchema.builder()
                .addProperty("COMPLETENESS",    scoreSchema())
                .addProperty("FACTUAL_FIDELITY", scoreSchema())
                .addProperty("issues", JsonArraySchema.builder()
                    .description("Quality issues found.")
                    .items(JsonStringSchema.builder().build())
                    .build())
                .required("COMPLETENESS", "FACTUAL_FIDELITY", "issues")
                .build())
            .build())
        .build();

    private static JsonObjectSchema scoreSchema() {
        return JsonObjectSchema.builder()
            .addIntegerProperty("score", "Score 0-5")
            .addStringProperty("reasoning", "Brief reasoning")
            .required("score", "reasoning")
            .build();
    }

    private final ChatModel judgeModel;
    private final ObjectMapper mapper;

    @Inject
    public PromptJudge(@Any final Instance<ChatModel> judgeModelInstance,
                       final ObjectMapper mapper) {
        if (!judgeModelInstance.isResolvable()) {
            throw new IllegalStateException(
                "Judge ChatModel not configured. Add a LangChain4j provider to eval/pom.xml " +
                "and configure credentials in application-eval.properties.");
        }
        this.judgeModel = judgeModelInstance.get();
        this.mapper = mapper;
    }

    PromptJudge(final ChatModel judgeModel, final ObjectMapper mapper) {
        this.judgeModel = judgeModel;
        this.mapper = mapper;
    }

    public EvalResult evaluate(final EvalCase evalCase, final RenderedPrompt rendered) {
        final RenderFormat format = evalCase.context().format();
        final Set<EvalDimension> applicable = EvalDimension.applicableFor(format);

        // 1. Format-aware completeness check (deterministic — not sent to LLM)
        final List<String> missing = computeMissingCapabilities(evalCase, rendered);
        final boolean complete = missing.isEmpty();

        // 2. Build judge payload
        final ObjectNode userPayload = mapper.createObjectNode();
        try {
            userPayload.set("descriptor", mapper.valueToTree(evalCase.descriptor()));
            userPayload.put("rendered", rendered.content());
        } catch (final Exception e) {
            throw new IllegalStateException("Failed to build judge payload", e);
        }

        // 3. Select system prompt and schema
        final String systemPrompt = switch (format) {
            case MARKDOWN  -> MARKDOWN_SYSTEM_PROMPT;
            case PROSE     -> PROSE_SYSTEM_PROMPT;
            case A2A_CARD  -> A2A_SYSTEM_PROMPT;
        };
        final ResponseFormat responseFormat = switch (format) {
            case MARKDOWN, PROSE -> STANDARD_JUDGE_RESPONSE_FORMAT;
            case A2A_CARD        -> A2A_JUDGE_RESPONSE_FORMAT;
        };

        // 4. Judge LLM call
        final Map<EvalDimension, EvalScore> scores;
        final List<String> issues;
        try {
            final var request = ChatRequest.builder()
                .messages(
                    SystemMessage.from(systemPrompt),
                    UserMessage.from(mapper.writeValueAsString(userPayload)))
                .responseFormat(responseFormat)
                .build();
            final var response = judgeModel.chat(request);
            final var parsed = parseResponse(response.aiMessage().text(), applicable);
            scores = parsed.scores();
            issues = parsed.issues();
        } catch (final Exception e) {
            throw new IllegalStateException("Judge LLM call failed — check judge model configuration", e);
        }

        final double overall = scores.values().stream()
            .mapToInt(EvalScore::score)
            .average()
            .orElse(0.0);

        return new EvalResult(evalCase, rendered, complete, missing, scores, overall, issues);
    }

    private List<String> computeMissingCapabilities(final EvalCase evalCase,
                                                      final RenderedPrompt rendered) {
        if (evalCase.context().format() == RenderFormat.A2A_CARD) {
            return computeA2aMissingDescriptions(evalCase, rendered.content());
        }
        return evalCase.descriptor().capabilities().stream()
            .map(AgentCapability::name)
            .filter(n -> !rendered.content().contains(n))
            .toList();
    }

    private List<String> computeA2aMissingDescriptions(final EvalCase evalCase,
                                                         final String json) {
        try {
            final JsonNode root = mapper.readTree(json);
            final JsonNode caps = root.get("capabilities");
            if (caps == null || !caps.isArray()) {
                return evalCase.descriptor().capabilities().isEmpty()
                    ? List.of()
                    : evalCase.descriptor().capabilities().stream()
                        .map(AgentCapability::name).toList();
            }
            final Map<String, String> descriptions = new HashMap<>();
            caps.forEach(cap -> {
                final JsonNode name = cap.get("name");
                final JsonNode desc = cap.get("description");
                if (name != null) {
                    descriptions.put(name.asText(), desc != null ? desc.asText() : "");
                }
            });
            return evalCase.descriptor().capabilities().stream()
                .map(AgentCapability::name)
                .filter(n -> {
                    final String desc = descriptions.get(n);
                    return desc == null || desc.isBlank();
                })
                .toList();
        } catch (final Exception e) {
            throw new IllegalStateException("Failed to parse A2A card JSON for completeness check", e);
        }
    }

    private record ParsedResponse(Map<EvalDimension, EvalScore> scores, List<String> issues) {}

    private ParsedResponse parseResponse(final String json,
                                          final Set<EvalDimension> applicable)
            throws JsonProcessingException {
        final JsonNode root = mapper.readTree(json);
        final Map<EvalDimension, EvalScore> scores = new EnumMap<>(EvalDimension.class);

        // Iterate applicable dimensions only — extra keys in response are ignored
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

        final List<String> issues = new ArrayList<>();
        final JsonNode issuesNode = root.get("issues");
        if (issuesNode != null && issuesNode.isArray()) {
            issuesNode.forEach(n -> issues.add(n.asText()));
        }
        return new ParsedResponse(scores, issues);
    }
}
```

- [ ] **Step 4: Run all PromptJudgeTest tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -Dtest=PromptJudgeTest
```

Expected: BUILD SUCCESS, all tests pass (existing + new A2A tests).

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add eval/src/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#21): PromptJudge format-aware evaluation

Three system prompt constants (MARKDOWN, PROSE, A2A). Two judge schemas
(4-dimension prose, 2-dimension A2A). Format-aware completenessPass: prose
uses substring check, A2A parses JSON and checks description field presence.
parseResponse iterates applicable dimensions only.

Refs #21"
```

---

### Task 9: EvalDataset new cases + EvalReportWriter update

**Files:**
- Modify: `eval/src/main/java/io/casehub/eidos/eval/EvalDataset.java`
- Modify: `eval/src/main/java/io/casehub/eidos/eval/EvalReportWriter.java`
- Modify: `eval/src/test/java/io/casehub/eidos/eval/EvalDatasetTest.java`
- Modify: `eval/src/test/java/io/casehub/eidos/eval/PromptEvalTest.java`

- [ ] **Step 1: Update EvalDataset.java**

Replace `EvalDataset.java` with the full updated version. Key changes:
1. All `RenderFormat.CLAUDE_MD` → `RenderFormat.MARKDOWN`
2. Add 4 new private factory methods
3. Update `all()` to return 9 cases

```java
package io.casehub.eidos.eval;

import io.casehub.eidos.api.*;
import io.casehub.eidos.api.SystemPromptRenderer.RenderFormat;

import java.util.List;
import java.util.Map;

public class EvalDataset {

    public static List<EvalCase> all() {
        return List.of(
            // MARKDOWN (5 existing)
            devtownPlanner(), crossVocab(), epistemicWeak(), minimal(), maximal(),
            // PROSE (2 new)
            devtownPlannerProse(), maximalProse(),
            // A2A_CARD (2 new)
            devtownPlannerA2a(), minimalA2a()
        );
    }

    // ── MARKDOWN cases (existing — format reference updated) ──────────────────

    private static EvalCase devtownPlanner() {
        final var descriptor = new AgentDescriptor(
            "planner-1", "Devtown Planner", "1.0", "anthropic",
            "claude", "claude-3-5-sonnet", null,
            "https://vocab.casehub.io/devtown",
            "https://vocab.casehub.io/devtown", null,
            "planner",
            List.of(
                new AgentCapability("sprint-planning", 0.9, 200L, "medium",
                    List.of("backlog", "team-capacity"), List.of("sprint-plan"), List.of(),
                    Map.of("agile", 0.9, "kanban", 0.7)),
                new AgentCapability("estimation", 0.8, 100L, "low",
                    List.of("user-story"), List.of("story-points"), List.of(),
                    Map.of("agile", 0.85))
            ),
            new AgentDisposition("collaborative", "adaptive", "moderate", "assisted", true),
            null, null, "devtown-1"
        );
        final var context = AgentPromptContext.forFormat(RenderFormat.MARKDOWN)
            .withGoal(new GoalContext("Plan sprint 42",
                List.of("Prioritise backlog", "Assign capacity"), "case-sprint-42"));
        return new EvalCase("devtown-planner", descriptor, context);
    }

    private static EvalCase crossVocab() {
        final var descriptor = new AgentDescriptor(
            "reviewer-1", "Code Reviewer", "2.0", "anthropic",
            "claude", "claude-3-7-sonnet", null,
            "https://vocab.casehub.io/svo",
            "https://vocab.casehub.io/devtown", null,
            "reviewer",
            List.of(new AgentCapability("code-review", 0.95, 150L, "low",
                List.of("pull-request"), List.of("review-comment"), List.of(),
                Map.of("java", 0.95, "rust", 0.4, "python", 0.7))),
            new AgentDisposition("independent", "strict", "conservative", "directed", false),
            "EU", "gdpr-compliant", "devtown-1"
        );
        return new EvalCase("cross-vocab", descriptor, AgentPromptContext.forFormat(RenderFormat.MARKDOWN));
    }

    private static EvalCase epistemicWeak() {
        final var descriptor = new AgentDescriptor(
            "ml-agent-1", "ML Researcher", "1.0", "openai",
            "gpt", "gpt-4o", null, null, null, null,
            "researcher",
            List.of(new AgentCapability("literature-review", 0.6, 500L, "high",
                List.of("papers"), List.of("summary"), List.of(),
                Map.of("reinforcement-learning", 0.25, "supervised-learning", 0.8))),
            null, null, null, "research-1"
        );
        final var context = AgentPromptContext.forFormat(RenderFormat.MARKDOWN)
            .withSituationalContext("Reviewing recent RL papers for quarterly report");
        return new EvalCase("epistemic-weak", descriptor, context);
    }

    private static EvalCase minimal() {
        final var descriptor = new AgentDescriptor(
            "min-1", "Minimal Agent", null, null, null, null, null,
            null, null, null, "worker", List.of(), null, null, null, "tenant-1"
        );
        return new EvalCase("minimal", descriptor, AgentPromptContext.forFormat(RenderFormat.MARKDOWN));
    }

    private static EvalCase maximal() {
        final var descriptor = new AgentDescriptor(
            "max-agent-001", "Maximal Agent", "3.1.4", "anthropic",
            "claude", "claude-opus-4-7", "fp-abc123def456",
            "https://vocab.casehub.io/svo",
            "https://vocab.casehub.io/casehub-slot",
            "https://vocab.casehub.io/conscientiousness",
            "orchestrator",
            List.of(
                new AgentCapability("planning", 0.95, 200L, "medium",
                    List.of("goal", "constraints"), List.of("plan", "timeline"), List.of("urgent"),
                    Map.of("strategy", 0.9, "operations", 0.8)),
                new AgentCapability("delegation", 0.85, 50L, "low",
                    List.of("task-spec"), List.of("assignment"), List.of(),
                    Map.of("team-management", 0.75))
            ),
            new AgentDisposition("collaborative", "adaptive", "moderate", "autonomous", true),
            "US", "hipaa-compliant", "enterprise-1"
        );
        final var context = AgentPromptContext.forFormat(RenderFormat.MARKDOWN)
            .withGoal(new GoalContext("Coordinate quarterly planning cycle",
                List.of("Gather input from all teams", "Synthesise priorities", "Produce roadmap"),
                "case-q3-planning"))
            .withResources(List.of(
                new Resource("https://internal.company.io/roadmap", "Current Roadmap", "document"),
                new Resource("https://internal.company.io/okrs", "OKRs", "spreadsheet")))
            .withSituationalContext("End of Q2 — all teams must submit priorities by Friday.");
        return new EvalCase("maximal", descriptor, context);
    }

    // ── PROSE cases (new) ──────────────────────────────────────────────────────

    private static EvalCase devtownPlannerProse() {
        final var descriptor = new AgentDescriptor(
            "planner-1", "Devtown Planner", "1.0", "anthropic",
            "claude", "claude-3-5-sonnet", null,
            "https://vocab.casehub.io/devtown",
            "https://vocab.casehub.io/devtown", null,
            "planner",
            List.of(
                new AgentCapability("sprint-planning", 0.9, 200L, "medium",
                    List.of("backlog", "team-capacity"), List.of("sprint-plan"), List.of(),
                    Map.of("agile", 0.9, "kanban", 0.7)),
                new AgentCapability("estimation", 0.8, 100L, "low",
                    List.of("user-story"), List.of("story-points"), List.of(),
                    Map.of("agile", 0.85))
            ),
            new AgentDisposition("collaborative", "adaptive", "moderate", "assisted", true),
            null, null, "devtown-1"
        );
        final var context = AgentPromptContext.forFormat(RenderFormat.PROSE)
            .withGoal(new GoalContext("Plan sprint 42",
                List.of("Prioritise backlog", "Assign capacity"), "case-sprint-42"));
        return new EvalCase("devtown-planner-prose", descriptor, context);
    }

    private static EvalCase maximalProse() {
        final var descriptor = new AgentDescriptor(
            "max-agent-001", "Maximal Agent", "3.1.4", "anthropic",
            "claude", "claude-opus-4-7", "fp-abc123def456",
            "https://vocab.casehub.io/svo",
            "https://vocab.casehub.io/casehub-slot",
            "https://vocab.casehub.io/conscientiousness",
            "orchestrator",
            List.of(
                new AgentCapability("planning", 0.95, 200L, "medium",
                    List.of("goal", "constraints"), List.of("plan", "timeline"), List.of("urgent"),
                    Map.of("strategy", 0.9, "operations", 0.8)),
                new AgentCapability("delegation", 0.85, 50L, "low",
                    List.of("task-spec"), List.of("assignment"), List.of(),
                    Map.of("team-management", 0.75))
            ),
            new AgentDisposition("collaborative", "adaptive", "moderate", "autonomous", true),
            "US", "hipaa-compliant", "enterprise-1"
        );
        final var context = AgentPromptContext.forFormat(RenderFormat.PROSE)
            .withGoal(new GoalContext("Coordinate quarterly planning cycle",
                List.of("Gather input from all teams", "Synthesise priorities", "Produce roadmap"),
                "case-q3-planning"))
            .withResources(List.of(
                new Resource("https://internal.company.io/roadmap", "Current Roadmap", "document"),
                new Resource("https://internal.company.io/okrs", "OKRs", "spreadsheet")))
            .withSituationalContext("End of Q2 — all teams must submit priorities by Friday.");
        return new EvalCase("maximal-prose", descriptor, context);
    }

    // ── A2A_CARD cases (new) ──────────────────────────────────────────────────

    private static EvalCase devtownPlannerA2a() {
        final var descriptor = new AgentDescriptor(
            "planner-1", "Devtown Planner", "1.0", "anthropic",
            "claude", "claude-3-5-sonnet", null,
            "https://vocab.casehub.io/devtown",
            "https://vocab.casehub.io/devtown", null,
            "planner",
            List.of(
                new AgentCapability("sprint-planning", 0.9, 200L, "medium",
                    List.of("backlog", "team-capacity"), List.of("sprint-plan"), List.of(),
                    Map.of("agile", 0.9, "kanban", 0.7)),
                new AgentCapability("estimation", 0.8, 100L, "low",
                    List.of("user-story"), List.of("story-points"), List.of(),
                    Map.of("agile", 0.85))
            ),
            new AgentDisposition("collaborative", "adaptive", "moderate", "assisted", true),
            null, null, "devtown-1"
        );
        return new EvalCase("devtown-planner-a2a", descriptor,
            AgentPromptContext.forFormat(RenderFormat.A2A_CARD));
    }

    private static EvalCase minimalA2a() {
        final var descriptor = new AgentDescriptor(
            "min-1", "Minimal Agent", null, null, null, null, null,
            null, null, null, "worker", List.of(), null, null, null, "tenant-1"
        );
        return new EvalCase("minimal-a2a", descriptor,
            AgentPromptContext.forFormat(RenderFormat.A2A_CARD));
    }
}
```

- [ ] **Step 2: Update EvalReportWriter.java**

Replace `summaryTable()`:

```java
public static String summaryTable(final EvalReport report) {
    final var sb = new StringBuilder();

    report.summaryByFormat().forEach((format, summary) -> {
        final int count = report.resultsByFormat().getOrDefault(format, List.of()).size();
        sb.append(String.format("=== %s (%d case%s) ===%n",
            format.name(), count, count == 1 ? "" : "s"));
        sb.append(String.format("%-30s %5s%n", "Dimension", "Score"));
        sb.append("-".repeat(37)).append("\n");

        summary.meanByDimension().entrySet().stream()
            .sorted(Comparator.comparingDouble(Map.Entry<EvalDimension, Double>::getValue)
                               .thenComparingInt(e -> e.getKey().ordinal()))
            .forEach(e -> sb.append(String.format("%-30s %5.2f%n",
                e.getKey().name().toLowerCase().replace('_', ' '), e.getValue())));

        sb.append("\n");
        sb.append(String.format("Overall mean:          %5.2f / 5.0%n", summary.meanOverall()));
        sb.append(String.format("All cases complete:    %s%n",
            summary.allCasesComplete() ? "YES" : "NO"));
        sb.append(String.format("Lowest dimension:      %s%n",
            summary.lowestScoringDimension().name().toLowerCase().replace('_', ' ')));
        sb.append("\n");
    });

    return sb.toString();
}
```

Add import at top: `import java.util.List;`

- [ ] **Step 3: Update EvalReportWriterTest**

`EvalReportWriterTest` builds an `EvalReport` using the old shape — it will fail to compile. Read the file first (`cat eval/src/test/java/io/casehub/eidos/eval/EvalReportWriterTest.java`), then replace its `report` construction with `EvalReport.build(List.of(result), "judge-model")` where `result` is an `EvalResult` built using the helper from `EvalReportTest` (copy `resultFor` inline or import it). The assertions on `summaryTable()` output should change to check for the `=== MARKDOWN` format header and per-format dimension lines.

- [ ] **Step 4: Update EvalDatasetTest**

```java
// Update case count:
assertThat(EvalDataset.all()).hasSize(9);

// Update format coverage test (already added in Task 7, now should pass):
@Test
void all_cases_cover_all_three_formats() {
    final var formats = EvalDataset.all().stream()
        .map(c -> c.context().format())
        .collect(java.util.stream.Collectors.toSet());
    assertThat(formats).containsExactlyInAnyOrder(
        RenderFormat.MARKDOWN, RenderFormat.PROSE, RenderFormat.A2A_CARD);
}
```

- [ ] **Step 4: Update PromptEvalTest**

```java
@QuarkusTest
@TestProfile(EvalProfile.class)
@Tag("eval")
class PromptEvalTest {

    private static final Map<RenderFormat, Double> SCORE_FLOORS = Map.of(
        RenderFormat.MARKDOWN,  3.5,   // calibrate after first run
        RenderFormat.PROSE,     3.5,   // calibrate after first run
        RenderFormat.A2A_CARD,  3.5    // calibrate after first run
    );

    @Inject SystemPromptRenderer renderer;
    @Inject PromptJudge judge;

    @Test
    void evaluateAllScenarios() throws Exception {
        final List<EvalResult> results = EvalDataset.all().stream()
            .map(c -> judge.evaluate(c, renderer.render(c.descriptor(), c.context())))
            .toList();

        final EvalReport report = EvalReport.build(results, "judge");
        final Path outPath = Path.of("target/eval-report.json");
        Files.createDirectories(outPath.getParent());
        EvalReportWriter.writeJson(report, outPath);
        System.out.println(EvalReportWriter.summaryTable(report));

        report.summaryByFormat().forEach((format, summary) -> {
            assertThat(summary.allCasesComplete())
                .as("All %s cases must include every declared capability", format)
                .isTrue();
            assertThat(summary.meanOverall())
                .as("Mean judge score for %s", format)
                .isGreaterThanOrEqualTo(SCORE_FLOORS.get(format));
        });
    }
}
```

Add import: `import java.util.Map;`

- [ ] **Step 5: Run all eval unit tests (excluding @Tag("eval"))**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval
```

Expected: BUILD SUCCESS, all tests pass (PromptEvalTest skipped — no LLM configured for unit tests).

- [ ] **Step 6: Run full build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install
```

Expected: BUILD SUCCESS across all modules.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add eval/src/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#21): multi-format EvalDataset (PROSE + A2A_CARD cases), format-grouped EvalReportWriter, per-format PromptEvalTest floors

9 eval cases: 5 MARKDOWN (existing), 2 PROSE, 2 A2A_CARD.
EvalReportWriter.summaryTable() groups output by format with case count.
PromptEvalTest asserts per-format completeness and score floor.

Closes #21"
```

---

## Final Step — Promote spec to project repo

- [ ] **Step 1: Copy spec to project repo**

```bash
mkdir -p /Users/mdproctor/claude/casehub/eidos/docs/superpowers/specs
cp /Users/mdproctor/claude/public/casehub/eidos/specs/2026-05-31-multi-format-eval-validation-design.md \
   /Users/mdproctor/claude/casehub/eidos/docs/superpowers/specs/
```

- [ ] **Step 2: Commit spec to project repo**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add docs/superpowers/specs/2026-05-31-multi-format-eval-validation-design.md
git -C /Users/mdproctor/claude/casehub/eidos commit -m "docs: promote design spec for eidos#21, #20, #22

Refs #21, #20, #22"
```
