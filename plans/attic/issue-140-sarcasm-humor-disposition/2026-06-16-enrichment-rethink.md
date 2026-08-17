# Enrichment Rethink Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Narrow enrichment to disposition+goal only, fix the mechanics that prevented it from running, add `briefing` as a first-class vocabulary-gap answer, and replace a broken proximity eval with one that measures the right thing.

**Architecture:** Five changes across three independent GitHub issues. Issues A (enrichment-mechanics) must complete first; B (briefing-field) and C (proximity-eval-redesign) can proceed in parallel after A. Full validation of B's effect on FULL-LOSS cases requires C.

**Tech Stack:** Java 21, Quarkus 3.32.2, LangChain4j 1.14.1, JUnit 5 + AssertJ, Jackson, Flyway

---

## File Map

**Issue A — enrichment-mechanics (Changes 1+2+3)**

| Action | Path |
|--------|------|
| Create | `runtime/src/main/java/io/casehub/eidos/runtime/renderer/JsonExtractionUtil.java` |
| Modify | `runtime/src/main/java/io/casehub/eidos/runtime/renderer/SemanticEnrichment.java` |
| Modify | `runtime/src/main/java/io/casehub/eidos/runtime/renderer/SemanticEnrichmentStep.java` |
| Modify | `runtime/src/main/java/io/casehub/eidos/runtime/renderer/ReactiveSemanticEnrichmentStep.java` |
| Modify | `runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosRenderPipeline.java` |
| Modify | `runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosSystemPromptRenderer.java` |
| Modify | `runtime/src/main/java/io/casehub/eidos/runtime/renderer/DefaultReactiveSystemPromptRenderer.java` |
| Modify | `runtime/src/test/java/io/casehub/eidos/runtime/renderer/SemanticEnrichmentStepTest.java` |
| Modify | `runtime/src/test/java/io/casehub/eidos/runtime/renderer/ReactiveSemanticEnrichmentStepTest.java` |
| Modify | `runtime/src/test/java/io/casehub/eidos/runtime/renderer/EidosSystemPromptRendererTest.java` |

**Issue B — briefing-field (Change 4)**

| Action | Path |
|--------|------|
| Modify | `api/src/main/java/io/casehub/eidos/api/AgentDescriptor.java` |
| Modify | `api/src/main/java/io/casehub/eidos/api/AgentDescriptorValidator.java` |
| Modify | `runtime/src/main/resources/db/eidos/migration/V1__initial_schema.sql` |
| Modify | `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentDescriptorEntity.java` |
| Modify | `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentDescriptorMapper.java` |
| Modify | `runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosRenderPipeline.java` |
| Modify | `api/src/test/java/io/casehub/eidos/api/AgentDescriptorValidatorTest.java` |
| Modify | `runtime/src/test/java/io/casehub/eidos/runtime/renderer/EidosSystemPromptRendererTest.java` |

**Issue C — proximity-eval-redesign (Change 5)**

| Action | Path |
|--------|------|
| Modify | `eval/src/main/java/io/casehub/eidos/eval/ProximityJudge.java` |
| Modify | `eval/src/test/java/io/casehub/eidos/eval/ProximityJudgeTest.java` |

---

## Issue A — enrichment-mechanics

### Task A1: JsonExtractionUtil

**Files:**
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/renderer/JsonExtractionUtil.java`
- Test: create inline in `SemanticEnrichmentStepTest.java` (no separate test file needed — package-private utility)

- [ ] **Step 1: Create JsonExtractionUtil**

```java
// runtime/src/main/java/io/casehub/eidos/runtime/renderer/JsonExtractionUtil.java
package io.casehub.eidos.runtime.renderer;

class JsonExtractionUtil {

    private JsonExtractionUtil() {}

    /**
     * Extracts a JSON object from LLM output that may contain code fences or prose preamble.
     * Strips markdown code fences (```json...``` or ```...```).
     * Finds the outermost {...} block if preceded by prose preamble.
     */
    static String extractJson(final String text) {
        String s = text.strip();
        // Strip markdown code fences
        if (s.startsWith("```")) {
            final int nl = s.indexOf('\n');
            if (nl != -1) s = s.substring(nl + 1).strip();
            if (s.endsWith("```")) s = s.substring(0, s.length() - 3).stripTrailing();
        }
        // Extract from prose preamble (finds outermost {...})
        final int first = s.indexOf('{');
        final int last = s.lastIndexOf('}');
        if (first > 0 && last > first) s = s.substring(first, last + 1);
        return s;
    }
}
```

- [ ] **Step 2: Run runtime tests to verify they still pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=SemanticEnrichmentStepTest -f /Users/mdproctor/claude/casehub/eidos/pom.xml
```

Expected: existing tests pass (no references to JsonExtractionUtil yet).

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add runtime/src/main/java/io/casehub/eidos/runtime/renderer/JsonExtractionUtil.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#XX): JsonExtractionUtil — extractJson for code fences and prose preamble Refs #XX"
```

---

### Task A2: Narrow SemanticEnrichment to 2 fields

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/renderer/SemanticEnrichment.java`

The current record has 6 fields. After this change it has 2.

- [ ] **Step 1: Replace the record**

Replace the entire file content:

```java
// runtime/src/main/java/io/casehub/eidos/runtime/renderer/SemanticEnrichment.java
package io.casehub.eidos.runtime.renderer;

import java.util.Optional;

record SemanticEnrichment(
        Optional<String> dispositionNarrative,
        Optional<String> goalNarrative
) {}
```

- [ ] **Step 2: Verify compile failures surface (expected)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -f /Users/mdproctor/claude/casehub/eidos/pom.xml 2>&1 | grep "error:" | head -20
```

Expected: compile errors in `SemanticEnrichmentStep.java`, `ReactiveSemanticEnrichmentStep.java`, `EidosRenderPipeline.java`, and test files — all referencing the removed fields. These are fixed in subsequent tasks.

---

### Task A3: Update SemanticEnrichmentStep — extractJson + retry

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/renderer/SemanticEnrichmentStep.java`
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/renderer/SemanticEnrichmentStepTest.java`

- [ ] **Step 1: Rewrite SemanticEnrichmentStep**

Replace the entire file:

```java
// runtime/src/main/java/io/casehub/eidos/runtime/renderer/SemanticEnrichmentStep.java
package io.casehub.eidos.runtime.renderer;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import dev.langchain4j.data.message.SystemMessage;
import dev.langchain4j.data.message.UserMessage;
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.model.chat.request.ChatRequest;
import org.jboss.logging.Logger;

import java.util.Optional;

class SemanticEnrichmentStep {

    private static final Logger log = Logger.getLogger(SemanticEnrichmentStep.class);

    private final ObjectMapper mapper;

    SemanticEnrichmentStep(final ObjectMapper mapper) {
        this.mapper = mapper;
    }

    Optional<SemanticEnrichment> enrich(final ChatModel llm, final ObjectNode payload) {
        try {
            final ChatRequest request = ChatRequest.builder()
                    .messages(
                            SystemMessage.from(EidosRenderPipeline.PROMPT_TEMPLATE),
                            UserMessage.from(mapper.writeValueAsString(payload))
                    )
                    .responseFormat(EidosRenderPipeline.RESPONSE_FORMAT)
                    .build();
            try {
                return Optional.of(parse(llm.chat(request).aiMessage().text()));
            } catch (final JsonProcessingException first) {
                log.warnf("Enrichment: non-JSON response, retrying (%s)", first.getMessage());
                return Optional.of(parse(llm.chat(request).aiMessage().text()));
            }
        } catch (final Exception e) {
            log.warnf("Semantic enrichment failed (%s), falling back to structural", e.getMessage());
            return Optional.empty();
        }
    }

    private SemanticEnrichment parse(final String json) throws JsonProcessingException {
        final JsonNode node = mapper.readTree(JsonExtractionUtil.extractJson(json));
        return new SemanticEnrichment(
                optional(node, "dispositionNarrative"),
                optional(node, "goalNarrative")
        );
    }

    private static Optional<String> optional(final JsonNode node, final String field) {
        final JsonNode n = node.get(field);
        if (n == null || n.isNull()) return Optional.empty();
        final String v = n.asText("").strip();
        return v.isEmpty() ? Optional.empty() : Optional.of(v);
    }
}
```

- [ ] **Step 2: Rewrite SemanticEnrichmentStepTest**

Replace the entire file:

```java
// runtime/src/test/java/io/casehub/eidos/runtime/renderer/SemanticEnrichmentStepTest.java
package io.casehub.eidos.runtime.renderer;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import dev.langchain4j.data.message.AiMessage;
import dev.langchain4j.data.message.UserMessage;
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.model.chat.request.ChatRequest;
import dev.langchain4j.model.chat.response.ChatResponse;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;

class SemanticEnrichmentStepTest {

    static final ObjectMapper MAPPER = new ObjectMapper();
    SemanticEnrichmentStep step;

    @BeforeEach
    void setUp() {
        step = new SemanticEnrichmentStep(MAPPER);
    }

    static ObjectNode payload() {
        final ObjectNode node = MAPPER.createObjectNode();
        node.put("name", "Test Agent");
        node.put("slot", "reviewer");
        final ObjectNode disp = node.putObject("disposition");
        disp.put("riskAppetite", "bold");
        disp.put("canDelegate", false);
        return node;
    }

    static ChatModel mockReturning(final String json) {
        return new ChatModel() {
            @Override public ChatResponse doChat(final ChatRequest r) {
                return ChatResponse.builder().aiMessage(AiMessage.from(json)).build();
            }
        };
    }

    @Test
    void parses_disposition_and_goal() {
        final String json = """
            {"dispositionNarrative":"You approve boldly.",
             "goalNarrative":"Review PR #42."}""";

        final Optional<SemanticEnrichment> result = step.enrich(mockReturning(json), payload());

        assertThat(result).isPresent();
        assertThat(result.get().dispositionNarrative()).contains("You approve boldly.");
        assertThat(result.get().goalNarrative()).contains("Review PR #42.");
    }

    @Test
    void blank_optional_fields_become_empty() {
        final String json = """
            {"dispositionNarrative":"","goalNarrative":"  "}""";

        final Optional<SemanticEnrichment> result = step.enrich(mockReturning(json), payload());

        assertThat(result.get().dispositionNarrative()).isEmpty();
        assertThat(result.get().goalNarrative()).isEmpty();
    }

    @Test
    void json_wrapped_in_markdown_code_block_is_parsed() {
        final String json = """
            {"dispositionNarrative":"You approve boldly.","goalNarrative":""}""";
        final String wrapped = "```json\n" + json + "\n```";

        final Optional<SemanticEnrichment> result = step.enrich(mockReturning(wrapped), payload());

        assertThat(result).isPresent();
        assertThat(result.get().dispositionNarrative()).contains("You approve boldly.");
    }

    @Test
    void prose_preamble_before_json_is_stripped() {
        final String response = "Here is the JSON:\n{\"dispositionNarrative\":\"Bold.\",\"goalNarrative\":\"\"}";

        final Optional<SemanticEnrichment> result = step.enrich(mockReturning(response), payload());

        assertThat(result).isPresent();
        assertThat(result.get().dispositionNarrative()).contains("Bold.");
    }

    @Test
    void exception_from_llm_returns_empty() {
        final ChatModel throwing = new ChatModel() {
            @Override public ChatResponse doChat(final ChatRequest r) {
                throw new RuntimeException("Model unavailable");
            }
        };
        assertThat(step.enrich(throwing, payload())).isEmpty();
    }

    @Test
    void malformed_json_retries_then_falls_back_to_empty() {
        // Both attempts return malformed JSON — should fall back to empty
        assertThat(step.enrich(mockReturning("not json at all"), payload())).isEmpty();
    }

    @Test
    void system_message_equals_prompt_template() {
        final String[] captured = {""};
        final ChatModel capturingMock = new ChatModel() {
            @Override public ChatResponse doChat(final ChatRequest r) {
                r.messages().stream()
                    .filter(m -> m instanceof dev.langchain4j.data.message.SystemMessage)
                    .map(m -> ((dev.langchain4j.data.message.SystemMessage) m).text())
                    .findFirst().ifPresent(t -> captured[0] = t);
                return ChatResponse.builder().aiMessage(AiMessage.from(
                    "{\"dispositionNarrative\":\"\",\"goalNarrative\":\"\"}")).build();
            }
        };
        step.enrich(capturingMock, payload());
        assertThat(captured[0]).isEqualTo(EidosRenderPipeline.PROMPT_TEMPLATE);
    }

    @Test
    void chat_request_has_response_format() {
        final boolean[] hasFormat = {false};
        final ChatModel checkingMock = new ChatModel() {
            @Override public ChatResponse doChat(final ChatRequest r) {
                hasFormat[0] = r.parameters() != null && r.parameters().responseFormat() != null;
                return ChatResponse.builder().aiMessage(AiMessage.from(
                    "{\"dispositionNarrative\":\"\",\"goalNarrative\":\"\"}")).build();
            }
        };
        step.enrich(checkingMock, payload());
        assertThat(hasFormat[0]).isTrue();
    }
}
```

- [ ] **Step 3: Run tests (partial — EidosRenderPipeline still broken)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=SemanticEnrichmentStepTest -f /Users/mdproctor/claude/casehub/eidos/pom.xml 2>&1 | tail -20
```

Expected: compile errors from other files that still use 6-field record. SemanticEnrichmentStepTest itself should be clean once the compile errors in other files are fixed.

---

### Task A4: Update ReactiveSemanticEnrichmentStep

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/renderer/ReactiveSemanticEnrichmentStep.java`
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/renderer/ReactiveSemanticEnrichmentStepTest.java`

- [ ] **Step 1: Update parseOrEmpty() — extractJson + 2-field constructor**

In `ReactiveSemanticEnrichmentStep.java`, replace `parseOrEmpty()`:

```java
private Optional<SemanticEnrichment> parseOrEmpty(final String json) {
    try {
        final JsonNode node = mapper.readTree(JsonExtractionUtil.extractJson(json));
        return Optional.of(new SemanticEnrichment(
                optional(node, "dispositionNarrative"),
                optional(node, "goalNarrative")
        ));
    } catch (final Exception e) {
        log.warn("Reactive enrichment: parse failed (" + e.getMessage() + "), falling back");
        return Optional.empty();
    }
}
```

(Keep the `optional()` helper method unchanged — it's already in the class.)

- [ ] **Step 2: Update ReactiveSemanticEnrichmentStepTest**

Replace the file:

```java
// runtime/src/test/java/io/casehub/eidos/runtime/renderer/ReactiveSemanticEnrichmentStepTest.java
package io.casehub.eidos.runtime.renderer;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import dev.langchain4j.data.message.AiMessage;
import dev.langchain4j.model.chat.StreamingChatModel;
import dev.langchain4j.model.chat.request.ChatRequest;
import dev.langchain4j.model.chat.response.ChatResponse;
import dev.langchain4j.model.chat.response.StreamingChatResponseHandler;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;

class ReactiveSemanticEnrichmentStepTest {

    static final ObjectMapper MAPPER = new ObjectMapper();
    static final String VALID_JSON = """
            {"dispositionNarrative":"You approve boldly.",
             "goalNarrative":"Review PR #42."}""";

    ReactiveSemanticEnrichmentStep step;

    @BeforeEach
    void setUp() {
        step = new ReactiveSemanticEnrichmentStep(MAPPER);
    }

    static ObjectNode payload() {
        final ObjectNode node = MAPPER.createObjectNode();
        node.put("name", "Test Agent");
        node.put("slot", "reviewer");
        return node;
    }

    static StreamingChatModel successMock() {
        return (request, handler) -> handler.onCompleteResponse(
            ChatResponse.builder().aiMessage(AiMessage.from(VALID_JSON)).build());
    }

    static StreamingChatModel errorMock() {
        return (request, handler) -> handler.onError(new RuntimeException("model unavailable"));
    }

    static StreamingChatModel malformedJsonMock() {
        return (request, handler) -> handler.onCompleteResponse(
            ChatResponse.builder().aiMessage(AiMessage.from("not valid json")).build());
    }

    @Test
    void completes_with_disposition_and_goal_when_llm_succeeds() {
        final Optional<SemanticEnrichment> result =
            step.enrich(successMock(), payload()).await().indefinitely();

        assertThat(result).isPresent();
        assertThat(result.get().dispositionNarrative()).contains("You approve boldly.");
        assertThat(result.get().goalNarrative()).contains("Review PR #42.");
    }

    @Test
    void falls_back_to_empty_when_llm_fires_on_error() {
        assertThat(step.enrich(errorMock(), payload()).await().indefinitely()).isEmpty();
    }

    @Test
    void falls_back_to_empty_when_parse_fails() {
        assertThat(step.enrich(malformedJsonMock(), payload()).await().indefinitely()).isEmpty();
    }

    @Test
    void invokes_streaming_api_not_blocking_overload() {
        final boolean[] streamingCalled = {false};
        final StreamingChatModel trackingMock = (request, handler) -> {
            streamingCalled[0] = true;
            handler.onCompleteResponse(
                ChatResponse.builder().aiMessage(AiMessage.from(VALID_JSON)).build());
        };
        step.enrich(trackingMock, payload()).await().indefinitely();
        assertThat(streamingCalled[0]).isTrue();
    }

    @Test
    void returns_non_null_uni_for_hanging_provider() {
        final StreamingChatModel hangingMock = (request, handler) -> { /* never fires */ };
        assertThat(step.enrich(hangingMock, payload())).isNotNull();
    }
}
```

---

### Task A5: Decompose EidosRenderPipeline assembly + selective override

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosRenderPipeline.java`

This task handles the `assembleMarkdown` and `assembleProse` changes only. The PROMPT_TEMPLATE, RESPONSE_FORMAT, and `buildEnrichmentPayload` changes come in Task A6.

- [ ] **Step 1: Extract section methods from assembleMarkdownStructural**

In `EidosRenderPipeline.java`, replace `assembleMarkdownStructural` with five focused methods. The existing code in `assembleMarkdownStructural` is split into:

```java
// Role section — always structural
private void assembleMarkdownRole(final StringBuilder sb, final AgentDescriptor descriptor) {
    if (descriptor.slot() != null) {
        sb.append("\n## Role\n");
        descriptor.vocabUriForSlot().ifPresentOrElse(
            uri -> vocab.resolve(uri, descriptor.slot()).ifPresentOrElse(
                term -> {
                    if (term.label() != null)       sb.append(term.label()).append("\n");
                    if (term.description() != null) sb.append(term.description()).append("\n");
                },
                () -> sb.append(descriptor.slot()).append("\n")
            ),
            () -> sb.append(descriptor.slot()).append("\n")
        );
    }
}

// Capabilities section — always structural
private void assembleMarkdownCapabilities(final StringBuilder sb, final AgentDescriptor descriptor) {
    if (descriptor.capabilities() != null && !descriptor.capabilities().isEmpty()) {
        sb.append("\n## Capabilities\n");
        for (final AgentCapability cap : descriptor.capabilities()) {
            sb.append("- **").append(cap.name()).append("**");
            if (cap.inputTypes() != null && !cap.inputTypes().isEmpty())
                sb.append(": accepts ").append(String.join(", ", cap.inputTypes()));
            if (cap.outputTypes() != null && !cap.outputTypes().isEmpty())
                sb.append(" → ").append(String.join(", ", cap.outputTypes()));
            sb.append("\n");
        }
    }
}

// Disposition section — structural rendering
private void assembleMarkdownDisposition(final StringBuilder sb, final AgentDescriptor descriptor) {
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
}

// Data handling — always structural
private void assembleMarkdownDataHandling(final StringBuilder sb, final AgentDescriptor descriptor) {
    if (descriptor.jurisdiction() != null || descriptor.dataHandlingPolicy() != null) {
        sb.append("\n## Data Handling\n");
        if (descriptor.jurisdiction() != null)
            sb.append("Jurisdiction: ").append(descriptor.jurisdiction()).append("\n");
        if (descriptor.dataHandlingPolicy() != null)
            sb.append("Policy: ").append(descriptor.dataHandlingPolicy()).append("\n");
    }
}

// Goal section — structural rendering
private void assembleMarkdownGoal(final StringBuilder sb, final AgentPromptContext context) {
    context.goal().ifPresent(goal -> {
        sb.append("\n## Current Goal\n");
        sb.append(goal.description()).append("\n");
        if (!goal.subGoals().isEmpty()) {
            goal.subGoals().forEach(sub -> sb.append("- ").append(sub).append("\n"));
        }
        if (goal.caseRef() != null) sb.append("Case: ").append(goal.caseRef()).append("\n");
    });
}
```

Then replace `assembleMarkdownStructural` to be a simple composition:

```java
private void assembleMarkdownStructural(final StringBuilder sb,
                                         final AgentDescriptor descriptor,
                                         final AgentPromptContext context) {
    assembleMarkdownRole(sb, descriptor);
    assembleMarkdownCapabilities(sb, descriptor);
    assembleMarkdownDisposition(sb, descriptor);
    assembleMarkdownDataHandling(sb, descriptor);
    assembleMarkdownGoal(sb, context);
}
```

- [ ] **Step 2: Change assembleMarkdown to selective override**

Replace `assembleMarkdown` body (keep header block unchanged, change the binary gate):

```java
private String assembleMarkdown(final Optional<SemanticEnrichment> enrichment,
                                 final AgentDescriptor descriptor,
                                 final AgentPromptContext context) {
    final var sb = new StringBuilder();

    // Header — always structural
    sb.append("# ").append(descriptor.name()).append("\n");
    sb.append("**Agent ID:** ").append(descriptor.agentId());
    final String model = combinedModel(descriptor);
    if (model != null)                   sb.append("  **Model:** ").append(model);
    if (descriptor.provider() != null)   sb.append("  **Provider:** ").append(descriptor.provider());
    sb.append("\n");

    // Role — always structural
    assembleMarkdownRole(sb, descriptor);

    // Capabilities — always structural
    assembleMarkdownCapabilities(sb, descriptor);

    // Disposition — enriched OR structural (selective override)
    if (enrichment.isPresent() && enrichment.get().dispositionNarrative().isPresent()) {
        sb.append("\n## How You Operate\n")
          .append(enrichment.get().dispositionNarrative().get()).append("\n");
    } else {
        assembleMarkdownDisposition(sb, descriptor);
    }

    // Data Handling — always structural
    assembleMarkdownDataHandling(sb, descriptor);

    // Goal — enriched OR structural (selective override)
    if (enrichment.isPresent() && enrichment.get().goalNarrative().isPresent()) {
        sb.append("\n## Current Goal\n")
          .append(enrichment.get().goalNarrative().get()).append("\n");
    } else {
        assembleMarkdownGoal(sb, context);
    }

    // Resources — always structural
    if (!context.resources().isEmpty()) {
        sb.append("\n## Resources\n");
        for (final Resource r : context.resources()) {
            sb.append("- **").append(r.label() != null ? r.label() : r.uri()).append("**: ").append(r.uri());
            if (r.type() != null) sb.append(" (").append(r.type()).append(")");
            sb.append("\n");
        }
    }

    // Situational context — always structural
    if (context.situationalContext() != null) {
        sb.append("\n## Context\n").append(context.situationalContext()).append("\n");
    }

    return sb.toString().trim();
}
```

- [ ] **Step 3: Change assembleProse to selective override**

Replace `assembleProse` body:

```java
private String assembleProse(final Optional<SemanticEnrichment> enrichment,
                              final AgentDescriptor descriptor,
                              final AgentPromptContext context) {
    final var sb = new StringBuilder();

    // Identity + Role — always structural (dense prose, no headers)
    sb.append(descriptor.name());
    if (descriptor.slot() != null) sb.append(", ").append(descriptor.slot());
    sb.append(".");
    if (descriptor.version() != null) sb.append(" Version ").append(descriptor.version()).append(".");
    sb.append("\n");

    // Capabilities — always structural
    if (descriptor.capabilities() != null && !descriptor.capabilities().isEmpty()) {
        sb.append("\nCapabilities: ");
        final var names = descriptor.capabilities().stream()
                .map(AgentCapability::name)
                .collect(Collectors.joining(", "));
        sb.append(names).append(".\n");
    }

    // Disposition — enriched OR structural (selective override)
    if (enrichment.isPresent() && enrichment.get().dispositionNarrative().isPresent()) {
        sb.append("\n").append(enrichment.get().dispositionNarrative().get()).append("\n");
    } else if (descriptor.disposition() != null) {
        final AgentDisposition d = descriptor.disposition();
        sb.append("\nOperating style:");
        for (DispositionAxis axis : DispositionAxis.values()) {
            d.get(axis).ifPresent(raw ->
                sb.append(" ").append(axisLabel(axis)).append(": ")
                  .append(resolveAxisDisplay(axis, raw, descriptor)).append("."));
        }
        sb.append(" Can delegate: ").append(d.delegation() ? "yes" : "no").append(".\n");
    }

    // Goal — enriched OR structural (selective override)
    if (enrichment.isPresent() && enrichment.get().goalNarrative().isPresent()) {
        sb.append("\n").append(enrichment.get().goalNarrative().get()).append("\n");
    } else {
        context.goal().ifPresent(goal -> {
            sb.append("\nGoal: ").append(goal.description()).append(".\n");
            if (!goal.subGoals().isEmpty()) {
                sb.append("Sub-goals: ").append(String.join(", ", goal.subGoals())).append(".\n");
            }
        });
    }

    // Resources — always structural
    if (!context.resources().isEmpty()) {
        sb.append("\nResources: ");
        final var resources = context.resources().stream()
                .map(r -> (r.label() != null ? r.label() : r.uri()) + " (" + r.uri() + ")")
                .collect(Collectors.joining(", "));
        sb.append(resources).append(".\n");
    }

    if (context.situationalContext() != null) {
        sb.append("\n").append(context.situationalContext()).append("\n");
    }

    return sb.toString().trim();
}
```

---

### Task A6: Update EidosRenderPipeline — PROMPT_TEMPLATE, RESPONSE_FORMAT, buildEnrichmentPayload

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosRenderPipeline.java`

- [ ] **Step 1: Rewrite PROMPT_TEMPLATE**

Replace the `PROMPT_TEMPLATE` constant:

```java
static final String PROMPT_TEMPLATE = """
        You are writing disposition and goal narratives for an AI agent's system prompt.

        Given the agent context in JSON, produce a JSON object with prose for two fields.
        Write in second person, addressing the agent directly.

        The payload contains:
        - name: the agent's name
        - slot: the agent's role type
        - slotLabel, slotDescription, slotVocabularyName: vocabulary-resolved role context (when present)
        - disposition: an object with one key per axis, each having a "value" field and optionally \
        "label", "vocabularyName"
        - goal: the current task (when present)
        - briefing: additional behavioral principles not expressible as axes (when present)

        FIELDS:
        - dispositionNarrative (2-4 sentences): Use name and slot to frame the narrative for this \
        agent's specific role. Cover ALL disposition axes present — omitting any present axis is \
        incorrect. Use vocabulary framework language when vocabularyName is present on an axis. \
        Weave briefing principles naturally when present — do not quote verbatim. \
        Empty string if no disposition object is in the payload.
        - goalNarrative (1-3 sentences): The agent's current task and objectives in flowing prose. \
        Sub-goals as natural continuation, not bullets. Empty string if no goal is present.

        RULES:
        - Second person only: "You are...", "Your role is...", "You have...".
        - Plain prose. No markdown, no bullet points, no headers.
        - Be concise. Every sentence must carry information the agent needs to act on.
        - Return ONLY the JSON object. No explanation, no preamble, no code fences.""";
```

- [ ] **Step 2: Narrow RESPONSE_FORMAT_SCHEMA_DESCRIPTIONS and RESPONSE_FORMAT**

Replace `RESPONSE_FORMAT_SCHEMA_DESCRIPTIONS`:

```java
static final List<String> RESPONSE_FORMAT_SCHEMA_DESCRIPTIONS = List.of(
        "How the agent operates — role-specific, covering all declared disposition axes. " +
        "Use vocabulary framework language when present. 2-4 sentences. Empty string if no disposition.",
        "Current task and objectives in flowing prose. Empty string if no goal."
);
```

Replace `RESPONSE_FORMAT`:

```java
static final ResponseFormat RESPONSE_FORMAT = ResponseFormat.builder()
        .type(ResponseFormatType.JSON)
        .jsonSchema(JsonSchema.builder()
                .name("SemanticEnrichment")
                .rootElement(JsonObjectSchema.builder()
                        .addStringProperty("dispositionNarrative",
                                RESPONSE_FORMAT_SCHEMA_DESCRIPTIONS.get(0))
                        .addStringProperty("goalNarrative",
                                RESPONSE_FORMAT_SCHEMA_DESCRIPTIONS.get(1))
                        .required("dispositionNarrative", "goalNarrative")
                        .build())
                .build())
        .build();
```

Update `TEMPLATE_HASH` to cover A2A fields too (keep existing computation, it's already correct since TEMPLATE_HASH = fingerprint of PROMPT_TEMPLATE + A2A_PROMPT_TEMPLATE + all schema descriptions):

The TEMPLATE_HASH declaration is unchanged — it's a computed fingerprint that automatically invalidates the cache when PROMPT_TEMPLATE changes.

- [ ] **Step 3: Add buildEnrichmentPayload; delete buildLlmPayload**

Add the new method:

```java
/**
 * Focused payload for enrichment — disposition, goal, and role context only.
 * Excludes identity, capabilities, constraint — those sections render structurally.
 * name + slot + vocab labels provide role context for disposition prose.
 */
ObjectNode buildEnrichmentPayload(final ObjectNode descriptorNode,
                                   final ObjectNode contextNode) {
    final ObjectNode payload = mapper.createObjectNode();
    copyIfPresent(payload, descriptorNode, "name");
    copyIfPresent(payload, descriptorNode, "slot");
    copyIfPresent(payload, descriptorNode, "slotLabel");
    copyIfPresent(payload, descriptorNode, "slotDescription");
    copyIfPresent(payload, descriptorNode, "slotVocabularyName");
    copyIfPresent(payload, descriptorNode, "disposition");
    copyIfPresent(payload, contextNode,    "goal");
    copyIfPresent(payload, descriptorNode, "briefing");
    return payload;
}

private static void copyIfPresent(final ObjectNode dest, final ObjectNode src, final String key) {
    if (src != null && src.has(key)) dest.set(key, src.get(key).deepCopy());
}
```

Delete the old `buildLlmPayload` method entirely.

---

### Task A7: Update call sites — both renderers

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosSystemPromptRenderer.java`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/renderer/DefaultReactiveSystemPromptRenderer.java`
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/renderer/EidosSystemPromptRendererTest.java`

- [ ] **Step 1: Update EidosSystemPromptRenderer.renderFresh()**

In `renderFresh()`, change:
```java
// OLD
final var llmPayload = pipeline.buildLlmPayload(s1.descriptorNode(), s1.contextNode());
// NEW
final var llmPayload = pipeline.buildEnrichmentPayload(s1.descriptorNode(), s1.contextNode());
```

- [ ] **Step 2: Update DefaultReactiveSystemPromptRenderer.executeStagesTwoAndThree()**

In `executeStagesTwoAndThree()`, change:
```java
// OLD
final ObjectNode llmPayload = needsEnrichment
        ? pipeline.buildLlmPayload(s1.descriptorNode(), s1.contextNode())
        : null;
// NEW
final ObjectNode llmPayload = needsEnrichment
        ? pipeline.buildEnrichmentPayload(s1.descriptorNode(), s1.contextNode())
        : null;
```

- [ ] **Step 3: Update EidosSystemPromptRendererTest.LLM_JSON_RESPONSE**

The mock LLM now returns 2-field JSON. Update these constants at the top of the test class:

```java
/** JSON that SemanticEnrichmentStep can parse with narrowed 2-field schema. */
static final String LLM_JSON_RESPONSE =
    "{\"dispositionNarrative\":\"You operate independently.\","
    + "\"goalNarrative\":\"\"}";
```

Also update `capturingMock` in the test (the inline mock that returns a hardcoded JSON response) to use the 2-field format.

Find all occurrences of the old 6-field format strings in the test and replace with the 2-field equivalent. Check for:
- `LLM_JSON_RESPONSE` constant
- Any inline JSON strings in mocks that reference the 6-field format

After updating, run the full renderer test suite:

- [ ] **Step 4: Run runtime tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -f /Users/mdproctor/claude/casehub/eidos/pom.xml 2>&1 | tail -30
```

Expected: `BUILD SUCCESS`. All `renderer/` tests pass.

- [ ] **Step 5: Commit all Issue A changes**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add \
  runtime/src/main/java/io/casehub/eidos/runtime/renderer/JsonExtractionUtil.java \
  runtime/src/main/java/io/casehub/eidos/runtime/renderer/SemanticEnrichment.java \
  runtime/src/main/java/io/casehub/eidos/runtime/renderer/SemanticEnrichmentStep.java \
  runtime/src/main/java/io/casehub/eidos/runtime/renderer/ReactiveSemanticEnrichmentStep.java \
  runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosRenderPipeline.java \
  runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosSystemPromptRenderer.java \
  runtime/src/main/java/io/casehub/eidos/runtime/renderer/DefaultReactiveSystemPromptRenderer.java \
  runtime/src/test/java/io/casehub/eidos/runtime/renderer/SemanticEnrichmentStepTest.java \
  runtime/src/test/java/io/casehub/eidos/runtime/renderer/ReactiveSemanticEnrichmentStepTest.java \
  runtime/src/test/java/io/casehub/eidos/runtime/renderer/EidosSystemPromptRendererTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#XX): enrichment-mechanics — narrowed scope, JsonExtractionUtil, retry, selective override, buildEnrichmentPayload Closes #XX"
```

---

## Issue B — briefing-field

*(Start after Issue A is merged to main.)*

### Task B1: Add briefing to AgentDescriptor + Validator

**Files:**
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentDescriptor.java`
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentDescriptorValidator.java`
- Modify: `api/src/test/java/io/casehub/eidos/api/AgentDescriptorValidatorTest.java`

- [ ] **Step 1: Write failing tests for briefing validation**

In `AgentDescriptorValidatorTest.java`, add a new test method:

```java
@Test
void briefing_accepts_null() {
    assertThatNoException().isThrownBy(() ->
        AgentDescriptor.builder()
            .agentId("a").name("n").slot("s").tenancyId("t")
            .briefing(null)
            .build());
}

@Test
void briefing_accepts_short_text() {
    assertThatNoException().isThrownBy(() ->
        AgentDescriptor.builder()
            .agentId("a").name("n").slot("s").tenancyId("t")
            .briefing("Speed is a feature. 90% elegant beats perfect.")
            .build());
}

@Test
void briefing_rejects_over_500_chars() {
    assertThatThrownBy(() ->
        AgentDescriptor.builder()
            .agentId("a").name("n").slot("s").tenancyId("t")
            .briefing("x".repeat(501))
            .build())
        .isInstanceOf(AgentValidationException.class)
        .satisfies(ex -> assertThat(((AgentValidationException) ex).fieldName())
            .isEqualTo("briefing"));
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=AgentDescriptorValidatorTest -f /Users/mdproctor/claude/casehub/eidos/pom.xml 2>&1 | tail -10
```

Expected: compile error — `briefing(null)` not a valid Builder method.

- [ ] **Step 3: Add briefing to AgentDescriptorValidator**

In `AgentDescriptorValidator.java`, add:

```java
static final int MAX_BRIEFING = 500;
```

- [ ] **Step 4: Add briefing field to AgentDescriptor record and Builder**

In `AgentDescriptor.java`, add `String briefing` as the last field in the record:

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
        Map<DispositionAxis, String> axisVocabularies,
        String slot,
        List<AgentCapability> capabilities,
        AgentDisposition disposition,
        String jurisdiction,
        String dataHandlingPolicy,
        String tenancyId,
        String briefing   // nullable — behavioral principles the structured axes cannot express
) {
    public AgentDescriptor {
        // ... existing validation ...
        AgentDescriptorValidator.validateOptional("briefing", briefing, AgentDescriptorValidator.MAX_BRIEFING);
    }
```

In the `Builder` inner class, add:

```java
private String briefing;

public Builder briefing(String v) { this.briefing = v; return this; }
```

And in `Builder.build()`, add `briefing` as the last argument to the `AgentDescriptor` constructor call.

- [ ] **Step 5: Run tests to verify they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=AgentDescriptorValidatorTest -f /Users/mdproctor/claude/casehub/eidos/pom.xml 2>&1 | tail -10
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add \
  api/src/main/java/io/casehub/eidos/api/AgentDescriptor.java \
  api/src/main/java/io/casehub/eidos/api/AgentDescriptorValidator.java \
  api/src/test/java/io/casehub/eidos/api/AgentDescriptorValidatorTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#YY): AgentDescriptor.briefing — nullable free-text, MAX_BRIEFING=500 Refs #YY"
```

---

### Task B2: Add briefing to persistence layer

**Files:**
- Modify: `runtime/src/main/resources/db/eidos/migration/V1__initial_schema.sql`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentDescriptorEntity.java`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentDescriptorMapper.java`

No existing installations — all schema changes go directly into the base migration file.

- [ ] **Step 1: Add briefing column to V1__initial_schema.sql**

In `V1__initial_schema.sql`, in the `CREATE TABLE agent_descriptor` block, add the column after `data_handling_policy`:

```sql
briefing               TEXT          NULL,
```

The complete relevant section should look like:

```sql
data_handling_policy   TEXT,
briefing               TEXT          NULL,
CONSTRAINT uq_agent UNIQUE (agent_id, tenancy_id)
```

- [ ] **Step 2: Add briefing to AgentDescriptorEntity**

In `AgentDescriptorEntity.java`, add the field after `dataHandlingPolicy`:

```java
@Column(columnDefinition = "TEXT")
String briefing;
```

- [ ] **Step 3: Update AgentDescriptorMapper**

In `toRecord()`, add `e.briefing` as the last argument to the `AgentDescriptor` constructor (it must match the field order in the record):

```java
AgentDescriptor toRecord(AgentDescriptorEntity e) {
    return new AgentDescriptor(
        e.agentId, e.name, e.version, e.provider,
        e.modelFamily, e.modelVersion, e.weightsFingerprint,
        e.domainVocabulary, e.slotVocabulary, e.dispositionVocabulary,
        readJson(e.axisVocabularies, new TypeReference<Map<DispositionAxis, String>>() {}),
        e.slot,
        e.capabilities.stream().map(this::toCapability).toList(),
        readJson(e.disposition, AgentDisposition.class),
        e.jurisdiction, e.dataHandlingPolicy, e.tenancyId,
        e.briefing   // new
    );
}
```

In `toEntity()`, add:

```java
e.briefing = d.briefing();
```

(after `e.dataHandlingPolicy = d.dataHandlingPolicy();`)

- [ ] **Step 4: Run runtime tests to verify the persistence layer compiles**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -f /Users/mdproctor/claude/casehub/eidos/pom.xml 2>&1 | tail -20
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add \
  runtime/src/main/resources/db/eidos/migration/V1__initial_schema.sql \
  runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentDescriptorEntity.java \
  runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentDescriptorMapper.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#YY): briefing persistence — schema column, entity field, mapper Refs #YY"
```

---

### Task B3: Add briefing to renderer pipeline

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosRenderPipeline.java`
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/renderer/EidosSystemPromptRendererTest.java`

- [ ] **Step 1: Add briefing to buildDescriptorPayload (cache key correctness)**

In `buildDescriptorPayload()`, add after `dataHandlingPolicy`:

```java
addIfPresent(node, "briefing", descriptor.briefing());
```

- [ ] **Step 2: Structural fallback — briefing in MARKDOWN format**

In `assembleMarkdownDisposition()`, add briefing rendering after the structural disposition block. But wait — briefing renders only when enrichment fails AND briefing is non-null. Briefing is on the `AgentDescriptor`. Pass the descriptor to a new helper or inline it in `assembleMarkdown`:

In `assembleMarkdown`, after the disposition selective override block:

```java
// Briefing structural fallback — only when disposition is structural (enrichment absent or failed)
// AND briefing is present. When enrichment succeeded, briefing is already woven into dispositionNarrative.
if (!(enrichment.isPresent() && enrichment.get().dispositionNarrative().isPresent())
        && descriptor.briefing() != null) {
    sb.append("\n## Operating Principles\n").append(descriptor.briefing()).append("\n");
}
```

- [ ] **Step 3: Structural fallback — briefing in PROSE format**

In `assembleProse`, after the disposition selective override block:

```java
// Briefing structural fallback in PROSE — paragraph after disposition, no header
if (!(enrichment.isPresent() && enrichment.get().dispositionNarrative().isPresent())
        && descriptor.briefing() != null) {
    sb.append("\n").append(descriptor.briefing()).append("\n");
}
```

- [ ] **Step 4: Write renderer tests for briefing**

In `EidosSystemPromptRendererTest.java`, add:

```java
@Test
void structural_markdown_includes_operating_principles_when_briefing_set() {
    final var desc = AgentDescriptor.builder()
        .agentId("a").name("Bold Engineer").slot("reviewer")
        .disposition(AgentDisposition.builder().riskAppetite("bold").build())
        .briefing("Speed is a feature. 90% elegant beats perfect.")
        .tenancyId("t")
        .build();
    final var ctx = AgentPromptContext.forFormat(MARKDOWN);

    final String content = rendererStructural.render(desc, ctx).content();

    assertThat(content).contains("## Operating Principles");
    assertThat(content).contains("Speed is a feature.");
}

@Test
void structural_prose_includes_briefing_paragraph_when_set() {
    final var desc = AgentDescriptor.builder()
        .agentId("a").name("Bold Engineer").slot("reviewer")
        .disposition(AgentDisposition.builder().riskAppetite("bold").build())
        .briefing("Speed is a feature.")
        .tenancyId("t")
        .build();
    final var ctx = AgentPromptContext.forFormat(PROSE);

    final String content = rendererStructural.render(desc, ctx).content();

    assertThat(content).contains("Speed is a feature.");
    assertThat(content).doesNotContain("## Operating Principles"); // PROSE has no headers
}

@Test
void no_operating_principles_when_briefing_null() {
    final var desc = AgentDescriptor.builder()
        .agentId("a").name("n").slot("s").tenancyId("t").build();
    final var ctx = AgentPromptContext.forFormat(MARKDOWN);

    final String content = rendererStructural.render(desc, ctx).content();

    assertThat(content).doesNotContain("Operating Principles");
}
```

- [ ] **Step 5: Run renderer tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -f /Users/mdproctor/claude/casehub/eidos/pom.xml 2>&1 | tail -20
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add \
  runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosRenderPipeline.java \
  runtime/src/test/java/io/casehub/eidos/runtime/renderer/EidosSystemPromptRendererTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#YY): briefing in renderer — cache key, enrichment payload, structural fallback MARKDOWN+PROSE Closes #YY"
```

---

## Issue C — proximity-eval-redesign

*(Start after Issue A is merged to main.)*

### Task C1: Update ProximityJudge

**Files:**
- Modify: `eval/src/main/java/io/casehub/eidos/eval/ProximityJudge.java`
- Modify: `eval/src/test/java/io/casehub/eidos/eval/ProximityJudgeTest.java`

- [ ] **Step 1: Write failing tests for new ProximityJudge behaviour**

Replace `ProximityJudgeTest.java`:

```java
// eval/src/test/java/io/casehub/eidos/eval/ProximityJudgeTest.java
package io.casehub.eidos.eval;

import com.fasterxml.jackson.databind.ObjectMapper;
import dev.langchain4j.data.message.AiMessage;
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.model.chat.request.ChatRequest;
import dev.langchain4j.model.chat.response.ChatResponse;
import io.casehub.eidos.api.*;
import io.casehub.eidos.api.SystemPromptRenderer.RenderFormat;
import io.casehub.eidos.api.SystemPromptRenderer.RenderedPrompt;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;
import java.util.concurrent.atomic.AtomicReference;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class ProximityJudgeTest {

    static final String VALID_JSON = """
        { "score": 4, "reasoning": "Axes clearly conveyed.", "gaps": ["no autonomy axis"] }
        """;

    ProximityJudge judge;
    ProfiledEvalCase evalCase;
    RenderedPrompt rendered;

    @BeforeEach
    void setUp() {
        judge = new ProximityJudge(new ChatModel() {
            @Override
            public ChatResponse doChat(final ChatRequest request) {
                return ChatResponse.builder().aiMessage(AiMessage.from(VALID_JSON)).build();
            }
        }, new ObjectMapper());
        evalCase = caseWithDisposition(
            AgentDisposition.builder().riskAppetite("bold").ruleFollowing("strict").build());
        rendered = new RenderedPrompt("You approve boldly.", RenderFormat.MARKDOWN, "dh", "ch");
    }

    @Test
    void evaluate_parses_score() {
        assertThat(judge.evaluate(evalCase, rendered).score()).isEqualTo(4);
    }

    @Test
    void evaluate_parses_reasoning() {
        assertThat(judge.evaluate(evalCase, rendered).reasoning()).isEqualTo("Axes clearly conveyed.");
    }

    @Test
    void evaluate_parses_gaps() {
        assertThat(judge.evaluate(evalCase, rendered).gaps())
            .containsExactly("no autonomy axis");
    }

    @Test
    void evaluate_payload_contains_disposition_axes_not_originalProse() {
        final AtomicReference<String> capturedPayload = new AtomicReference<>();
        final ProximityJudge capturingJudge = new ProximityJudge(new ChatModel() {
            @Override
            public ChatResponse doChat(final ChatRequest request) {
                request.messages().forEach(m -> {
                    if (m instanceof dev.langchain4j.data.message.UserMessage um)
                        capturedPayload.set(um.singleText());
                });
                return ChatResponse.builder().aiMessage(AiMessage.from(VALID_JSON)).build();
            }
        }, new ObjectMapper());
        capturingJudge.evaluate(evalCase, rendered);

        // payload must contain disposition axes
        assertThat(capturedPayload.get()).contains("riskAppetite");
        assertThat(capturedPayload.get()).contains("bold");
        // payload must NOT contain originalProse
        assertThat(capturedPayload.get()).doesNotContain("You are careful.");
    }

    @Test
    void null_axes_excluded_from_payload() {
        // disposition with only riskAppetite set; conflictMode is null
        final AtomicReference<String> capturedPayload = new AtomicReference<>();
        final ProximityJudge capturingJudge = new ProximityJudge(new ChatModel() {
            @Override
            public ChatResponse doChat(final ChatRequest request) {
                request.messages().forEach(m -> {
                    if (m instanceof dev.langchain4j.data.message.UserMessage um)
                        capturedPayload.set(um.singleText());
                });
                return ChatResponse.builder().aiMessage(AiMessage.from(VALID_JSON)).build();
            }
        }, new ObjectMapper());
        capturingJudge.evaluate(evalCase, rendered);

        // "conflictMode" must not appear (null axis)
        assertThat(capturedPayload.get()).doesNotContain("conflictMode");
    }

    @Test
    void null_disposition_scores_5_with_no_axes_declared() {
        final var descNoDisp = AgentDescriptor.builder()
            .agentId("id").name("N").slot("s").tenancyId("t").build();
        final var profileNoDisp = new AgentProfile(
            "p", "r", "d", null, null, SourceType.ANTHROPIC_LIBRARY,
            "prose", null, null, Map.of(), Map.of(), descNoDisp, List.of());
        final var caseNoDisp = new ProfiledEvalCase("nodisp", descNoDisp,
            AgentPromptContext.forFormat(RenderFormat.MARKDOWN), profileNoDisp);

        // Mock returns score 5 for "no axes" case as instructed by SYSTEM_PROMPT
        final ProximityJudge scoresFiveJudge = new ProximityJudge(new ChatModel() {
            @Override
            public ChatResponse doChat(final ChatRequest request) {
                return ChatResponse.builder().aiMessage(AiMessage.from(
                    "{\"score\":5,\"reasoning\":\"No disposition axes declared\",\"gaps\":[]}")).build();
            }
        }, new ObjectMapper());

        final var result = scoresFiveJudge.evaluate(caseNoDisp, rendered);
        assertThat(result.score()).isEqualTo(5);
        assertThat(result.reasoning()).isEqualTo("No disposition axes declared");
    }

    @Test
    void evaluate_throws_malformed_when_score_missing() {
        final var noScore = new ProximityJudge(new ChatModel() {
            @Override public ChatResponse doChat(final ChatRequest r) {
                return ChatResponse.builder().aiMessage(
                    AiMessage.from("{\"reasoning\":\"ok\",\"gaps\":[]}")).build();
            }
        }, new ObjectMapper());
        assertThatThrownBy(() -> noScore.evaluate(evalCase, rendered))
            .isInstanceOf(MalformedJudgeResponseException.class);
    }

    private static ProfiledEvalCase caseWithDisposition(final AgentDisposition disp) {
        final var desc = AgentDescriptor.builder()
            .agentId("id").name("N").slot("reviewer")
            .disposition(disp)
            .capabilities(List.of())
            .tenancyId("t")
            .build();
        final var profile = new AgentProfile(
            "sw-engineer-careful", "SW Eng", "sw", null, null,
            SourceType.ANTHROPIC_LIBRARY, "You are careful.", null, null,
            Map.of(), Map.of(), desc, List.of());
        return new ProfiledEvalCase("test", desc,
            AgentPromptContext.forFormat(RenderFormat.MARKDOWN), profile);
    }
}
```

- [ ] **Step 2: Run test to verify it fails (expected)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -f /Users/mdproctor/claude/casehub/eidos/pom.xml -Dtest=ProximityJudgeTest 2>&1 | tail -20
```

Expected: tests fail because `evaluate()` still uses `originalProse`.

- [ ] **Step 3: Rewrite ProximityJudge.SYSTEM_PROMPT and evaluate() body**

In `ProximityJudge.java`, replace `SYSTEM_PROMPT`:

```java
static final String SYSTEM_PROMPT = """
    You are evaluating whether a rendered AI agent system prompt correctly and completely \
    expresses the agent's disposition axes as declared in the descriptor.

    For each axis present in the disposition object, check whether the render conveys \
    that axis value.

    Score 0–5:
    - 5: All axes clearly and correctly expressed.
    - 4: Minor omission or softening of one axis.
    - 3: Partial coverage — some axes present, others missing.
    - 2: Significant axes missing from the render.
    - 1: Most axes missing or contradicted by the render.
    - 0: Disposition entirely absent from the render.

    If no disposition object is provided in the payload, the descriptor has no disposition \
    axes — return score: 5, reasoning: "No disposition axes declared", gaps: [].

    Return JSON: { "score": int, "reasoning": string, "gaps": string[] }
    """;
```

Replace `evaluate()` body:

```java
public ProximityResult evaluate(final ProfiledEvalCase evalCase, final RenderedPrompt rendered) {
    final ObjectNode payload = mapper.createObjectNode();

    // Build disposition node using only non-null axes — avoids null JSON values
    // that would confuse the judge about which axes to evaluate.
    final AgentDisposition disp = evalCase.descriptor().disposition();
    if (disp != null) {
        final ObjectNode dispNode = mapper.createObjectNode();
        for (final DispositionAxis axis : DispositionAxis.values()) {
            disp.get(axis).ifPresent(raw -> dispNode.put(axis.jsonKey(), raw));
        }
        dispNode.put("canDelegate", disp.delegation());
        payload.set("disposition", dispNode);
    }
    // If disp is null, no "disposition" key is added — judge skips scoring (no axes to evaluate)

    payload.put("rendered", rendered.content());

    try {
        final var request = ChatRequest.builder()
            .messages(
                SystemMessage.from(SYSTEM_PROMPT),
                UserMessage.from(mapper.writeValueAsString(payload)))
            .responseFormat(RESPONSE_FORMAT)
            .build();
        ProximityResult result;
        try {
            result = parse(evalCase, model.chat(request).aiMessage().text());
        } catch (final MalformedJudgeResponseException first) {
            System.err.printf("[WARN] ProximityJudge non-JSON response, retrying (%s)%n", first.getMessage());
            result = parse(evalCase, model.chat(request).aiMessage().text());
        }
        return result;
    } catch (final MalformedJudgeResponseException e) {
        throw e;
    } catch (final Exception e) {
        throw new IllegalStateException("ProximityJudge LLM call failed", e);
    }
}
```

Add import: `import io.casehub.eidos.api.DispositionAxis;` (already imported via `io.casehub.eidos.api.*` wildcard if present, otherwise add explicitly).

- [ ] **Step 4: Run tests to verify they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -f /Users/mdproctor/claude/casehub/eidos/pom.xml -Dtest=ProximityJudgeTest 2>&1 | tail -20
```

Expected: `BUILD SUCCESS`, all `ProximityJudgeTest` tests pass.

- [ ] **Step 5: Run full eval test suite (excluding @Tag("eval") which requires LLM)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -f /Users/mdproctor/claude/casehub/eidos/pom.xml 2>&1 | tail -20
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add \
  eval/src/main/java/io/casehub/eidos/eval/ProximityJudge.java \
  eval/src/test/java/io/casehub/eidos/eval/ProximityJudgeTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#ZZ): ProximityJudge — descriptor-axis completeness, DispositionAxis.jsonKey null-safe, null-disposition guard Closes #ZZ"
```

---

## Final Validation

After all issues are merged:

- [ ] **Full test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api,runtime,eval -f /Users/mdproctor/claude/casehub/eidos/pom.xml 2>&1 | tail -20
```

Expected: `BUILD SUCCESS`.

- [ ] **Eval harness — enrichment success rate**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean test -pl eval -Peval \
  -Dtest=PromptEvalTest#evaluateRealWorldScenarios \
  -Dcasehub.eval.renders-cache.path=/tmp/eidos-renders-cache.json \
  -f /Users/mdproctor/claude/casehub/eidos/pom.xml 2>&1 | grep -E "WARN.*Semantic|How You Operate|tests run"
```

Expected: WARN messages for enrichment fallbacks should be rare (<20% of cases). Rendered disposition sections should be prose paragraphs, not bullet lists.

- [ ] **Proximity eval — redesigned judge**

After populating `briefing` in at least one eval profile YAML (e.g. `sw-engineer-bold.yaml`):

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean test -pl eval -Peval \
  -Dtest=PromptEvalTest#evaluateRealWorldScenarios \
  -Dcasehub.eval.renders-cache.path=/tmp/eidos-renders-cache.json \
  -f /Users/mdproctor/claude/casehub/eidos/pom.xml 2>&1 | grep "proximity"
```

Run the independent judge comparison last (after Changes A+B+C all in place):

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean test -pl eval -Peval,eval-ollama \
  -Dtest=PromptEvalTest#evaluateWithIndependentJudge \
  -Dcasehub.eval.claude-provider.enabled=false \
  -Dcasehub.eval.renders-cache.path=/tmp/eidos-renders-cache.json \
  "-Dquarkus.langchain4j.ollama.chat-model.model-name=qwen3:8b" \
  "-Dquarkus.langchain4j.ollama.chat-model.format=json" \
  "-Dquarkus.langchain4j.ollama.timeout=300s" \
  -Dcasehub.eval.model.label=qwen3-8b \
  -f /Users/mdproctor/claude/casehub/eidos/pom.xml 2>&1 | tail -30
```
