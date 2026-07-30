# Vocabulary Imbue-and-Verify Test Suite Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #122 — Vocabulary imbue-and-verify test suite in eidos-eval
**Issue group:** #122

**Goal:** Prove each personality framework (Jungian, Belbin, DISC,
Conscientiousness) produces expected behavioral signal when an LLM is
imbued with it, both in isolation and in pairwise composition.

**Architecture:** Two-layer test suite in eidos-eval. Layer 1 (structural)
builds descriptors, runs them through DescriptorCollector and
SystemPromptRenderer with a stub ChatModel, and asserts deterministic
outcomes (profile weights, axis derivation, prompt content). Layer 2
(eval) uses live LLM judges — existing MbtiAlignmentJudge,
FunctionActivationJudge, PersonalityEvolutionJudge plus a new
DispositionPresenceJudge — to verify behavioral differentiation.

**Tech Stack:** Java 26, Quarkus, JUnit 5, AssertJ, CDI, langchain4j

## Global Constraints

- All production code in `eval/src/main/java/io/casehub/eidos/eval/`
- All test code in `eval/src/test/java/io/casehub/eidos/eval/`
- No new Maven dependencies (all judges use existing CDI ChatModel + ObjectMapper)
- Structural tests: no `@Tag("eval")`, must pass without LLM
- Eval tests: `@Tag("eval")`, run only under `-Peval`
- Every Jungian descriptor sets `dispositionVocabulary: urn:casehub:vocab:jungian`
- Every Belbin descriptor sets `slotVocabulary: urn:casehub:vocab:belbin`
- Every DISC descriptor sets `dispositionVocabulary: urn:casehub:vocab:disc`
- Use `DescriptorCollector.deriveDispositionAxes()` for axis derivation — never set axes manually on derived vocabs

---

### Task 1: DispositionPresenceJudge + unit test

**Files:**
- Create: `eval/src/main/java/io/casehub/eidos/eval/DispositionPresenceResult.java`
- Create: `eval/src/main/java/io/casehub/eidos/eval/DispositionPresenceJudge.java`
- Create: `eval/src/test/java/io/casehub/eidos/eval/DispositionPresenceJudgeTest.java`

**Interfaces:**
- Consumes: `PromptJudge.extractJson(String)` for JSON response parsing
- Produces: `DispositionPresenceJudge.evaluate(String systemPrompt, String termLabel, String termDescription) → DispositionPresenceResult`
- Produces: `DispositionPresenceResult(String termLabel, double score, String reasoning, boolean aligned)` — aligned when score ≥ 0.7

- [ ] **Step 1: Write DispositionPresenceResult record**

```java
package io.casehub.eidos.eval;

public record DispositionPresenceResult(
        String termLabel,
        double score,
        String reasoning,
        boolean aligned) {}
```

Use `ide_create_file` for `eval/src/main/java/io/casehub/eidos/eval/DispositionPresenceResult.java`.

- [ ] **Step 2: Write failing test for DispositionPresenceJudge**

```java
package io.casehub.eidos.eval;

import com.fasterxml.jackson.databind.ObjectMapper;
import dev.langchain4j.data.message.AiMessage;
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.model.chat.request.ChatRequest;
import dev.langchain4j.model.chat.response.ChatResponse;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class DispositionPresenceJudgeTest {

    static final String HIGH_SCORE_RESPONSE = """
        { "score": 0.9, "reasoning": "The prompt explicitly describes a driven, challenging personality" }
        """;

    static final String LOW_SCORE_RESPONSE = """
        { "score": 0.2, "reasoning": "The trait is not evident in the prompt" }
        """;

    @Test
    void high_score_is_aligned() {
        var judge = new DispositionPresenceJudge(stubModel(HIGH_SCORE_RESPONSE), new ObjectMapper());
        var result = judge.evaluate("You are a driven leader.", "Shaper",
                "Challenges the team to improve; driven, dynamic, thrives under pressure");
        assertThat(result.aligned()).isTrue();
        assertThat(result.score()).isGreaterThanOrEqualTo(0.7);
        assertThat(result.termLabel()).isEqualTo("Shaper");
    }

    @Test
    void low_score_is_not_aligned() {
        var judge = new DispositionPresenceJudge(stubModel(LOW_SCORE_RESPONSE), new ObjectMapper());
        var result = judge.evaluate("You are a calm mediator.", "Shaper",
                "Challenges the team to improve; driven, dynamic, thrives under pressure");
        assertThat(result.aligned()).isFalse();
        assertThat(result.score()).isLessThan(0.7);
    }

    @Test
    void reasoning_is_captured() {
        var judge = new DispositionPresenceJudge(stubModel(HIGH_SCORE_RESPONSE), new ObjectMapper());
        var result = judge.evaluate("prompt", "Shaper", "description");
        assertThat(result.reasoning()).contains("driven");
    }

    @Test
    void boundary_score_0_7_is_aligned() {
        var response = """
            { "score": 0.7, "reasoning": "borderline" }
            """;
        var judge = new DispositionPresenceJudge(stubModel(response), new ObjectMapper());
        assertThat(judge.evaluate("prompt", "X", "desc").aligned()).isTrue();
    }

    private static ChatModel stubModel(String response) {
        return new ChatModel() {
            @Override
            public ChatResponse doChat(ChatRequest request) {
                return ChatResponse.builder().aiMessage(AiMessage.from(response)).build();
            }
        };
    }
}
```

Use `ide_create_file` for the test file.

- [ ] **Step 3: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -Dtest=DispositionPresenceJudgeTest -f /Users/mdproctor/claude/casehub/eidos/pom.xml`
Expected: FAIL — `DispositionPresenceJudge` does not exist yet.

- [ ] **Step 4: Implement DispositionPresenceJudge**

```java
package io.casehub.eidos.eval;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import dev.langchain4j.data.message.SystemMessage;
import dev.langchain4j.data.message.UserMessage;
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.model.chat.request.ChatRequest;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Any;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;

@ApplicationScoped
public class DispositionPresenceJudge {

    static final String JUDGE_PROMPT = """
        You are evaluating whether an AI agent's system prompt expresses a specific
        personality or behavioral trait.

        The trait: [%s] — %s

        Read the system prompt carefully. Score how strongly this trait is expressed:
        - 1.0: The trait is explicitly and clearly expressed
        - 0.7: The trait is present but not dominant
        - 0.4: Weakly present or only implied
        - 0.0: Not present or contradicted

        Return ONLY raw JSON — no markdown, no code blocks:
        { "score": number, "reasoning": string }
        """;

    private final ChatModel judgeModel;
    private final ObjectMapper mapper;

    @Inject
    public DispositionPresenceJudge(@Any final Instance<ChatModel> models,
                                     final ObjectMapper mapper) {
        if (!models.isResolvable()) throw new IllegalStateException("ChatModel not configured.");
        this.judgeModel = models.get();
        this.mapper = mapper;
    }

    DispositionPresenceJudge(final ChatModel judgeModel, final ObjectMapper mapper) {
        this.judgeModel = judgeModel;
        this.mapper = mapper;
    }

    public DispositionPresenceResult evaluate(final String systemPrompt,
                                               final String termLabel,
                                               final String termDescription) {
        final String judgePrompt = String.format(JUDGE_PROMPT, termLabel, termDescription);
        try {
            final var request = ChatRequest.builder()
                    .messages(
                            SystemMessage.from(judgePrompt),
                            UserMessage.from(systemPrompt))
                    .build();
            final String response = judgeModel.chat(request).aiMessage().text();
            return parse(termLabel, response);
        } catch (final MalformedJudgeResponseException e) {
            throw e;
        } catch (final Exception e) {
            throw new IllegalStateException("DispositionPresenceJudge LLM call failed", e);
        }
    }

    private DispositionPresenceResult parse(final String termLabel, final String json) {
        try {
            final JsonNode root = mapper.readTree(PromptJudge.extractJson(json));
            final double score = root.has("score") ? root.get("score").asDouble() : 0.0;
            final String reasoning = root.has("reasoning") ? root.get("reasoning").asText() : "";
            return new DispositionPresenceResult(termLabel, score, reasoning, score >= 0.7);
        } catch (final Exception e) {
            throw new MalformedJudgeResponseException(
                    "Failed to parse DispositionPresenceJudge response: " + e.getMessage());
        }
    }
}
```

Use `ide_create_file` for the production file.

- [ ] **Step 5: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -Dtest=DispositionPresenceJudgeTest -f /Users/mdproctor/claude/casehub/eidos/pom.xml`
Expected: PASS — all 4 tests green.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add eval/src/main/java/io/casehub/eidos/eval/DispositionPresenceResult.java eval/src/main/java/io/casehub/eidos/eval/DispositionPresenceJudge.java eval/src/test/java/io/casehub/eidos/eval/DispositionPresenceJudgeTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(#122): DispositionPresenceJudge — general-purpose vocabulary trait alignment judge"
```

---

### Task 2: VocabularyImbueFixtures — shared descriptor builders

**Files:**
- Create: `eval/src/test/java/io/casehub/eidos/eval/VocabularyImbueFixtures.java`

**Interfaces:**
- Consumes: `MbtiTypeTerm.defaultProfile()`, `AgentDescriptor.builder()`, `DescriptorCollector.deriveDispositionAxes()`
- Produces:
  - `jungianDescriptor(MbtiTypeTerm type) → AgentDescriptor`
  - `belbinDescriptor(BelbinTerm role) → AgentDescriptor`
  - `discDescriptor(DiscTerm style, VocabularyRegistry vocabRegistry) → AgentDescriptor`
  - `conscientiousnessDescriptor(Map<DispositionAxis, String> axes) → AgentDescriptor`
  - `composite(AgentDescriptor base, AgentDescriptor overlay) → AgentDescriptor`

- [ ] **Step 1: Write failing test for fixtures**

Create a small test class that exercises the fixture methods and asserts basic correctness.

```java
package io.casehub.eidos.eval;

import io.casehub.eidos.api.DispositionAxis;
import io.casehub.eidos.api.VocabularyRegistry;
import io.casehub.eidos.vocab.BelbinTerm;
import io.casehub.eidos.vocab.DiscTerm;
import io.casehub.eidos.vocab.MbtiTypeTerm;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class VocabularyImbueFixturesTest {

    @Inject VocabularyRegistry vocabRegistry;

    @Test
    void jungian_descriptor_has_profile_and_vocab_uri() {
        var desc = VocabularyImbueFixtures.jungianDescriptor(MbtiTypeTerm.ENTJ);
        assertThat(desc.dispositionVocabulary()).isEqualTo("urn:casehub:vocab:jungian");
        assertThat(desc.disposition().dispositionProfile()).hasSize(8);
        assertThat(desc.disposition().dispositionProfile().get(0).term()).isEqualTo("te");
    }

    @Test
    void belbin_descriptor_has_slot_and_slot_vocab() {
        var desc = VocabularyImbueFixtures.belbinDescriptor(BelbinTerm.SHAPER);
        assertThat(desc.slotVocabulary()).isEqualTo("urn:casehub:vocab:belbin");
        assertThat(desc.slot()).isEqualTo("shaper");
        assertThat(desc.disposition().dispositionProfile()).isEmpty();
    }

    @Test
    void disc_descriptor_has_profile_and_vocab_uri() {
        var desc = VocabularyImbueFixtures.discDescriptor(DiscTerm.DOMINANCE, vocabRegistry);
        assertThat(desc.dispositionVocabulary()).isEqualTo("urn:casehub:vocab:disc");
        assertThat(desc.disposition().dispositionProfile()).hasSize(1);
        assertThat(desc.disposition().dispositionProfile().get(0).term()).isEqualTo("dominance");
    }

    @Test
    void conscientiousness_descriptor_has_flat_axes() {
        var desc = VocabularyImbueFixtures.conscientiousnessDescriptor(
                Map.of(DispositionAxis.RULE_FOLLOWING, "strict",
                       DispositionAxis.RISK_APPETITE, "conservative"));
        assertThat(desc.disposition().primaryTerm(DispositionAxis.RULE_FOLLOWING)).isEqualTo("strict");
        assertThat(desc.disposition().primaryTerm(DispositionAxis.RISK_APPETITE)).isEqualTo("conservative");
        assertThat(desc.disposition().dispositionProfile()).isEmpty();
    }

    @Test
    void composite_merges_jungian_and_belbin() {
        var jungian = VocabularyImbueFixtures.jungianDescriptor(MbtiTypeTerm.ENTJ);
        var belbin = VocabularyImbueFixtures.belbinDescriptor(BelbinTerm.SHAPER);
        var comp = VocabularyImbueFixtures.composite(jungian, belbin);
        assertThat(comp.dispositionVocabulary()).isEqualTo("urn:casehub:vocab:jungian");
        assertThat(comp.disposition().dispositionProfile()).hasSize(8);
        assertThat(comp.slotVocabulary()).isEqualTo("urn:casehub:vocab:belbin");
        assertThat(comp.slot()).isEqualTo("shaper");
    }
}
```

Use `ide_create_file`.

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -Dtest=VocabularyImbueFixturesTest -f /Users/mdproctor/claude/casehub/eidos/pom.xml`
Expected: FAIL — `VocabularyImbueFixtures` does not exist.

- [ ] **Step 3: Implement VocabularyImbueFixtures**

```java
package io.casehub.eidos.eval;

import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.eidos.api.AgentDisposition;
import io.casehub.eidos.api.DispositionAxis;
import io.casehub.eidos.api.DispositionValue;
import io.casehub.eidos.api.VocabularyRegistry;
import io.casehub.eidos.runtime.registrar.DescriptorCollector;
import io.casehub.eidos.vocab.BelbinTerm;
import io.casehub.eidos.vocab.DiscTerm;
import io.casehub.eidos.vocab.MbtiTypeTerm;

import java.util.List;
import java.util.Map;

final class VocabularyImbueFixtures {

    private VocabularyImbueFixtures() {}

    static AgentDescriptor jungianDescriptor(MbtiTypeTerm type) {
        return AgentDescriptor.builder()
                .agentId("imbue-" + type.value()).name("Test " + type.label())
                .slot("test-agent").tenancyId("imbue-test")
                .dispositionVocabulary("urn:casehub:vocab:jungian")
                .disposition(AgentDisposition.builder()
                        .dispositionProfile(type.defaultProfile())
                        .build())
                .build();
    }

    static AgentDescriptor belbinDescriptor(BelbinTerm role) {
        return AgentDescriptor.builder()
                .agentId("imbue-belbin-" + role.value()).name("Test " + role.label())
                .slot(role.value()).tenancyId("imbue-test")
                .slotVocabulary("urn:casehub:vocab:belbin")
                .disposition(AgentDisposition.builder().build())
                .build();
    }

    static AgentDescriptor discDescriptor(DiscTerm style, VocabularyRegistry vocabRegistry) {
        var raw = AgentDescriptor.builder()
                .agentId("imbue-disc-" + style.value()).name("Test " + style.label())
                .slot("test-agent").tenancyId("imbue-test")
                .dispositionVocabulary("urn:casehub:vocab:disc")
                .disposition(AgentDisposition.builder()
                        .dispositionProfile(List.of(new DispositionValue(style.value(), 1.0)))
                        .build())
                .build();
        return DescriptorCollector.deriveDispositionAxes(raw, vocabRegistry);
    }

    static AgentDescriptor conscientiousnessDescriptor(Map<DispositionAxis, String> axes) {
        var builder = AgentDisposition.builder();
        axes.forEach((axis, value) -> {
            switch (axis) {
                case SOCIAL_ORIENTATION -> builder.socialOrient(value);
                case RULE_FOLLOWING -> builder.ruleFollowing(value);
                case RISK_APPETITE -> builder.riskAppetite(value);
                case AUTONOMY -> builder.autonomy(value);
                case CONFLICT_MODE -> builder.conflictMode(value);
            }
        });
        return AgentDescriptor.builder()
                .agentId("imbue-conscientiousness").name("Test Conscientiousness")
                .slot("test-agent").tenancyId("imbue-test")
                .disposition(builder.build())
                .build();
    }

    static AgentDescriptor composite(AgentDescriptor base, AgentDescriptor overlay) {
        return AgentDescriptor.builder()
                .agentId(base.agentId() + "+" + overlay.agentId())
                .name(base.name() + " + " + overlay.name())
                .slot(overlay.slot() != null ? overlay.slot() : base.slot())
                .tenancyId(base.tenancyId())
                .dispositionVocabulary(base.dispositionVocabulary())
                .slotVocabulary(overlay.slotVocabulary())
                .briefing(base.briefing())
                .disposition(AgentDisposition.builder()
                        .dispositionProfile(base.disposition().dispositionProfile())
                        .build())
                .build();
    }
}
```

Use `ide_create_file`.

**Note:** `DescriptorCollector` is package-private (`final class`) in `io.casehub.eidos.runtime.registrar`. The eval module cannot call `deriveDispositionAxes` directly. The fix: make both the class and the method public. `DescriptorCollector` is a stateless utility (private constructor, two static methods) — there's no encapsulation reason to hide it.

Use `ide_edit_member` on `DescriptorCollector` (member = class name) to change `final class DescriptorCollector` to `public final class DescriptorCollector`, and on `deriveDispositionAxes` to change from `static` to `public static`.

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -Dtest=VocabularyImbueFixturesTest -f /Users/mdproctor/claude/casehub/eidos/pom.xml`
Expected: PASS — all 5 tests green.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add eval/src/test/java/io/casehub/eidos/eval/VocabularyImbueFixtures.java eval/src/test/java/io/casehub/eidos/eval/VocabularyImbueFixturesTest.java runtime/src/main/java/io/casehub/eidos/runtime/registrar/DescriptorCollector.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(#122): VocabularyImbueFixtures — shared descriptor builders for imbue tests"
```

---

### Task 3: VocabularyImbueStructuralTest — Layer 1

**Files:**
- Create: `eval/src/test/java/io/casehub/eidos/eval/VocabularyImbueStructuralTest.java`

**Interfaces:**
- Consumes: `VocabularyImbueFixtures.*`, `DescriptorCollector.deriveDispositionAxes()`, `SystemPromptRenderer.render()`
- Produces: No public API — test-only class

- [ ] **Step 1: Write single-vocabulary structural tests**

```java
package io.casehub.eidos.eval;

import io.casehub.eidos.api.*;
import io.casehub.eidos.api.SystemPromptRenderer.RenderFormat;
import io.casehub.eidos.runtime.registrar.DescriptorCollector;
import io.casehub.eidos.vocab.BelbinTerm;
import io.casehub.eidos.vocab.DiscTerm;
import io.casehub.eidos.vocab.MbtiTypeTerm;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class VocabularyImbueStructuralTest {

    @Inject SystemPromptRenderer renderer;
    @Inject VocabularyRegistry vocabRegistry;

    // ── Jungian ──────────────────────────────────────────────────────

    @Test
    void jungian_entj_has_8_function_profile() {
        var desc = VocabularyImbueFixtures.jungianDescriptor(MbtiTypeTerm.ENTJ);
        var derived = DescriptorCollector.deriveDispositionAxes(desc, vocabRegistry);
        assertThat(derived.disposition().dispositionProfile()).hasSize(8);
        assertThat(derived.disposition().dispositionProfile().get(0).term()).isEqualTo("te");
        assertThat(derived.disposition().dispositionProfile().get(0).weight()).isEqualTo(0.35);
        assertThat(derived.disposition().dispositionProfile().get(1).term()).isEqualTo("ni");
        assertThat(derived.disposition().dispositionProfile().get(1).weight()).isEqualTo(0.20);
    }

    @Test
    void jungian_entj_derives_all_5_axes() {
        var desc = VocabularyImbueFixtures.jungianDescriptor(MbtiTypeTerm.ENTJ);
        var derived = DescriptorCollector.deriveDispositionAxes(desc, vocabRegistry);
        assertThat(derived.disposition().primaryTerm(DispositionAxis.SOCIAL_ORIENTATION)).isEqualTo("independent");
        assertThat(derived.disposition().primaryTerm(DispositionAxis.RULE_FOLLOWING)).isEqualTo("strict");
        assertThat(derived.disposition().primaryTerm(DispositionAxis.RISK_APPETITE)).isEqualTo("measured");
        assertThat(derived.disposition().primaryTerm(DispositionAxis.AUTONOMY)).isEqualTo("semi-autonomous");
        assertThat(derived.disposition().primaryTerm(DispositionAxis.CONFLICT_MODE)).isEqualTo("competing");
    }

    @Test
    void jungian_prompt_contains_profile_terms() {
        var desc = VocabularyImbueFixtures.jungianDescriptor(MbtiTypeTerm.ENTJ);
        var derived = DescriptorCollector.deriveDispositionAxes(desc, vocabRegistry);
        var prompt = renderer.render(derived, AgentPromptContext.forFormat(RenderFormat.MARKDOWN));
        assertThat(prompt.content()).containsIgnoringCase("thinking");
    }

    // ── Belbin ───────────────────────────────────────────────────────

    @Test
    void belbin_shaper_has_no_profile() {
        var desc = VocabularyImbueFixtures.belbinDescriptor(BelbinTerm.SHAPER);
        assertThat(desc.disposition().dispositionProfile()).isEmpty();
    }

    @Test
    void belbin_shaper_has_no_derived_axes() {
        var desc = VocabularyImbueFixtures.belbinDescriptor(BelbinTerm.SHAPER);
        for (var axis : DispositionAxis.values()) {
            assertThat(desc.disposition().get(axis)).isEmpty();
        }
    }

    @Test
    void belbin_prompt_contains_slot_label() {
        var desc = VocabularyImbueFixtures.belbinDescriptor(BelbinTerm.SHAPER);
        var prompt = renderer.render(desc, AgentPromptContext.forFormat(RenderFormat.MARKDOWN));
        assertThat(prompt.content()).containsIgnoringCase("shaper");
    }

    // ── DISC ─────────────────────────────────────────────────────────

    @Test
    void disc_dominance_derives_all_5_axes() {
        var desc = VocabularyImbueFixtures.discDescriptor(DiscTerm.DOMINANCE, vocabRegistry);
        assertThat(desc.disposition().primaryTerm(DispositionAxis.SOCIAL_ORIENTATION)).isEqualTo("independent");
        assertThat(desc.disposition().primaryTerm(DispositionAxis.RULE_FOLLOWING)).isEqualTo("flexible");
        assertThat(desc.disposition().primaryTerm(DispositionAxis.RISK_APPETITE)).isEqualTo("bold");
        assertThat(desc.disposition().primaryTerm(DispositionAxis.AUTONOMY)).isEqualTo("autonomous");
        assertThat(desc.disposition().primaryTerm(DispositionAxis.CONFLICT_MODE)).isEqualTo("competing");
    }

    @Test
    void disc_descriptor_has_no_default_profile_expansion() {
        var desc = VocabularyImbueFixtures.discDescriptor(DiscTerm.DOMINANCE, vocabRegistry);
        assertThat(desc.disposition().dispositionProfile()).hasSize(1);
    }

    // ── Conscientiousness ────────────────────────────────────────────

    @Test
    void conscientiousness_flat_axes_preserved() {
        var desc = VocabularyImbueFixtures.conscientiousnessDescriptor(
                Map.of(DispositionAxis.RULE_FOLLOWING, "strict",
                       DispositionAxis.RISK_APPETITE, "conservative",
                       DispositionAxis.AUTONOMY, "directed"));
        assertThat(desc.disposition().primaryTerm(DispositionAxis.RULE_FOLLOWING)).isEqualTo("strict");
        assertThat(desc.disposition().primaryTerm(DispositionAxis.RISK_APPETITE)).isEqualTo("conservative");
        assertThat(desc.disposition().primaryTerm(DispositionAxis.AUTONOMY)).isEqualTo("directed");
    }

    @Test
    void conscientiousness_prompt_contains_axis_values() {
        var desc = VocabularyImbueFixtures.conscientiousnessDescriptor(
                Map.of(DispositionAxis.RULE_FOLLOWING, "strict"));
        var prompt = renderer.render(desc, AgentPromptContext.forFormat(RenderFormat.MARKDOWN));
        assertThat(prompt.content()).containsIgnoringCase("strict");
    }

    // ── Pairwise: additive ───────────────────────────────────────────

    @Test
    void jungian_plus_belbin_both_signals_present() {
        var jungian = VocabularyImbueFixtures.jungianDescriptor(MbtiTypeTerm.ENTJ);
        var belbin = VocabularyImbueFixtures.belbinDescriptor(BelbinTerm.SHAPER);
        var comp = VocabularyImbueFixtures.composite(jungian, belbin);
        var derived = DescriptorCollector.deriveDispositionAxes(comp, vocabRegistry);
        // Jungian axes derived
        assertThat(derived.disposition().primaryTerm(DispositionAxis.SOCIAL_ORIENTATION)).isNotNull();
        assertThat(derived.disposition().primaryTerm(DispositionAxis.CONFLICT_MODE)).isNotNull();
        // Belbin slot preserved
        assertThat(derived.slotVocabulary()).isEqualTo("urn:casehub:vocab:belbin");
        assertThat(derived.slot()).isEqualTo("shaper");
    }

    @Test
    void belbin_plus_disc_both_signals_present() {
        var belbin = VocabularyImbueFixtures.belbinDescriptor(BelbinTerm.SHAPER);
        var disc = VocabularyImbueFixtures.discDescriptor(DiscTerm.DOMINANCE, vocabRegistry);
        var comp = VocabularyImbueFixtures.composite(disc, belbin);
        // DISC axes derived
        assertThat(comp.disposition().primaryTerm(DispositionAxis.RISK_APPETITE)).isEqualTo("bold");
        // Belbin slot preserved
        assertThat(comp.slotVocabulary()).isEqualTo("urn:casehub:vocab:belbin");
        assertThat(comp.slot()).isEqualTo("shaper");
    }

    @Test
    void belbin_plus_conscientiousness_both_signals_present() {
        var consc = VocabularyImbueFixtures.conscientiousnessDescriptor(
                Map.of(DispositionAxis.RULE_FOLLOWING, "strict"));
        var belbin = VocabularyImbueFixtures.belbinDescriptor(BelbinTerm.TEAMWORKER);
        var comp = VocabularyImbueFixtures.composite(consc, belbin);
        assertThat(comp.disposition().primaryTerm(DispositionAxis.RULE_FOLLOWING)).isEqualTo("strict");
        assertThat(comp.slotVocabulary()).isEqualTo("urn:casehub:vocab:belbin");
        assertThat(comp.slot()).isEqualTo("teamworker");
    }

    // ── Pairwise: redundant ──────────────────────────────────────────

    @Test
    void jungian_plus_disc_axes_conflict() {
        var jungian = VocabularyImbueFixtures.jungianDescriptor(MbtiTypeTerm.ENTJ);
        var jungianDerived = DescriptorCollector.deriveDispositionAxes(jungian, vocabRegistry);
        var disc = VocabularyImbueFixtures.discDescriptor(DiscTerm.DOMINANCE, vocabRegistry);
        // ENTJ: ruleFollowing=strict, DISC-D: ruleFollowing=flexible — different
        assertThat(jungianDerived.disposition().primaryTerm(DispositionAxis.RULE_FOLLOWING))
                .isNotEqualTo(disc.disposition().primaryTerm(DispositionAxis.RULE_FOLLOWING));
    }

    @Test
    void jungian_plus_conscientiousness_axes_differ() {
        var jungian = VocabularyImbueFixtures.jungianDescriptor(MbtiTypeTerm.ENTJ);
        var jungianDerived = DescriptorCollector.deriveDispositionAxes(jungian, vocabRegistry);
        var consc = VocabularyImbueFixtures.conscientiousnessDescriptor(
                Map.of(DispositionAxis.RULE_FOLLOWING, "flexible",
                       DispositionAxis.RISK_APPETITE, "bold"));
        // ENTJ derives strict, conscientiousness set to flexible — different
        assertThat(jungianDerived.disposition().primaryTerm(DispositionAxis.RULE_FOLLOWING))
                .isNotEqualTo(consc.disposition().primaryTerm(DispositionAxis.RULE_FOLLOWING));
    }

    @Test
    void disc_plus_conscientiousness_axes_differ() {
        var disc = VocabularyImbueFixtures.discDescriptor(DiscTerm.DOMINANCE, vocabRegistry);
        var consc = VocabularyImbueFixtures.conscientiousnessDescriptor(
                Map.of(DispositionAxis.RULE_FOLLOWING, "strict"));
        // DISC-D: ruleFollowing=flexible, conscientiousness: strict — different
        assertThat(disc.disposition().primaryTerm(DispositionAxis.RULE_FOLLOWING))
                .isNotEqualTo(consc.disposition().primaryTerm(DispositionAxis.RULE_FOLLOWING));
    }
}
```

Use `ide_create_file`.

- [ ] **Step 2: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -Dtest=VocabularyImbueStructuralTest -f /Users/mdproctor/claude/casehub/eidos/pom.xml`
Expected: PASS — all tests green. Some prompt-content tests may need adjustment based on actual rendered output format.

- [ ] **Step 3: Fix any failing assertions**

Adjust prompt-content assertions based on actual rendered output. The renderer may use different casing or formatting.

- [ ] **Step 4: Run full eval module tests (non-eval)**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -f /Users/mdproctor/claude/casehub/eidos/pom.xml`
Expected: PASS — no regressions.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add eval/src/test/java/io/casehub/eidos/eval/VocabularyImbueStructuralTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "test(#122): VocabularyImbueStructuralTest — structural verification for all 4 vocabularies + 6 pairwise combinations"
```

---

### Task 4: VocabularyImbueEvalTest — Layer 2

**Files:**
- Create: `eval/src/test/java/io/casehub/eidos/eval/VocabularyImbueEvalTest.java`

**Interfaces:**
- Consumes: `VocabularyImbueFixtures.*`, `MbtiAlignmentJudge.evaluate()`, `FunctionActivationJudge.evaluate()`, `PersonalityEvolutionJudge.evaluate()`, `DispositionPresenceJudge.evaluate()`, `SystemPromptRenderer.render()`
- Produces: No public API — test-only class

- [ ] **Step 1: Write eval test class**

```java
package io.casehub.eidos.eval;

import io.casehub.eidos.api.*;
import io.casehub.eidos.api.SystemPromptRenderer.RenderFormat;
import io.casehub.eidos.eval.FunctionActivationJudge.FunctionScenario;
import io.casehub.eidos.runtime.registrar.DescriptorCollector;
import io.casehub.eidos.vocab.BelbinTerm;
import io.casehub.eidos.vocab.DiscTerm;
import io.casehub.eidos.vocab.MbtiTypeTerm;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
@Tag("eval")
class VocabularyImbueEvalTest {

    @Inject SystemPromptRenderer renderer;
    @Inject VocabularyRegistry vocabRegistry;
    @Inject MbtiAlignmentJudge mbtiJudge;
    @Inject FunctionActivationJudge functionJudge;
    @Inject PersonalityEvolutionJudge evolutionJudge;
    @Inject DispositionPresenceJudge presenceJudge;
    @Inject DispositionSignalStore signalStore;

    static final List<FunctionScenario> ENTJ_SCENARIOS = List.of(
            new FunctionScenario("te",
                    "You must organize a team to complete a time-critical project. Describe your approach."),
            new FunctionScenario("ni",
                    "You notice a pattern in recent events that others have missed. What do you see and what does it mean?")
    );

    private String renderPrompt(AgentDescriptor descriptor) {
        var derived = DescriptorCollector.deriveDispositionAxes(descriptor, vocabRegistry);
        return renderer.render(derived, AgentPromptContext.forFormat(RenderFormat.MARKDOWN)).content();
    }

    // ── Single vocabulary: Jungian ───────────────────────────────────

    @Test
    void jungian_entj_mbti_alignment() {
        var prompt = renderPrompt(VocabularyImbueFixtures.jungianDescriptor(MbtiTypeTerm.ENTJ));
        var result = mbtiJudge.evaluate(prompt, "ENTJ");
        assertThat(result.overallAligned()).isTrue();
    }

    @Test
    void jungian_entj_function_activation() {
        var prompt = renderPrompt(VocabularyImbueFixtures.jungianDescriptor(MbtiTypeTerm.ENTJ));
        var result = functionJudge.evaluate(prompt, "ENTJ", ENTJ_SCENARIOS);
        assertThat(result.taa()).isGreaterThanOrEqualTo(0.5);
    }

    @Test
    void jungian_intp_personality_evolution() {
        var descriptor = VocabularyImbueFixtures.jungianDescriptor(MbtiTypeTerm.INTP);
        var derived = DescriptorCollector.deriveDispositionAxes(descriptor, vocabRegistry);
        var result = evolutionJudge.evaluate(derived, "ne", 4);
        assertThat(result.evolutionType()).isNotNull();
    }

    // ── Single vocabulary: Belbin ────────────────────────────────────

    @Test
    void belbin_shaper_presence() {
        var prompt = renderPrompt(VocabularyImbueFixtures.belbinDescriptor(BelbinTerm.SHAPER));
        var result = presenceJudge.evaluate(prompt, BelbinTerm.SHAPER.label(),
                BelbinTerm.SHAPER.description());
        assertThat(result.aligned()).isTrue();
    }

    @Test
    void belbin_teamworker_presence() {
        var prompt = renderPrompt(VocabularyImbueFixtures.belbinDescriptor(BelbinTerm.TEAMWORKER));
        var result = presenceJudge.evaluate(prompt, BelbinTerm.TEAMWORKER.label(),
                BelbinTerm.TEAMWORKER.description());
        assertThat(result.aligned()).isTrue();
    }

    // ── Single vocabulary: DISC ──────────────────────────────────────

    @Test
    void disc_dominance_presence() {
        var prompt = renderPrompt(VocabularyImbueFixtures.discDescriptor(DiscTerm.DOMINANCE, vocabRegistry));
        var result = presenceJudge.evaluate(prompt, DiscTerm.DOMINANCE.label(),
                DiscTerm.DOMINANCE.description());
        assertThat(result.aligned()).isTrue();
    }

    // ── Single vocabulary: Conscientiousness ─────────────────────────

    @Test
    void conscientiousness_strict_presence() {
        var prompt = renderPrompt(VocabularyImbueFixtures.conscientiousnessDescriptor(
                Map.of(DispositionAxis.RULE_FOLLOWING, "strict",
                       DispositionAxis.RISK_APPETITE, "conservative")));
        var result = presenceJudge.evaluate(prompt, "Strict rule-following",
                "Follows rules and procedures strictly, values compliance and predictability");
        assertThat(result.aligned()).isTrue();
    }

    // ── Pairwise: Jungian + Belbin ───────────────────────────────────

    @Test
    void composite_jungian_belbin_mbti_aligned() {
        var comp = VocabularyImbueFixtures.composite(
                VocabularyImbueFixtures.jungianDescriptor(MbtiTypeTerm.ENTJ),
                VocabularyImbueFixtures.belbinDescriptor(BelbinTerm.SHAPER));
        var prompt = renderPrompt(comp);
        var mbtiResult = mbtiJudge.evaluate(prompt, "ENTJ");
        assertThat(mbtiResult.overallAligned()).isTrue();
    }

    @Test
    void composite_jungian_belbin_shaper_present() {
        var comp = VocabularyImbueFixtures.composite(
                VocabularyImbueFixtures.jungianDescriptor(MbtiTypeTerm.ENTJ),
                VocabularyImbueFixtures.belbinDescriptor(BelbinTerm.SHAPER));
        var prompt = renderPrompt(comp);
        var presenceResult = presenceJudge.evaluate(prompt, BelbinTerm.SHAPER.label(),
                BelbinTerm.SHAPER.description());
        assertThat(presenceResult.aligned()).isTrue();
    }

    // ── Pairwise: Belbin + DISC ──────────────────────────────────────

    @Test
    void composite_belbin_disc_both_present() {
        var comp = VocabularyImbueFixtures.composite(
                VocabularyImbueFixtures.discDescriptor(DiscTerm.DOMINANCE, vocabRegistry),
                VocabularyImbueFixtures.belbinDescriptor(BelbinTerm.SHAPER));
        var prompt = renderPrompt(comp);
        var shaperResult = presenceJudge.evaluate(prompt, BelbinTerm.SHAPER.label(),
                BelbinTerm.SHAPER.description());
        var dominanceResult = presenceJudge.evaluate(prompt, DiscTerm.DOMINANCE.label(),
                DiscTerm.DOMINANCE.description());
        assertThat(shaperResult.aligned()).isTrue();
        assertThat(dominanceResult.aligned()).isTrue();
    }
}
```

Use `ide_create_file`.

- [ ] **Step 2: Verify the test compiles**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -pl eval -f /Users/mdproctor/claude/casehub/eidos/pom.xml`
Expected: BUILD SUCCESS — all types resolve.

Note: These tests are `@Tag("eval")` and excluded from normal builds. They run only under `-Peval` with a live LLM. Do NOT run them as part of this implementation — verify compilation only.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add eval/src/test/java/io/casehub/eidos/eval/VocabularyImbueEvalTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "test(#122): VocabularyImbueEvalTest — LLM-backed eval for single-vocab and pairwise composition"
```

---

### Task 5: Final verification + diagnostics

**Files:**
- No new files

- [ ] **Step 1: Run all eval module tests (non-eval)**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -f /Users/mdproctor/claude/casehub/eidos/pom.xml`
Expected: PASS — all structural tests + existing tests green, no regressions.

- [ ] **Step 2: Run IDE diagnostics on new files**

Run `ide_diagnostics` on each new file:
- `eval/src/main/java/io/casehub/eidos/eval/DispositionPresenceResult.java`
- `eval/src/main/java/io/casehub/eidos/eval/DispositionPresenceJudge.java`
- `eval/src/test/java/io/casehub/eidos/eval/DispositionPresenceJudgeTest.java`
- `eval/src/test/java/io/casehub/eidos/eval/VocabularyImbueFixtures.java`
- `eval/src/test/java/io/casehub/eidos/eval/VocabularyImbueFixturesTest.java`
- `eval/src/test/java/io/casehub/eidos/eval/VocabularyImbueStructuralTest.java`
- `eval/src/test/java/io/casehub/eidos/eval/VocabularyImbueEvalTest.java`

Expected: No errors.

- [ ] **Step 3: Verify full project build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -f /Users/mdproctor/claude/casehub/eidos/pom.xml -DskipTests`
Expected: BUILD SUCCESS — no compilation errors across modules.
