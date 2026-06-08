# Framework Grounding Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ground AgentDescriptor disposition axes and slot in their theoretical vocabulary frameworks so the LLM renderer produces Belbin-, DISC-, and Thomas-Kilmann-canonical language.

**Architecture:** Add `description()` to `@VocabularyMetadata`, expose a `vocabularyMetadata(uri)` lookup on `VocabularyRegistry`, and enrich `EidosRenderPipeline` to resolve disposition axes through the vocab registry — building per-axis nested objects with label and vocabulary context. Fix `conflictMode` omission in payload and structural renderers. Update the LLM prompt template to use framework canonical language. All six vocab enums gain empirical-basis descriptions.

**Tech Stack:** Java 21, Quarkus 3.32.2, Jackson, JUnit 5, AssertJ, `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn`

**Spec:** `docs/superpowers/specs/2026-06-08-framework-grounding-design.md`

---

## File Map

| Action | File |
|--------|------|
| Modify | `api/src/main/java/io/casehub/eidos/api/VocabularyMetadata.java` |
| Modify | `api/src/main/java/io/casehub/eidos/api/VocabularyRegistry.java` |
| Modify | `runtime/src/main/java/io/casehub/eidos/runtime/vocabulary/CdiVocabularyRegistry.java` |
| Modify | `runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosRenderPipeline.java` |
| Modify | `runtime/src/test/java/io/casehub/eidos/runtime/vocabulary/CdiVocabularyRegistryTest.java` |
| Modify | `runtime/src/test/java/io/casehub/eidos/runtime/renderer/EidosRenderPipelineTest.java` |
| Modify | `vocab/src/main/java/io/casehub/eidos/vocab/SvoTerm.java` |
| Modify | `vocab/src/main/java/io/casehub/eidos/vocab/ConscientiousnessTerm.java` |
| Modify | `vocab/src/main/java/io/casehub/eidos/vocab/CasehubSlotTerm.java` |
| Modify | `vocab/src/main/java/io/casehub/eidos/vocab/BelbinTerm.java` |
| Modify | `vocab/src/main/java/io/casehub/eidos/vocab/DiscTerm.java` |
| Modify | `vocab/src/main/java/io/casehub/eidos/vocab/ThomasKilmannTerm.java` |
| Modify | `examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/SystemPromptRendererTest.java` |

---

## Task 1: Add `description()` to `@VocabularyMetadata`

**Files:**
- Modify: `api/src/main/java/io/casehub/eidos/api/VocabularyMetadata.java`

This is a pure API change. All downstream tasks depend on it being in place first.

- [ ] **Step 1: Add `description()` attribute**

Replace the entire file content:

```java
package io.casehub.eidos.api;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Vocabulary-level metadata for an enum implementing {@link VocabularyTerm}.
 * {@code name()}, {@code version()}, and {@code description()} default to {@code ""}
 * meaning "not provided"; callers should treat {@code isEmpty()} as absent.
 */
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface VocabularyMetadata {
    String uri();
    String name()        default "";
    String version()     default "";
    String description() default "";
}
```

- [ ] **Step 2: Compile the api module to verify no errors**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api -q
```

Expected: `BUILD SUCCESS`

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add api/src/main/java/io/casehub/eidos/api/VocabularyMetadata.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#27): add description() to @VocabularyMetadata"
```

---

## Task 2: Add `vocabularyMetadata()` to `VocabularyRegistry` SPI + implement + test

**Files:**
- Modify: `api/src/main/java/io/casehub/eidos/api/VocabularyRegistry.java`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/vocabulary/CdiVocabularyRegistry.java`
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/vocabulary/CdiVocabularyRegistryTest.java`

- [ ] **Step 1: Write failing tests in `CdiVocabularyRegistryTest`**

Add these two tests at the end of the `CdiVocabularyRegistryTest` class (before the closing `}`). The test enum `SourceTerm` (already defined in the class, with `name = "Source Vocab"`) is used in the first test. The second test uses `TargetTerm` (defined with `uri = "urn:test:target"`, no name).

```java
    // --- vocabularyMetadata() tests ---

    @Test
    void vocabularyMetadata_registered_uri_returns_annotation() {
        registry.register(SourceTerm.class);
        var meta = registry.vocabularyMetadata("urn:test:source");
        assertThat(meta).isPresent();
        assertThat(meta.get().uri()).isEqualTo("urn:test:source");
        assertThat(meta.get().name()).isEqualTo("Source Vocab");
    }

    @Test
    void vocabularyMetadata_unregistered_uri_returns_empty() {
        assertThat(registry.vocabularyMetadata("urn:does-not-exist")).isEmpty();
    }
```

- [ ] **Step 2: Run tests — verify they fail (compilation error)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=CdiVocabularyRegistryTest 2>&1 | tail -5
```

Expected: compilation error — `vocabularyMetadata(String)` not found on `VocabularyRegistry`.

- [ ] **Step 3: Add `vocabularyMetadata()` to `VocabularyRegistry` SPI**

Add at the end of `VocabularyRegistry.java`, before the closing `}`:

```java
    // --- Vocabulary-level metadata ---

    /**
     * Returns the vocabulary-level metadata annotation for the given URI.
     * Empty if the URI is not registered.
     */
    Optional<VocabularyMetadata> vocabularyMetadata(String uri);
```

Also add the import at the top of the file (it already imports `Optional`; check whether `VocabularyMetadata` is already in the same package — it is, so no import needed).

- [ ] **Step 4: Implement in `CdiVocabularyRegistry`**

Add this method at the end of `CdiVocabularyRegistry`, before the closing `}`:

```java
    @Override
    public Optional<VocabularyMetadata> vocabularyMetadata(String uri) {
        var clazz = byUri.get(uri);
        if (clazz == null) return Optional.empty();
        // register() guarantees @VocabularyMetadata is present for anything in byUri
        return Optional.of(clazz.getAnnotation(VocabularyMetadata.class));
    }
```

- [ ] **Step 5: Run tests — verify they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=CdiVocabularyRegistryTest 2>&1 | tail -10
```

Expected: `BUILD SUCCESS`, all tests pass.

- [ ] **Step 6: Run full api + runtime test suite to confirm no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api,runtime -q
```

Expected: `BUILD SUCCESS`

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add \
  api/src/main/java/io/casehub/eidos/api/VocabularyRegistry.java \
  runtime/src/main/java/io/casehub/eidos/runtime/vocabulary/CdiVocabularyRegistry.java \
  runtime/src/test/java/io/casehub/eidos/runtime/vocabulary/CdiVocabularyRegistryTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#27): add VocabularyRegistry.vocabularyMetadata(uri) — SPI + impl + tests"
```

---

## Task 3: Add `addIfNonBlank` + fix slot section in pipeline + tests

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosRenderPipeline.java`
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/renderer/EidosRenderPipelineTest.java`

- [ ] **Step 1: Add test vocab enums and update `setUp` in `EidosRenderPipelineTest`**

Add `import java.util.Optional;` to the import block at the top of `EidosRenderPipelineTest`.

Change the `vocab` field and `setUp` method (the existing `pipeline` field is kept; a new `vocab` field is added so tests can register vocabs):

```java
    static final ObjectMapper MAPPER = new ObjectMapper();
    CdiVocabularyRegistry vocab;
    EidosRenderPipeline pipeline;

    @BeforeEach
    void setUp() {
        vocab = new CdiVocabularyRegistry();
        pipeline = new EidosRenderPipeline(vocab, MAPPER);
    }
```

Add these inner test enum declarations inside the `EidosRenderPipelineTest` class (after the imports/class open, before the fields):

```java
    @VocabularyMetadata(uri = "urn:test:disp", name = "Test Disposition Vocab", version = "1.0",
                        description = "A test disposition vocabulary description")
    enum TestDispTerm implements VocabularyTerm {
        INDEPENDENT("independent", "Independent", "Works alone by preference", List.of("alone"));
        private final String value, label, description;
        private final List<String> aliases;
        TestDispTerm(String v, String l, String d, List<String> a) {
            value = v; label = l; description = d; aliases = a;
        }
        @Override public String value()         { return value; }
        @Override public String label()         { return label; }
        @Override public String description()   { return description; }
        @Override public List<String> aliases() { return aliases; }
    }

    @VocabularyMetadata(uri = "urn:test:slot", name = "Test Slot Vocab", version = "1.0",
                        description = "A test slot vocabulary description")
    enum TestSlotTerm implements VocabularyTerm {
        REVIEWER("reviewer", "Reviewer", "Reviews the work", List.of());
        private final String value, label, description;
        private final List<String> aliases;
        TestSlotTerm(String v, String l, String d, List<String> a) {
            value = v; label = l; description = d; aliases = a;
        }
        @Override public String value()         { return value; }
        @Override public String label()         { return label; }
        @Override public String description()   { return description; }
        @Override public List<String> aliases() { return aliases; }
    }

    @VocabularyMetadata(uri = "urn:test:noname")
    enum TestNoNameTerm implements VocabularyTerm {
        TERM("term", "Term", List.of());
        private final String value, label;
        private final List<String> aliases;
        TestNoNameTerm(String v, String l, List<String> a) {
            value = v; label = l; aliases = a;
        }
        @Override public String value()         { return value; }
        @Override public String label()         { return label; }
        @Override public List<String> aliases() { return aliases; }
    }
```

- [ ] **Step 2: Write failing tests for slot section**

Add these tests to `EidosRenderPipelineTest` (any location in the file):

```java
    // ── Slot vocabulary context ───────────────────────────────────────────────

    @Test
    void slot_payload_includes_vocabulary_name_and_description() {
        vocab.register(TestSlotTerm.class);
        var desc = AgentDescriptor.builder()
            .agentId("a").name("N").slotVocabulary("urn:test:slot").slot("reviewer").tenancyId("t").build();
        var node = pipeline.buildDescriptorPayload(desc);
        assertThat(node.get("slotVocabularyName").asText()).isEqualTo("Test Slot Vocab");
        assertThat(node.get("slotVocabularyDescription").asText()).isEqualTo("A test slot vocabulary description");
    }

    @Test
    void empty_vocab_name_not_emitted_in_payload() {
        vocab.register(TestNoNameTerm.class);
        var desc = AgentDescriptor.builder()
            .agentId("a").name("N").slotVocabulary("urn:test:noname").slot("term").tenancyId("t").build();
        var node = pipeline.buildDescriptorPayload(desc);
        // TestNoNameTerm has name="" — addIfNonBlank must suppress the key entirely
        assertThat(node.has("slotVocabularyName")).isFalse();
    }
```

- [ ] **Step 3: Run tests — verify they fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=EidosRenderPipelineTest#slot_payload_includes_vocabulary_name_and_description+empty_vocab_name_not_emitted_in_payload 2>&1 | tail -10
```

Expected: FAIL — `slotVocabularyName` node is null (key absent from current payload).

- [ ] **Step 4: Add `addIfNonBlank` helper to `EidosRenderPipeline`**

Add this method immediately after the existing `addIfPresent` method (around line 497):

```java
    private static void addIfNonBlank(final ObjectNode node, final String key, final String value) {
        if (value != null && !value.isEmpty()) node.put(key, value);
    }
```

Also add these three imports to the import block:

```java
import io.casehub.eidos.api.DispositionAxis;
import io.casehub.eidos.api.VocabularyMetadata;
import io.casehub.eidos.api.VocabularyTerm;
```

- [ ] **Step 5: Fix slot section in `buildDescriptorPayload`**

Locate the existing slot vocabulary block (around line 152):

```java
        // Vocabulary-resolved slot labels
        if (descriptor.slotVocabulary() != null) {
            vocab.resolve(descriptor.slotVocabulary(), descriptor.slot()).ifPresent(term -> {
                addIfPresent(node, "slotLabel", term.label());
                addIfPresent(node, "slotDescription", term.description());
            });
        }
```

Replace it with:

```java
        // Vocabulary-resolved slot labels and vocabulary context
        if (descriptor.slotVocabulary() != null) {
            vocab.resolve(descriptor.slotVocabulary(), descriptor.slot()).ifPresent(term -> {
                addIfPresent(node,  "slotLabel",       term.label());
                addIfNonBlank(node, "slotDescription", term.description());
            });
            vocab.vocabularyMetadata(descriptor.slotVocabulary()).ifPresent(meta -> {
                addIfNonBlank(node, "slotVocabularyName",        meta.name());
                addIfNonBlank(node, "slotVocabularyDescription", meta.description());
            });
        }
```

- [ ] **Step 6: Run tests — verify they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=EidosRenderPipelineTest 2>&1 | tail -10
```

Expected: `BUILD SUCCESS`, all tests pass.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add \
  runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosRenderPipeline.java \
  runtime/src/test/java/io/casehub/eidos/runtime/renderer/EidosRenderPipelineTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#27): addIfNonBlank helper + slot vocabulary context in payload"
```

---

## Task 4: Replace disposition section + add `axisJsonKey` + tests

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosRenderPipeline.java`
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/renderer/EidosRenderPipelineTest.java`

- [ ] **Step 1: Write failing tests for disposition payload**

Add these tests to `EidosRenderPipelineTest`:

```java
    // ── Disposition payload (nested per-axis objects) ─────────────────────────

    @Test
    void disposition_payload_is_nested_object_per_axis() {
        vocab.register(TestDispTerm.class);
        var desc = AgentDescriptor.builder()
            .agentId("a").name("N").slot("s")
            .dispositionVocabulary("urn:test:disp")
            .disposition(AgentDisposition.builder().socialOrient("independent").build())
            .tenancyId("t").build();
        var dispNode = pipeline.buildDescriptorPayload(desc).get("disposition");
        var socialOrient = dispNode.get("socialOrient");
        assertThat(socialOrient.isObject()).isTrue();
        assertThat(socialOrient.get("value").asText()).isEqualTo("independent");
        assertThat(socialOrient.get("label").asText()).isEqualTo("Independent");
        assertThat(socialOrient.get("vocabularyName").asText()).isEqualTo("Test Disposition Vocab");
    }

    @Test
    void conflict_mode_included_in_payload_when_set() {
        var desc = AgentDescriptor.builder()
            .agentId("a").name("N").slot("s")
            .disposition(AgentDisposition.builder().conflictMode("avoiding").build())
            .tenancyId("t").build();
        var dispNode = pipeline.buildDescriptorPayload(desc).get("disposition");
        assertThat(dispNode.has("conflictMode")).isTrue();
        assertThat(dispNode.get("conflictMode").get("value").asText()).isEqualTo("avoiding");
    }

    @Test
    void disposition_without_registered_vocab_has_value_only() {
        var desc = AgentDescriptor.builder()
            .agentId("a").name("N").slot("s")
            .dispositionVocabulary("urn:test:unregistered")
            .disposition(AgentDisposition.builder().socialOrient("custom-value").build())
            .tenancyId("t").build();
        var axisNode = pipeline.buildDescriptorPayload(desc).get("disposition").get("socialOrient");
        assertThat(axisNode.isObject()).isTrue();
        assertThat(axisNode.get("value").asText()).isEqualTo("custom-value");
        assertThat(axisNode.has("label")).isFalse();
        assertThat(axisNode.has("vocabularyName")).isFalse();
    }

    @Test
    void different_disposition_vocab_produces_different_descriptor_hash() {
        vocab.register(TestDispTerm.class);
        var descWithVocab = AgentDescriptor.builder()
            .agentId("a").name("N").slot("s")
            .dispositionVocabulary("urn:test:disp")
            .disposition(AgentDisposition.builder().socialOrient("independent").build())
            .tenancyId("t").build();
        var descWithout = AgentDescriptor.builder()
            .agentId("a").name("N").slot("s")
            .disposition(AgentDisposition.builder().socialOrient("independent").build())
            .tenancyId("t").build();
        var ctx = AgentPromptContext.forFormat(MARKDOWN);
        assertThat(pipeline.buildStage1(descWithVocab, ctx).descriptorHash())
            .isNotEqualTo(pipeline.buildStage1(descWithout, ctx).descriptorHash());
    }
```

- [ ] **Step 2: Run tests — verify they fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=EidosRenderPipelineTest#conflict_mode_included_in_payload_when_set 2>&1 | tail -10
```

Expected: FAIL — `conflictMode` is null (absent from current payload).

- [ ] **Step 3: Add `axisJsonKey` private static helper to `EidosRenderPipeline`**

Add after the `addIfNonBlank` method:

```java
    private static String axisJsonKey(final DispositionAxis axis) {
        return switch (axis) {
            case SOCIAL_ORIENTATION -> "socialOrient";
            case RULE_FOLLOWING     -> "ruleFollowing";
            case RISK_APPETITE      -> "riskAppetite";
            case AUTONOMY           -> "autonomy";
            case CONFLICT_MODE      -> "conflictMode";
        };
    }
```

The exhaustive switch (no `default`) ensures adding a new `DispositionAxis` value causes a compile error here, forcing explicit coverage.

- [ ] **Step 4: Replace the disposition block in `buildDescriptorPayload`**

Locate the existing disposition block (around line 183):

```java
        // Disposition
        if (descriptor.disposition() != null) {
            final AgentDisposition d = descriptor.disposition();
            final ObjectNode dispNode = node.putObject("disposition");
            addIfPresent(dispNode, "socialOrient",   d.socialOrient());
            addIfPresent(dispNode, "ruleFollowing",  d.ruleFollowing());
            addIfPresent(dispNode, "riskAppetite",   d.riskAppetite());
            addIfPresent(dispNode, "autonomy",        d.autonomy());
            dispNode.put("canDelegate", d.delegation());
        }
```

Replace it with:

```java
        // Disposition — per-axis nested objects with vocabulary context
        if (descriptor.disposition() != null) {
            final AgentDisposition d = descriptor.disposition();
            final ObjectNode dispNode = node.putObject("disposition");
            for (DispositionAxis axis : DispositionAxis.values()) {
                d.get(axis).ifPresent(rawValue -> {
                    final ObjectNode axisNode = dispNode.putObject(axisJsonKey(axis));
                    axisNode.put("value", rawValue);
                    descriptor.vocabUriForAxis(axis).ifPresent(uri -> {
                        vocab.resolve(uri, rawValue).ifPresent(term -> {
                            addIfPresent(axisNode,  "label",       term.label());
                            addIfNonBlank(axisNode, "description", term.description());
                        });
                        vocab.vocabularyMetadata(uri).ifPresent(meta -> {
                            addIfNonBlank(axisNode, "vocabularyName",        meta.name());
                            addIfNonBlank(axisNode, "vocabularyDescription", meta.description());
                        });
                    });
                });
            }
            dispNode.put("canDelegate", d.delegation());
        }
```

- [ ] **Step 5: Run tests — verify all pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=EidosRenderPipelineTest 2>&1 | tail -10
```

Expected: `BUILD SUCCESS`, all tests pass.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add \
  runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosRenderPipeline.java \
  runtime/src/test/java/io/casehub/eidos/runtime/renderer/EidosRenderPipelineTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#27): per-axis nested disposition payload with vocabulary context — fixes conflictMode omission"
```

---

## Task 5: Update `PROMPT_TEMPLATE` and `RESPONSE_FORMAT`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosRenderPipeline.java`

No unit tests — the prompt template content is exercised by the integration tests in Task 8. Changing it bumps `TEMPLATE_HASH`, invalidating all existing cache entries (correct behaviour).

- [ ] **Step 1: Replace the `PROMPT_TEMPLATE` text block**

Locate `static final String PROMPT_TEMPLATE = """` (around line 36) and replace the entire text block:

```java
    static final String PROMPT_TEMPLATE = """
            You are writing narrative descriptions for an AI agent's system prompt.

            Given the agent definition in JSON, produce a JSON object with prose descriptions
            for each field. Write in second person, addressing the agent directly.

            REQUIRED FIELDS (always populate):
            - identityNarrative (1-2 sentences): The agent's name, model, and version context.
            - roleNarrative (1-3 sentences): The role this agent plays and its purpose.
              If slotLabel and slotDescription are present, prefer them over the raw slot value.
              If slotVocabularyName is present, use that framework's canonical language —
              e.g., slotVocabularyName "Belbin Team Roles" → open with the Belbin archetype
              framing ("You are the team's Monitor Evaluator...").
            - capabilityNarrative (2-4 sentences): What the agent can do.
              Include inputTypes and outputTypes when present.
              For epistemicDomains, use natural language confidence:
                >= 0.7 -> "strong expertise", 0.4-0.69 -> "working knowledge", < 0.4 -> "limited familiarity".

            OPTIONAL FIELDS (use empty string "" if the source data is absent):
            - dispositionNarrative (2-3 sentences): How the agent operates across all disposition
              axes present in the payload. The disposition object contains one nested object per axis;
              each has a "value" field and optionally "label", "description", and "vocabularyName".
              Cover all axes that have values: socialOrient, ruleFollowing, riskAppetite, autonomy,
              conflictMode. When "vocabularyName" is present, use that framework's canonical language
              rather than generic phrasing — e.g., "vocabularyName: Thomas-Kilmann Conflict Modes"
              → use TKI mode language; "vocabularyName: DISC Behavioral Styles" → use DISC canonical
              phrasing. Include delegation intent if canDelegate is true.
              Use "" if no disposition is present.
            - constraintNarrative (1-2 sentences): Data handling obligations - jurisdiction
              and compliance requirements the agent must observe.
            - goalNarrative (1-3 sentences): The agent's current task and objectives.
              Include sub-goals as a natural continuation, not a bullet list.

            RULES:
            - Second person only: "You are...", "Your role is...", "You have...".
            - Plain prose. No markdown, no bullet points, no headers.
            - Be concise. Every sentence must carry information the agent needs to act on.
            - Return ONLY the JSON object. No explanation, no preamble, no code fences.""";
```

- [ ] **Step 2: Update the `dispositionNarrative` entry in `RESPONSE_FORMAT`**

Locate (around line 78):
```java
                        .addStringProperty("dispositionNarrative",
                                "How the agent operates. Empty string if no disposition data.")
```

Replace with:
```java
                        .addStringProperty("dispositionNarrative",
                                "How the agent operates across all disposition axes in the payload. " +
                                "Each axis object carries value, optional label, optional vocabularyName. " +
                                "Use framework canonical language when vocabularyName is present. " +
                                "2-3 sentences. Empty string if no disposition data.")
```

- [ ] **Step 3: Compile to confirm no errors**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -q
```

Expected: `BUILD SUCCESS`

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add \
  runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosRenderPipeline.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#27): update PROMPT_TEMPLATE and RESPONSE_FORMAT for framework-grounded disposition"
```

---

## Task 6: Fix structural renderers + add `axisLabel` + `resolveAxisDisplay` + tests

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosRenderPipeline.java`
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/renderer/EidosRenderPipelineTest.java`

- [ ] **Step 1: Write failing tests for structural renderers**

Add these tests to `EidosRenderPipelineTest`:

```java
    // ── Structural renderers ─────────────────────────────────────────────────

    @Test
    void structural_markdown_shows_axis_label_not_raw_value() {
        vocab.register(TestDispTerm.class);
        var desc = AgentDescriptor.builder()
            .agentId("a").name("N").slot("s")
            .dispositionVocabulary("urn:test:disp")
            .disposition(AgentDisposition.builder().socialOrient("independent").build())
            .tenancyId("t").build();
        var ctx = AgentPromptContext.forFormat(MARKDOWN);
        var s1 = pipeline.buildStage1(desc, ctx);
        var result = pipeline.assemble(s1, Optional.empty(), Optional.empty(), desc, ctx);
        assertThat(result.content()).contains("Independent (Test Disposition Vocab)");
        assertThat(result.content()).doesNotContain("Social orientation: independent\n");
    }

    @Test
    void structural_markdown_includes_conflict_mode() {
        var desc = AgentDescriptor.builder()
            .agentId("a").name("N").slot("s")
            .disposition(AgentDisposition.builder().conflictMode("avoiding").build())
            .tenancyId("t").build();
        var ctx = AgentPromptContext.forFormat(MARKDOWN);
        var s1 = pipeline.buildStage1(desc, ctx);
        var result = pipeline.assemble(s1, Optional.empty(), Optional.empty(), desc, ctx);
        assertThat(result.content()).contains("Conflict mode: avoiding");
    }

    @Test
    void structural_prose_includes_all_disposition_axes() {
        var desc = AgentDescriptor.builder()
            .agentId("a").name("N").slot("s")
            .disposition(AgentDisposition.builder()
                .socialOrient("independent")
                .ruleFollowing("strict")
                .riskAppetite("conservative")
                .autonomy("directed")
                .conflictMode("avoiding")
                .build())
            .tenancyId("t").build();
        var ctx = AgentPromptContext.forFormat(PROSE);
        var s1 = pipeline.buildStage1(desc, ctx);
        var result = pipeline.assemble(s1, Optional.empty(), Optional.empty(), desc, ctx);
        assertThat(result.content()).contains("Social orientation: independent");
        assertThat(result.content()).contains("Rule following: strict");
        assertThat(result.content()).contains("Risk appetite: conservative");
        assertThat(result.content()).contains("Autonomy: directed");
        assertThat(result.content()).contains("Conflict mode: avoiding");
    }
```

- [ ] **Step 2: Run tests — verify they fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest="EidosRenderPipelineTest#structural_markdown_includes_conflict_mode+structural_prose_includes_all_disposition_axes" \
  2>&1 | tail -10
```

Expected: FAIL — `conflictMode` absent from structural output; `socialOrient` and `riskAppetite` absent from prose output.

- [ ] **Step 3: Add `axisLabel` private static helper to `EidosRenderPipeline`**

Add after `axisJsonKey`:

```java
    private static String axisLabel(final DispositionAxis axis) {
        return switch (axis) {
            case SOCIAL_ORIENTATION -> "Social orientation";
            case RULE_FOLLOWING     -> "Rule following";
            case RISK_APPETITE      -> "Risk appetite";
            case AUTONOMY           -> "Autonomy";
            case CONFLICT_MODE      -> "Conflict mode";
        };
    }
```

- [ ] **Step 4: Add `resolveAxisDisplay` private instance helper**

Add after `axisLabel`:

```java
    private String resolveAxisDisplay(final DispositionAxis axis, final String raw,
                                       final AgentDescriptor descriptor) {
        final Optional<String> vocabUri = descriptor.vocabUriForAxis(axis);
        final String label = vocabUri
            .flatMap(uri -> vocab.resolve(uri, raw))
            .map(VocabularyTerm::label)
            .orElse(raw);
        final String vocabName = vocabUri
            .flatMap(uri -> vocab.vocabularyMetadata(uri))
            .map(VocabularyMetadata::name)
            .filter(n -> !n.isEmpty())
            .orElse(null);
        return vocabName != null ? label + " (" + vocabName + ")" : label;
    }
```

- [ ] **Step 5: Replace disposition block in `assembleMarkdownStructural`**

Locate the disposition block in `assembleMarkdownStructural` (around line 364):

```java
        // Disposition
        if (descriptor.disposition() != null) {
            final AgentDisposition d = descriptor.disposition();
            sb.append("\n## How You Operate\n");
            if (d.socialOrient() != null)  sb.append("- Social orientation: ").append(d.socialOrient()).append("\n");
            if (d.ruleFollowing() != null) sb.append("- Rule following: ").append(d.ruleFollowing()).append("\n");
            if (d.riskAppetite() != null)  sb.append("- Risk appetite: ").append(d.riskAppetite()).append("\n");
            if (d.autonomy() != null)      sb.append("- Autonomy: ").append(d.autonomy()).append("\n");
            sb.append("- Can delegate: ").append(d.delegation() ? "yes" : "no").append("\n");
        }
```

Replace with:

```java
        // Disposition
        if (descriptor.disposition() != null) {
            final AgentDisposition d = descriptor.disposition();
            sb.append("\n## How You Operate\n");
            for (DispositionAxis axis : DispositionAxis.values()) {
                d.get(axis).ifPresent(raw ->
                    sb.append("- ").append(axisLabel(axis)).append(": ")
                      .append(resolveAxisDisplay(axis, raw, descriptor)).append("\n"));
            }
            sb.append("- Can delegate: ").append(d.delegation() ? "yes" : "no").append("\n");
        }
```

- [ ] **Step 6: Replace disposition block in `assembleProse`**

Locate the disposition block in `assembleProse` (around line 423):

```java
            if (descriptor.disposition() != null) {
                final AgentDisposition d = descriptor.disposition();
                sb.append("\nOperating style:");
                if (d.ruleFollowing() != null) sb.append(" ").append(d.ruleFollowing()).append(" rule-following.");
                if (d.autonomy() != null)      sb.append(" Autonomy: ").append(d.autonomy()).append(".");
                sb.append(" Can delegate: ").append(d.delegation() ? "yes" : "no").append(".\n");
            }
```

Replace with:

```java
            if (descriptor.disposition() != null) {
                final AgentDisposition d = descriptor.disposition();
                sb.append("\nOperating style:");
                for (DispositionAxis axis : DispositionAxis.values()) {
                    d.get(axis).ifPresent(raw ->
                        sb.append(" ").append(axisLabel(axis)).append(": ")
                          .append(resolveAxisDisplay(axis, raw, descriptor)).append("."));
                }
                sb.append(" Can delegate: ").append(d.delegation() ? "yes" : "no").append(".\n");
            }
```

- [ ] **Step 7: Run all runtime tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime 2>&1 | tail -10
```

Expected: `BUILD SUCCESS`, all tests pass.

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add \
  runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosRenderPipeline.java \
  runtime/src/test/java/io/casehub/eidos/runtime/renderer/EidosRenderPipelineTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#27): structural renderers — vocab-resolved axis labels, conflictMode, all prose axes"
```

---

## Task 7: Add `description` to all 6 vocab enum annotations

**Files:**
- Modify: `vocab/src/main/java/io/casehub/eidos/vocab/SvoTerm.java`
- Modify: `vocab/src/main/java/io/casehub/eidos/vocab/ConscientiousnessTerm.java`
- Modify: `vocab/src/main/java/io/casehub/eidos/vocab/CasehubSlotTerm.java`
- Modify: `vocab/src/main/java/io/casehub/eidos/vocab/BelbinTerm.java`
- Modify: `vocab/src/main/java/io/casehub/eidos/vocab/DiscTerm.java`
- Modify: `vocab/src/main/java/io/casehub/eidos/vocab/ThomasKilmannTerm.java`

Each change is a single annotation attribute addition on the `@VocabularyMetadata` line. The `name` field is unchanged in every case.

- [ ] **Step 1: Update `SvoTerm.java`**

Change:
```java
@VocabularyMetadata(uri = "urn:casehub:vocab:svo", name = "SVO Roles", version = "1.0")
```
To:
```java
@VocabularyMetadata(uri = "urn:casehub:vocab:svo", name = "SVO Roles", version = "1.0",
                    description = "A simplified three-role model (Coordinator, Performer, Evaluator) for agent function in multi-agent workflows. Derived from Subject-Verb-Object role theory. Intended as a lightweight slot vocabulary.")
```

- [ ] **Step 2: Update `ConscientiousnessTerm.java`**

Change:
```java
@VocabularyMetadata(uri = "urn:casehub:vocab:conscientiousness",
                    name = "Conscientiousness Disposition Axes", version = "1.0")
```
To:
```java
@VocabularyMetadata(uri = "urn:casehub:vocab:conscientiousness",
                    name = "Conscientiousness Disposition Axes", version = "1.0",
                    description = "An operational axis vocabulary for agent behavioral disposition, grounded in Big Five Conscientiousness research. Covers rule-following, risk appetite, social orientation, and autonomy as independent dimensions.")
```

- [ ] **Step 3: Update `CasehubSlotTerm.java`**

Change:
```java
@VocabularyMetadata(uri = "urn:casehub:vocab:casehub-slot",
                    name = "CaseHub Slot Roles", version = "1.0")
```
To:
```java
@VocabularyMetadata(uri = "urn:casehub:vocab:casehub-slot",
                    name = "CaseHub Slot Roles", version = "1.0",
                    description = "CaseHub's native slot vocabulary defining four platform-standard roles: Planner, Executor, Reviewer, Supervisor. Use when an external team-role framework is not required.")
```

- [ ] **Step 4: Update `BelbinTerm.java`**

Change:
```java
@VocabularyMetadata(uri = "urn:casehub:vocab:belbin",
                    name = "Belbin Team Roles", version = "1.0")
```
To:
```java
@VocabularyMetadata(uri = "urn:casehub:vocab:belbin",
                    name = "Belbin Team Roles", version = "1.0",
                    description = "Nine complementary team-role archetypes developed by Meredith Belbin from observational research at Henley Management College (1981). Roles describe what a person contributes to a team's function. Medium scientific validity; widely adopted in UK and EU management development.")
```

- [ ] **Step 5: Update `DiscTerm.java`**

Change:
```java
@VocabularyMetadata(uri = "urn:casehub:vocab:disc",
                    name = "DISC Behavioral Styles", version = "1.0")
```
To:
```java
@VocabularyMetadata(uri = "urn:casehub:vocab:disc",
                    name = "DISC Behavioral Styles", version = "1.0",
                    description = "A four-quadrant behavioral style model (Dominance, Influence, Steadiness, Conscientiousness-DISC) used as a disposition shorthand. Correlates with Big Five Extraversion × Agreeableness. Low independent scientific validity, but bounded imprecision makes it usable in practice.")
```

- [ ] **Step 6: Update `ThomasKilmannTerm.java`**

Change:
```java
@VocabularyMetadata(uri = "urn:casehub:vocab:thomas-kilmann",
                    name = "Thomas-Kilmann Conflict Modes", version = "1.0")
```
To:
```java
@VocabularyMetadata(uri = "urn:casehub:vocab:thomas-kilmann",
                    name = "Thomas-Kilmann Conflict Modes", version = "1.0",
                    description = "Five conflict-handling modes from the Thomas-Kilmann Conflict Mode Instrument, based on the assertiveness × cooperativeness framework. Widely adopted in management and applied psychology. Maps to the CONFLICT_MODE disposition axis.")
```

- [ ] **Step 7: Build and test the vocab module**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl vocab -q
```

Expected: `BUILD SUCCESS`

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add \
  vocab/src/main/java/io/casehub/eidos/vocab/SvoTerm.java \
  vocab/src/main/java/io/casehub/eidos/vocab/ConscientiousnessTerm.java \
  vocab/src/main/java/io/casehub/eidos/vocab/CasehubSlotTerm.java \
  vocab/src/main/java/io/casehub/eidos/vocab/BelbinTerm.java \
  vocab/src/main/java/io/casehub/eidos/vocab/DiscTerm.java \
  vocab/src/main/java/io/casehub/eidos/vocab/ThomasKilmannTerm.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#27): add framework description to all 6 vocab enum @VocabularyMetadata annotations"
```

---

## Task 8: Integration tests in examples module

**Files:**
- Modify: `examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/SystemPromptRendererTest.java`

These `@QuarkusTest` tests verify end-to-end pipeline behaviour with CDI, real vocab registration via `VocabularyRegistrar` beans, and no LLM (structural fallback). The assertions target observable rendered output.

- [ ] **Step 1: Write failing tests**

Add these three test methods to `SystemPromptRendererTest` (after the existing tests, before the closing `}`):

```java
    @Test
    void conscientiousness_disposition_shows_vocab_resolved_labels() {
        registry.register(AgentDescriptor.builder()
            .agentId("cons-agent").name("Conscientiousness Agent")
            .slot("reviewer")
            .dispositionVocabulary("urn:casehub:vocab:conscientiousness")
            .disposition(AgentDisposition.builder()
                .socialOrient("facilitative")
                .ruleFollowing("principled")
                .build())
            .tenancyId("default").build());

        final var descriptor = registry.findById("cons-agent", "default").orElseThrow();
        final var context = AgentPromptContext.forFormat(RenderFormat.MARKDOWN);
        final var result = renderer.render(descriptor, context);

        assertThat(result.content()).contains("Facilitative (Conscientiousness Disposition Axes)");
        assertThat(result.content()).contains("Principled (Conscientiousness Disposition Axes)");
        // Raw values must not appear as plain text
        assertThat(result.content()).doesNotContain("Social orientation: facilitative\n");
    }

    @Test
    void thomas_kilmann_conflict_mode_appears_in_structural_output() {
        registry.register(AgentDescriptor.builder()
            .agentId("tk-agent").name("TK Agent")
            .slot("reviewer")
            .dispositionVocabulary("urn:casehub:vocab:thomas-kilmann")
            .disposition(AgentDisposition.builder()
                .conflictMode("collaborating")
                .build())
            .tenancyId("default").build());

        final var descriptor = registry.findById("tk-agent", "default").orElseThrow();
        final var context = AgentPromptContext.forFormat(RenderFormat.MARKDOWN);
        final var result = renderer.render(descriptor, context);

        assertThat(result.content()).contains("Conflict mode:");
        assertThat(result.content()).contains("Collaborating (Thomas-Kilmann Conflict Modes)");
    }

    @Test
    void descriptor_with_vocab_uri_has_different_hash_from_descriptor_without() {
        registry.register(AgentDescriptor.builder()
            .agentId("hash-vocab").name("Agent").slot("reviewer")
            .dispositionVocabulary("urn:casehub:vocab:conscientiousness")
            .disposition(AgentDisposition.builder().socialOrient("facilitative").build())
            .tenancyId("default").build());
        registry.register(AgentDescriptor.builder()
            .agentId("hash-plain").name("Agent").slot("reviewer")
            .disposition(AgentDisposition.builder().socialOrient("facilitative").build())
            .tenancyId("default").build());

        final var ctx = AgentPromptContext.forFormat(RenderFormat.MARKDOWN);
        final var resultVocab = renderer.render(
            registry.findById("hash-vocab", "default").orElseThrow(), ctx);
        final var resultPlain = renderer.render(
            registry.findById("hash-plain", "default").orElseThrow(), ctx);

        // Registered vocab adds vocabularyName to payload → different descriptor hash
        assertThat(resultVocab.descriptorHash()).isNotEqualTo(resultPlain.descriptorHash());
    }
```

- [ ] **Step 2: Run tests — verify they fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl examples/agent-scenarios \
  -Dtest=SystemPromptRendererTest#conscientiousness_disposition_shows_vocab_resolved_labels \
  2>&1 | tail -10
```

Expected: FAIL — `Facilitative (Conscientiousness Disposition Axes)` not found in structural output (Task 6 fix not yet applied... wait, Task 6 is already done). These tests should pass after Task 6 + Task 7. Run to confirm.

- [ ] **Step 3: Run full examples test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl examples/agent-scenarios 2>&1 | tail -10
```

Expected: `BUILD SUCCESS`, all tests pass.

- [ ] **Step 4: Run complete build to confirm no module regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test 2>&1 | tail -15
```

Expected: `BUILD SUCCESS`

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add \
  examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/SystemPromptRendererTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "test(eidos#27): integration tests — vocab-resolved labels, TK conflict mode, cache hash uniqueness Closes #27"
```

---

## Self-Review

**Spec coverage:**
- ✅ Change 1 — `@VocabularyMetadata.description()` → Task 1
- ✅ Change 2 — `VocabularyRegistry.vocabularyMetadata()` SPI + `CdiVocabularyRegistry` impl → Task 2
- ✅ Change 3 — `addIfNonBlank` helper → Task 3
- ✅ Change 4 — Slot section fix → Task 3
- ✅ Change 5 — Disposition section replacement with loop, `axisJsonKey` helper → Task 4
- ✅ Change 6 — `PROMPT_TEMPLATE` + `RESPONSE_FORMAT` update → Task 5
- ✅ Change 7 — Structural renderers + `axisLabel` + `resolveAxisDisplay` → Task 6
- ✅ Change 8 — Vocab enum descriptions → Task 7
- ✅ `vocabularyMetadata_registered_uri_returns_annotation` → Task 2
- ✅ `vocabularyMetadata_unregistered_uri_returns_empty` → Task 2
- ✅ `slot_payload_includes_vocabulary_name_and_description` → Task 3
- ✅ `empty_vocab_name_not_emitted_in_payload` → Task 3
- ✅ `disposition_payload_is_nested_object_per_axis` → Task 4
- ✅ `conflict_mode_included_in_payload_when_set` → Task 4
- ✅ `disposition_without_registered_vocab_has_value_only` → Task 4
- ✅ `different_disposition_vocab_produces_different_descriptor_hash` → Task 4
- ✅ `structural_markdown_shows_axis_label_not_raw_value` → Task 6
- ✅ `structural_markdown_includes_conflict_mode` → Task 6
- ✅ `structural_prose_includes_all_disposition_axes` → Task 6
- ✅ Integration tests (`conscientiousness`, `thomas_kilmann`, `hash uniqueness`) → Task 8

**Type consistency:** `DispositionAxis`, `VocabularyMetadata`, `VocabularyTerm`, `AgentDescriptor`, `AgentDisposition` — all referenced consistently throughout tasks. `axisJsonKey` returns `String`, `axisLabel` returns `String`, `resolveAxisDisplay` returns `String`. No renames or aliasing.

**Placeholder scan:** No TBDs, no "implement later", no "similar to Task N" — all code blocks are complete.
