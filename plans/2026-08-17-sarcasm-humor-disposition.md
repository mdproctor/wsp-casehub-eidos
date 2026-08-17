# Structured Sarcasm/Humor Dimensions Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #140 — Structured sarcasm/humor dimensions in disposition model
**Issue group:** #140

**Goal:** Add structured Sarc7 sarcasm type vocabulary with a new `styleProfile`/`styleVocabulary` concept on the descriptor, hybrid render pipeline, and eval integration.

**Architecture:** Two new fields on core API records (`styleVocabulary` on `AgentDescriptor`, `styleProfile` on `AgentDisposition`) provide a second personality profile slot for communication style vocabularies. `Sarc7Term` enum in `casehub-eidos-vocab` implements 7 sarcasm types with prompt guidance and reference cross-vocabulary mappings. The render pipeline gains a three-layer hybrid architecture (framework-specific structural rendering + generic guidance helper + generic fallback sweep).

**Tech Stack:** Java 21, Quarkus 3.32.2, Maven, JPA/Flyway

## Global Constraints

- Java 21 source level on Java 26 JVM
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- Test: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
- Use `mvn` not `./mvnw`
- All commits reference `Refs #140`
- Pre-release project — no backward compatibility constraints
- Schema changes go directly into base migration files (no deployed instances)
- `axisExactMatch` must use exhaustive switches on `DispositionAxis` (no default branch)

---

## Batch 1: Core API — styleProfile / styleVocabulary

### Task 1: Add styleProfile and styleVocabulary to core API records

**Files:**
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentDisposition.java`
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentDescriptor.java`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/registrar/ClasspathYamlDescriptorRegistrar.java`
- Modify: `runtime/src/main/resources/db/eidos/migration/V1__initial_schema.sql`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentDescriptorEntity.java`
- Test: `api/src/test/java/io/casehub/eidos/api/AgentDispositionStyleProfileTest.java`

**Interfaces:**
- Produces: `AgentDisposition.styleProfile()` → `List<DispositionValue>`
- Produces: `AgentDisposition.Builder.styleProfile(DispositionValue...)` and `styleProfile(List<DispositionValue>)`
- Produces: `AgentDescriptor.styleVocabulary()` → `String`
- Produces: `AgentDescriptor.Builder.styleVocabulary(String)`

- [ ] **Step 1: Write failing test for AgentDisposition.styleProfile**

Create `api/src/test/java/io/casehub/eidos/api/AgentDispositionStyleProfileTest.java`:

```java
package io.casehub.eidos.api;

import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.*;

class AgentDispositionStyleProfileTest {

    @Test
    void styleProfile_defaults_to_empty_list() {
        var disposition = AgentDisposition.builder().build();
        assertThat(disposition.styleProfile()).isEmpty();
    }

    @Test
    void styleProfile_via_builder_varargs() {
        var disposition = AgentDisposition.builder()
                .styleProfile(DispositionValue.of("deadpan"))
                .build();
        assertThat(disposition.styleProfile()).hasSize(1);
        assertThat(disposition.styleProfile().getFirst().term()).isEqualTo("deadpan");
    }

    @Test
    void styleProfile_via_builder_list() {
        var disposition = AgentDisposition.builder()
                .styleProfile(List.of(
                        new DispositionValue("deadpan", 0.7),
                        new DispositionValue("brooding", 0.3)))
                .build();
        assertThat(disposition.styleProfile()).hasSize(2);
    }

    @Test
    void styleProfile_is_immutable() {
        var disposition = AgentDisposition.builder()
                .styleProfile(DispositionValue.of("deadpan"))
                .build();
        assertThatThrownBy(() -> disposition.styleProfile().add(DispositionValue.of("polite")))
                .isInstanceOf(UnsupportedOperationException.class);
    }

    @Test
    void descriptor_styleVocabulary_defaults_to_null() {
        var descriptor = AgentDescriptor.builder()
                .agentId("test").tenancyId("t1").build();
        assertThat(descriptor.styleVocabulary()).isNull();
    }

    @Test
    void descriptor_styleVocabulary_set_via_builder() {
        var descriptor = AgentDescriptor.builder()
                .agentId("test").tenancyId("t1")
                .styleVocabulary("urn:casehub:vocab:sarc7")
                .build();
        assertThat(descriptor.styleVocabulary()).isEqualTo("urn:casehub:vocab:sarc7");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=AgentDispositionStyleProfileTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation failure — `styleProfile()` method does not exist.

- [ ] **Step 3: Add styleProfile to AgentDisposition**

Use `ide_edit_member` to modify the record. Add `styleProfile` as a new field after `dispositionProfile`:

In `AgentDisposition.java`:
- Add record component: `List<DispositionValue> styleProfile` after `dispositionProfile`
- Add to compact constructor: `styleProfile = styleProfile == null ? List.of() : List.copyOf(styleProfile);`
- Add Builder field: `private List<DispositionValue> styleProfile = List.of();`
- Add Builder methods: `styleProfile(DispositionValue... values)` and `styleProfile(List<DispositionValue> v)`
- Update `build()` to pass `styleProfile`

In `AgentDescriptor.java`:
- Add record component: `String styleVocabulary` after `dispositionVocabulary`
- Add Builder field and method: `styleVocabulary(String v)`
- Update `build()` and `toBuilder()` to include `styleVocabulary`

- [ ] **Step 4: Fix all compilation errors from the new record component**

Adding a component to a Java record changes the canonical constructor signature. All callers that construct `AgentDisposition` or `AgentDescriptor` directly (tests, JPA entity, YAML registrar, renderer) will fail to compile. Fix each caller by adding the new parameter.

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile`
Fix each compilation error. Most callers use the Builder, which won't break. Direct record construction and `toBuilder()` calls will need updating.

- [ ] **Step 5: Update ClasspathYamlDescriptorRegistrar**

In `ClasspathYamlDescriptorRegistrar.java`:
- Add `public String styleVocabulary;` to `DescriptorConfig` (line ~182)
- Add `public List<DispositionValueConfig> styleProfile;` to `DispositionConfig` (line ~197)
- In `toDescriptor()`, pass `cfg.styleVocabulary` to the descriptor builder
- In disposition building, map `cfg.disposition.styleProfile` to `List<DispositionValue>` and pass to the disposition builder

- [ ] **Step 6: Update V1__initial_schema.sql**

Add `style_vocabulary TEXT,` after `disposition_vocabulary TEXT,` (line 16).

The `disposition` column is a JSON TEXT blob — `styleProfile` is serialized inside it automatically via the entity's JSON mapping. No separate column needed.

- [ ] **Step 7: Update JPA entity**

In `AgentDescriptorEntity.java`, add `styleVocabulary` field with `@Column(name = "style_vocabulary")` and include it in the entity↔domain mapping methods.

- [ ] **Step 8: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api`
Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: all pass.

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add api/ runtime/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat: add styleProfile/styleVocabulary to AgentDisposition and AgentDescriptor Refs #140"
```

---

## Batch 2: Vocabulary — Sarc7Term

### Task 2: Create Sarc7Term enum and Sarc7VocabRegistrar

**Files:**
- Create: `vocab/src/main/java/io/casehub/eidos/vocab/Sarc7Term.java`
- Create: `vocab/src/main/java/io/casehub/eidos/vocab/Sarc7VocabRegistrar.java`
- Test: `vocab/src/test/java/io/casehub/eidos/vocab/Sarc7VocabularyTest.java`

**Interfaces:**
- Consumes: `VocabularyTerm` interface (api), `ConscientiousnessTerm` + `ThomasKilmannTerm` (vocab)
- Produces: `Sarc7Term` enum with 7 constants, `URI = "urn:casehub:vocab:sarc7"`
- Produces: `incongruity()`, `shockValue()`, `contextDependency()`, `emotionalTone()` — read-only dimension accessors
- Produces: `responseStyleGuidance()`, `antiPatternWarning()` — prompt guidance
- Produces: `axisExactMatch(Class<?>, DispositionAxis)` — reference cross-vocab mappings

- [ ] **Step 1: Write failing tests**

Create `vocab/src/test/java/io/casehub/eidos/vocab/Sarc7VocabularyTest.java`:

```java
package io.casehub.eidos.vocab;

import io.casehub.eidos.api.DispositionAxis;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

class Sarc7VocabularyTest {

    @Test
    void uri_constant_is_correct() {
        assertThat(Sarc7Term.URI).isEqualTo("urn:casehub:vocab:sarc7");
    }

    @Test
    void has_seven_terms() {
        assertThat(Sarc7Term.values()).hasSize(7);
    }

    @Test
    void deadpan_has_correct_value_and_label() {
        assertThat(Sarc7Term.DEADPAN.value()).isEqualTo("deadpan");
        assertThat(Sarc7Term.DEADPAN.label()).isEqualTo("Deadpan");
    }

    @Test
    void deadpan_dimensions_are_paper_derived() {
        assertThat(Sarc7Term.DEADPAN.incongruity()).isEqualTo(0.8);
        assertThat(Sarc7Term.DEADPAN.shockValue()).isEqualTo(0.2);
        assertThat(Sarc7Term.DEADPAN.contextDependency()).isEqualTo(0.6);
        assertThat(Sarc7Term.DEADPAN.emotionalTone()).isEqualTo(0.4);
    }

    @Test
    void all_terms_have_non_empty_responseStyleGuidance() {
        for (Sarc7Term t : Sarc7Term.values()) {
            assertThat(t.responseStyleGuidance())
                .as("responseStyleGuidance for %s", t.name())
                .isNotBlank();
        }
    }

    @Test
    void all_terms_have_non_empty_antiPatternWarning() {
        for (Sarc7Term t : Sarc7Term.values()) {
            assertThat(t.antiPatternWarning())
                .as("antiPatternWarning for %s", t.name())
                .isNotBlank();
        }
    }

    @Test
    void deadpan_maps_to_conscientiousness_independent_on_social_orientation() {
        assertThat(Sarc7Term.DEADPAN.axisExactMatch(ConscientiousnessTerm.class, DispositionAxis.SOCIAL_ORIENTATION))
            .hasValue(ConscientiousnessTerm.INDEPENDENT);
    }

    @Test
    void deadpan_maps_to_tk_avoiding_on_conflict_mode() {
        assertThat(Sarc7Term.DEADPAN.axisExactMatch(ThomasKilmannTerm.class, DispositionAxis.CONFLICT_MODE))
            .hasValue(ThomasKilmannTerm.AVOIDING);
    }

    @Test
    void obnoxious_maps_to_tk_competing_on_conflict_mode() {
        assertThat(Sarc7Term.OBNOXIOUS.axisExactMatch(ThomasKilmannTerm.class, DispositionAxis.CONFLICT_MODE))
            .hasValue(ThomasKilmannTerm.COMPETING);
    }

    @Test
    void unknown_target_vocab_returns_empty() {
        for (Sarc7Term t : Sarc7Term.values()) {
            for (DispositionAxis axis : DispositionAxis.values()) {
                assertThat(t.axisExactMatch(SvoTerm.class, axis)).isEmpty();
            }
        }
    }

    @Test
    void no_term_implies_supervision() {
        for (Sarc7Term t : Sarc7Term.values()) {
            assertThat(t.impliesSupervision())
                .as("impliesSupervision for %s", t.name())
                .isFalse();
        }
    }

    @Test
    void dimensions_are_within_valid_range() {
        for (Sarc7Term t : Sarc7Term.values()) {
            assertThat(t.incongruity()).isBetween(0.0, 1.0);
            assertThat(t.shockValue()).isBetween(0.0, 1.0);
            assertThat(t.contextDependency()).isBetween(0.0, 1.0);
            assertThat(t.emotionalTone()).isBetween(0.0, 1.0);
        }
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl vocab -Dtest=Sarc7VocabularyTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation failure — `Sarc7Term` does not exist.

- [ ] **Step 3: Implement Sarc7Term enum**

Create `vocab/src/main/java/io/casehub/eidos/vocab/Sarc7Term.java` following the `DiscTerm` pattern exactly. The enum:
- Implements `VocabularyTerm`
- Has `@VocabularyMetadata(uri = "urn:casehub:vocab:sarc7", name = "Sarc7 Sarcasm Types", version = "1.0", description = "...")`
- 7 constants with anonymous subclasses overriding `axisExactMatch()` — exhaustive switches on `DispositionAxis` (no default branch)
- Private fields: `value`, `label`, `description`, `aliases` (standard) + `incongruity`, `shockValue`, `contextDependency`, `emotionalTone` (Sarc7-specific)
- Public accessors for dimension fields
- `responseStyleGuidance()` and `antiPatternWarning()` overrides as exhaustive switches on `this`
- Cross-vocabulary mappings per the spec §1.4 table
- `impliesSupervision()` returns `false` for all terms (default)

- [ ] **Step 4: Create Sarc7VocabRegistrar**

Create `vocab/src/main/java/io/casehub/eidos/vocab/Sarc7VocabRegistrar.java`:

```java
package io.casehub.eidos.vocab;

import io.casehub.eidos.api.spi.VocabularyRegistrar;
import jakarta.enterprise.context.ApplicationScoped;

@ApplicationScoped
public class Sarc7VocabRegistrar implements VocabularyRegistrar {
    @Override
    public Class<Sarc7Term> vocabulary() {
        return Sarc7Term.class;
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl vocab`
Expected: all pass including new Sarc7VocabularyTest.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add vocab/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat: add Sarc7Term vocabulary with 7 sarcasm types and reference cross-mappings Refs #140"
```

---

## Batch 3: Render Pipeline — Hybrid Three-Layer Architecture

### Task 3: Extract renderGuidanceBlock helper and refactor Jungian method

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosRenderPipeline.java`
- Test: `runtime/src/test/java/io/casehub/eidos/runtime/renderer/EidosRenderPipelineTest.java` (existing)

**Interfaces:**
- Consumes: `VocabularyTerm.responseStyleGuidance()`, `VocabularyTerm.antiPatternWarning()`
- Produces: `renderGuidanceBlock(StringBuilder, VocabularyTerm, String)` — private helper
- Modifies: `assembleMarkdownCognitiveProfile()` — extracts inline guidance rendering to helper call

- [ ] **Step 1: Write failing test for guidance rendering via styleProfile**

Add to existing render pipeline test class — a test that creates a descriptor with `styleVocabulary = "urn:casehub:vocab:sarc7"` and `styleProfile = [DispositionValue("deadpan", 1.0)]`, renders in MARKDOWN format, and asserts the output contains Sarc7-specific guidance text. This test will fail until Task 4 wires the dispatch.

For now, verify the refactored Jungian path still works by running existing tests.

- [ ] **Step 2: Extract renderGuidanceBlock helper**

In `EidosRenderPipeline.java`, create a private helper method:

```java
private void renderGuidanceBlock(StringBuilder sb, VocabularyTerm term, String heading) {
    final String style = term.responseStyleGuidance();
    if (style != null && !style.isEmpty()) {
        sb.append("\n**").append(heading).append(":** ").append(style).append("\n");
    }
    final String avoid = term.antiPatternWarning();
    if (avoid != null && !avoid.isEmpty()) {
        sb.append("\n**Avoid:** ").append(avoid).append("\n");
    }
}
```

- [ ] **Step 3: Refactor assembleMarkdownCognitiveProfile to use the helper**

Replace the inline guidance rendering (lines ~629-638) with a call to:
```java
renderGuidanceBlock(sb, term, "Your Response Style");
```

- [ ] **Step 4: Run existing Jungian rendering tests to verify no regression**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: all pass — Jungian rendering output unchanged.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add runtime/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "refactor: extract renderGuidanceBlock helper from Jungian cognitive profile renderer Refs #140"
```

### Task 4: Add assembleSarc7HumorProfile, dispatch, and generic fallback sweep

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosRenderPipeline.java`
- Test: `runtime/src/test/java/io/casehub/eidos/runtime/renderer/Sarc7RenderPipelineTest.java`

**Interfaces:**
- Consumes: `Sarc7Term` enum (vocab), `AgentDescriptor.styleVocabulary()`, `AgentDisposition.styleProfile()` (from Task 1)
- Produces: `assembleSarc7HumorProfile(StringBuilder, AgentDescriptor)` — Sarc7-specific rendering
- Produces: `assembleGenericVocabularyGuidance(StringBuilder, AgentDescriptor, Set<String>)` — fallback sweep

- [ ] **Step 1: Write failing test for Sarc7 MARKDOWN rendering**

Create `runtime/src/test/java/io/casehub/eidos/runtime/renderer/Sarc7RenderPipelineTest.java`:

```java
package io.casehub.eidos.runtime.renderer;

import io.casehub.eidos.api.*;
import io.casehub.eidos.vocab.Sarc7Term;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

// @QuarkusTest with InMemory registry and Sarc7VocabRegistrar
class Sarc7RenderPipelineTest {

    @Test
    void markdown_renders_sarc7_humor_section() {
        var descriptor = AgentDescriptor.builder()
                .agentId("snarky-bot").tenancyId("t1")
                .styleVocabulary(Sarc7Term.URI)
                .disposition(AgentDisposition.builder()
                        .styleProfile(DispositionValue.of("deadpan"))
                        .build())
                .build();
        var context = new AgentPromptContext(null, List.of(), null, RenderFormat.MARKDOWN);
        // Render and assert output contains "Deadpan" and guidance text
        // Assert output contains "Your Humor Style:" heading
        // Assert output does NOT contain Jungian cognitive profile sections
    }

    @Test
    void jungian_and_sarc7_coexist_in_markdown() {
        var descriptor = AgentDescriptor.builder()
                .agentId("thoughtful-snarker").tenancyId("t1")
                .dispositionVocabulary("urn:casehub:vocab:jungian")
                .disposition(AgentDisposition.builder()
                        .dispositionProfile(
                                new DispositionValue("ti", 0.45),
                                new DispositionValue("ne", 0.20))
                        .styleProfile(DispositionValue.of("deadpan"))
                        .build())
                .styleVocabulary(Sarc7Term.URI)
                .build();
        var context = new AgentPromptContext(null, List.of(), null, RenderFormat.MARKDOWN);
        // Render and assert output contains BOTH cognitive profile AND humor profile
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=Sarc7RenderPipelineTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — no Sarc7 rendering method exists yet.

- [ ] **Step 3: Implement assembleSarc7HumorProfile**

Add to `EidosRenderPipeline.java`:

```java
private static final String SARC7_VOCAB_URI = Sarc7Term.URI;

private void assembleSarc7HumorProfile(StringBuilder sb, AgentDescriptor descriptor) {
    var styleProfile = descriptor.disposition().styleProfile();
    if (styleProfile.isEmpty()) return;

    var sorted = styleProfile.stream()
            .sorted((a, b) -> Double.compare(b.weight(), a.weight()))
            .toList();

    var primaryTerm = sorted.getFirst().term();
    var resolved = vocab.resolve(SARC7_VOCAB_URI, primaryTerm);
    if (resolved.isEmpty()) return;

    var term = resolved.get();
    sb.append("\n## Communication Style — Sarcasm\n\n");
    sb.append("Your default sarcasm style is **").append(term.label()).append("**");
    if (!term.description().isEmpty()) {
        sb.append(" — ").append(term.description());
    }
    sb.append(".\n");

    if (sorted.size() > 1) {
        sb.append("\nSecondary influences: ");
        sorted.stream().skip(1).forEach(dv ->
            vocab.resolve(SARC7_VOCAB_URI, dv.term()).ifPresent(t ->
                sb.append(t.label()).append(" (").append(formatWeight(dv.weight())).append("), ")));
        sb.setLength(sb.length() - 2);
        sb.append(".\n");
    }

    renderGuidanceBlock(sb, (io.casehub.eidos.api.VocabularyTerm) term, "Your Humor Style");
}
```

- [ ] **Step 4: Add dispatch in assembleMarkdown**

In the `assembleMarkdown()` method, add after `assembleMarkdownCognitiveProfile()`:

```java
if (SARC7_VOCAB_URI.equals(descriptor.styleVocabulary())
        && !descriptor.disposition().styleProfile().isEmpty()) {
    assembleSarc7HumorProfile(sb, descriptor);
}
```

- [ ] **Step 5: Implement assembleGenericVocabularyGuidance**

Add the fallback sweep method:

```java
private void assembleGenericVocabularyGuidance(StringBuilder sb, AgentDescriptor descriptor, Set<String> rendered) {
    var styleVocab = descriptor.styleVocabulary();
    if (styleVocab == null || rendered.contains(styleVocab)) return;
    var styleProfile = descriptor.disposition().styleProfile();
    if (styleProfile.isEmpty()) return;

    var primaryTerm = styleProfile.stream()
            .max((a, b) -> Double.compare(a.weight(), b.weight()))
            .map(dv -> vocab.resolve(styleVocab, dv.term()))
            .flatMap(opt -> opt);
    primaryTerm.ifPresent(term ->
        renderGuidanceBlock(sb, term, "Style Guidance"));
}
```

Wire into `assembleMarkdown()` after the Sarc7 dispatch, tracking rendered URIs:

```java
Set<String> renderedStyleVocabs = new HashSet<>();
if (SARC7_VOCAB_URI.equals(descriptor.styleVocabulary())) {
    renderedStyleVocabs.add(SARC7_VOCAB_URI);
}
assembleGenericVocabularyGuidance(sb, descriptor, renderedStyleVocabs);
```

- [ ] **Step 6: Add A2A_CARD styleProfile block**

In `assembleA2aCard()`, add a `styleProfile` JSON block alongside the existing disposition data. Include the Sarc7 type, dimensions, responseStyle, and antiPattern.

- [ ] **Step 7: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: all pass including new Sarc7RenderPipelineTest.

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add runtime/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat: add Sarc7 humor profile rendering with hybrid three-layer pipeline Refs #140"
```

---

## Batch 4: Documentation and Eval

### Task 5: Update personality-frameworks.md

**Files:**
- Modify: `docs/personality-frameworks.md`

**Interfaces:**
- Consumes: Sarc7 cross-mapping table from spec §1.4
- Produces: Updated reference document with Sarc7 section, cross-reference table, compatibility matrix

- [ ] **Step 1: Add §2.5 Sarc7 Sarcasm Types section**

After §2.4 Jungian Cognitive Functions, add a new section following the existing format: What it models, Scientific validity, Workplace adoption, Vocabulary role, mapping table with all 7 terms × 5 axes + delegation + conflictMode.

Include the note that mappings are editorial (no psychometric grounding) and that Sarc7 uses `styleProfile`/`styleVocabulary`, NOT disposition axes.

- [ ] **Step 2: Update §5 Cross-Reference Summary Table**

Add a `Sarc7` column. Rows `socialOrient` through `conflictMode` get `reference` (not `**disposition**` — mappings are reference only, not axis-population drivers). All other rows get `—`.

- [ ] **Step 3: Update §6 Framework Compatibility**

Add four entries:
- Sarc7 + Jungian: Additive
- Sarc7 + DISC: Additive
- Sarc7 + Belbin: Additive
- Sarc7 + Conscientiousness: Additive (Sarc7 uses styleProfile; no axis conflict)

- [ ] **Step 4: Add Sarc7 Profile combination pattern**

After the Jungian Profile pattern, add a Sarc7 Profile pattern showing `styleVocabulary` + `styleProfile` usage, and a combined Jungian+Sarc7 example.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add docs/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "docs: add Sarc7 sarcasm types to personality-frameworks.md Refs #140"
```

### Task 6: Add eval YAML profiles and scenarios

**Files:**
- Create: `eval/src/test/resources/profiles/sarc7-deadpan.yaml`
- Create: `eval/src/test/resources/profiles/sarc7-obnoxious.yaml`
- Create: `eval/src/test/resources/profiles/sarc7-polite.yaml`
- Create: `eval/src/test/resources/profiles/sarc7-self-deprecating.yaml`
- Create: `eval/src/test/resources/profiles/sarc7-brooding.yaml`
- Create: `eval/src/test/resources/profiles/sarc7-raging.yaml`
- Create: `eval/src/test/resources/profiles/sarc7-manic.yaml`
- Create: `eval/src/test/resources/profiles/jungian-sarc7-combined.yaml`
- Test: `eval/src/test/java/io/casehub/eidos/eval/Sarc7EvalTest.java`

**Interfaces:**
- Consumes: Sarc7Term enum, existing eval YAML profile format, TraitExpressionJudge, PairContrastJudge

- [ ] **Step 1: Create YAML profiles for all 7 Sarc7 types**

Each profile follows the existing eval YAML format. Example for deadpan:

```yaml
agentId: sarc7-deadpan
name: "Deadpan Sarcasm Agent"
tenancyId: eval
styleVocabulary: "urn:casehub:vocab:sarc7"
disposition:
  styleProfile:
    - term: deadpan
      weight: 1.0
```

Create all 7 single-type profiles.

- [ ] **Step 2: Create combined Jungian+Sarc7 profile**

```yaml
agentId: jungian-sarc7-intp-deadpan
name: "INTP with Deadpan Sarcasm"
tenancyId: eval
dispositionVocabulary: "urn:casehub:vocab:jungian"
styleVocabulary: "urn:casehub:vocab:sarc7"
disposition:
  mbtiType: INTP
  styleProfile:
    - term: deadpan
      weight: 1.0
```

- [ ] **Step 3: Create Sarc7EvalTest**

Create `eval/src/test/java/io/casehub/eidos/eval/Sarc7EvalTest.java` with:
- `evaluateSarc7Scenarios()` — renders all 7 profiles and asserts output contains sarcasm-type-specific guidance
- `evaluateSarc7PairContrast()` — uses PairContrastJudge to verify DEADPAN vs OBNOXIOUS produce distinguishable output
- `evaluateJungianSarc7Combination()` — verifies combined profile renders both cognitive profile and humor sections

- [ ] **Step 4: Run eval tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -Peval -Dtest=Sarc7EvalTest`
Expected: pass (structural rendering tests; LLM judge tests require claude CLI or Ollama).

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add eval/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat: add Sarc7 eval profiles and scenarios Refs #140"
```

---

## References

- [2026-08-16-sarcasm-humor-disposition-design.md] — design spec this plan implements
- [api/src/main/java/io/casehub/eidos/api/AgentDisposition.java] — core record being extended
- [api/src/main/java/io/casehub/eidos/api/AgentDescriptor.java] — core record being extended
- [vocab/src/main/java/io/casehub/eidos/vocab/DiscTerm.java] — pattern for vocabulary enum
- [vocab/src/main/java/io/casehub/eidos/vocab/DiscVocabRegistrar.java] — pattern for registrar bean
- [runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosRenderPipeline.java:591-639] — Jungian rendering being refactored
- [runtime/src/main/java/io/casehub/eidos/runtime/registrar/ClasspathYamlDescriptorRegistrar.java] — YAML support being extended
- [docs/personality-frameworks.md] — authoritative cross-vocabulary mapping reference
- [docs/protocols/eval/disposition-axis-string-boundary.md] — eval constants protocol
- [GitHub #140] — focal issue
