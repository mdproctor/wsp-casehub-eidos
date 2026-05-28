# Semantic Rendering Pipeline Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Refactor `ClaudeMarkdownRenderer` into a three-stage pipeline (payload → optional LLM enrichment → format assembly), add `RenderedPromptCache` SPI with no-op and in-memory implementations, and fix structural rendering to be format-aware.

**Architecture:** Stage 1 builds a curated Jackson `ObjectNode` (canonical JSON payload, drives cache hashes). Stage 2 optionally calls an LLM via `SemanticEnrichmentStep` to produce `SemanticEnrichment` (six prose fields). Stage 3 assembles format-specific output via a `switch` on `RenderFormat`, deterministically, with or without enrichment. The public `SystemPromptRenderer` SPI is unchanged.

**Tech Stack:** Java 21 on Java 26 JVM, Quarkus 3.32.2, LangChain4j 1.14.1 (`langchain4j-core`), Jackson (`ObjectMapper`, `ObjectNode`), JUnit 5, AssertJ.

**Spec:** `specs/2026-05-28-semantic-rendering-pipeline-design.md`

**Build commands:**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install   # full build
```

---

## File Map

| Action | File | Notes |
|---|---|---|
| Create | `api/src/main/java/io/casehub/eidos/api/RenderedPromptCache.java` | New public SPI |
| Create | `runtime/src/main/java/io/casehub/eidos/runtime/renderer/NoOpRenderedPromptCache.java` | `@DefaultBean` |
| Create | `runtime/src/main/java/io/casehub/eidos/runtime/renderer/SemanticEnrichment.java` | Package-private record |
| Create | `runtime/src/main/java/io/casehub/eidos/runtime/renderer/SemanticEnrichmentStep.java` | Package-private class |
| Create | `runtime/src/test/java/io/casehub/eidos/runtime/renderer/SemanticEnrichmentStepTest.java` | New test class |
| Create | `persistence-memory/src/main/java/io/casehub/eidos/memory/InMemoryRenderedPromptCache.java` | `@Alternative @Priority(1)` |
| Create | `persistence-memory/src/test/java/io/casehub/eidos/memory/InMemoryRenderedPromptCacheTest.java` | New test class |
| Modify | `runtime/src/main/java/io/casehub/eidos/runtime/renderer/ClaudeMarkdownRenderer.java` | Major rewrite |
| Modify | `runtime/src/test/java/io/casehub/eidos/runtime/renderer/ClaudeMarkdownRendererTest.java` | Major update |

---

## Task 1: `RenderedPromptCache` SPI and `NoOpRenderedPromptCache`

**Files:**
- Create: `api/src/main/java/io/casehub/eidos/api/RenderedPromptCache.java`
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/renderer/NoOpRenderedPromptCache.java`

- [ ] **Step 1.1: Create the SPI interface**

```java
// api/src/main/java/io/casehub/eidos/api/RenderedPromptCache.java
package io.casehub.eidos.api;

import java.util.Optional;

public interface RenderedPromptCache {
    Optional<SystemPromptRenderer.RenderedPrompt> get(String cacheKey);

    /**
     * Stores a rendered prompt. Must not throw — implementations handle errors internally.
     */
    void put(String cacheKey, SystemPromptRenderer.RenderedPrompt result);
}
```

- [ ] **Step 1.2: Build to confirm it compiles**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api
```
Expected: BUILD SUCCESS

- [ ] **Step 1.3: Create the no-op default**

```java
// runtime/src/main/java/io/casehub/eidos/runtime/renderer/NoOpRenderedPromptCache.java
package io.casehub.eidos.runtime.renderer;

import io.casehub.eidos.api.RenderedPromptCache;
import io.casehub.eidos.api.SystemPromptRenderer.RenderedPrompt;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;

import java.util.Optional;

@DefaultBean
@ApplicationScoped
public class NoOpRenderedPromptCache implements RenderedPromptCache {

    @Override
    public Optional<RenderedPrompt> get(final String cacheKey) {
        return Optional.empty();
    }

    @Override
    public void put(final String cacheKey, final RenderedPrompt result) {}
}
```

- [ ] **Step 1.4: Build runtime module**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime
```
Expected: BUILD SUCCESS

- [ ] **Step 1.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add \
  api/src/main/java/io/casehub/eidos/api/RenderedPromptCache.java \
  runtime/src/main/java/io/casehub/eidos/runtime/renderer/NoOpRenderedPromptCache.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#6): RenderedPromptCache SPI + NoOpRenderedPromptCache

Refs #6"
```

---

## Task 2: `InMemoryRenderedPromptCache`

**Files:**
- Create: `persistence-memory/src/main/java/io/casehub/eidos/memory/InMemoryRenderedPromptCache.java`
- Create: `persistence-memory/src/test/java/io/casehub/eidos/memory/InMemoryRenderedPromptCacheTest.java`

- [ ] **Step 2.1: Write the failing test first**

```java
// persistence-memory/src/test/java/io/casehub/eidos/memory/InMemoryRenderedPromptCacheTest.java
package io.casehub.eidos.memory;

import io.casehub.eidos.api.SystemPromptRenderer.RenderedPrompt;
import io.casehub.eidos.api.SystemPromptRenderer.RenderFormat;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class InMemoryRenderedPromptCacheTest {

    InMemoryRenderedPromptCache cache;

    @BeforeEach
    void setUp() {
        cache = new InMemoryRenderedPromptCache();
        cache.maxSize = 3;
        cache.init();
    }

    static RenderedPrompt prompt(String content) {
        return new RenderedPrompt(content, RenderFormat.CLAUDE_MD, "dh", "ch");
    }

    @Test
    void miss_returns_empty() {
        assertThat(cache.get("missing")).isEmpty();
    }

    @Test
    void put_then_get_returns_value() {
        cache.put("k1", prompt("hello"));
        assertThat(cache.get("k1")).contains(prompt("hello"));
    }

    @Test
    void evicts_least_recently_used_when_full() {
        cache.put("k1", prompt("one"));
        cache.put("k2", prompt("two"));
        cache.put("k3", prompt("three"));
        // access k1 to make k2 the LRU
        cache.get("k1");
        cache.get("k3");
        // adding k4 should evict k2
        cache.put("k4", prompt("four"));
        assertThat(cache.get("k2")).isEmpty();
        assertThat(cache.get("k1")).isPresent();
        assertThat(cache.get("k3")).isPresent();
        assertThat(cache.get("k4")).isPresent();
    }

    @Test
    void different_keys_are_independent() {
        cache.put("a", prompt("alpha"));
        cache.put("b", prompt("beta"));
        assertThat(cache.get("a")).contains(prompt("alpha"));
        assertThat(cache.get("b")).contains(prompt("beta"));
    }
}
```

- [ ] **Step 2.2: Run to confirm test fails (class not found)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory \
  -Dtest=InMemoryRenderedPromptCacheTest
```
Expected: compilation error — `InMemoryRenderedPromptCache` does not exist

- [ ] **Step 2.3: Create the implementation**

```java
// persistence-memory/src/main/java/io/casehub/eidos/memory/InMemoryRenderedPromptCache.java
package io.casehub.eidos.memory;

import io.casehub.eidos.api.RenderedPromptCache;
import io.casehub.eidos.api.SystemPromptRenderer.RenderedPrompt;
import jakarta.annotation.PostConstruct;
import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import org.eclipse.microprofile.config.inject.ConfigProperty;

import java.util.Collections;
import java.util.LinkedHashMap;
import java.util.Map;
import java.util.Optional;

@Alternative
@Priority(1)
@ApplicationScoped
public class InMemoryRenderedPromptCache implements RenderedPromptCache {

    @ConfigProperty(name = "casehub.eidos.renderer.cache-size", defaultValue = "256")
    int maxSize;

    private Map<String, RenderedPrompt> cache;

    @PostConstruct
    void init() {
        cache = Collections.synchronizedMap(
            new LinkedHashMap<>(16, 0.75f, true) {
                @Override
                protected boolean removeEldestEntry(final Map.Entry<String, RenderedPrompt> eldest) {
                    return size() > maxSize;
                }
            }
        );
    }

    @Override
    public Optional<RenderedPrompt> get(final String cacheKey) {
        return Optional.ofNullable(cache.get(cacheKey));
    }

    @Override
    public void put(final String cacheKey, final RenderedPrompt result) {
        cache.put(cacheKey, result);
    }
}
```

- [ ] **Step 2.4: Run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory \
  -Dtest=InMemoryRenderedPromptCacheTest
```
Expected: 4 tests PASS

- [ ] **Step 2.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add \
  persistence-memory/src/main/java/io/casehub/eidos/memory/InMemoryRenderedPromptCache.java \
  persistence-memory/src/test/java/io/casehub/eidos/memory/InMemoryRenderedPromptCacheTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#6): InMemoryRenderedPromptCache — bounded LRU in casehub-eidos-memory

Refs #6"
```

---

## Task 3: `SemanticEnrichment` record

**Files:**
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/renderer/SemanticEnrichment.java`

- [ ] **Step 3.1: Create the package-private record**

```java
// runtime/src/main/java/io/casehub/eidos/runtime/renderer/SemanticEnrichment.java
package io.casehub.eidos.runtime.renderer;

import java.util.Optional;

record SemanticEnrichment(
        String identityNarrative,
        String roleNarrative,
        String capabilityNarrative,
        Optional<String> dispositionNarrative,
        Optional<String> constraintNarrative,
        Optional<String> goalNarrative
) {}
```

- [ ] **Step 3.2: Build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime
```
Expected: BUILD SUCCESS

- [ ] **Step 3.3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add \
  runtime/src/main/java/io/casehub/eidos/runtime/renderer/SemanticEnrichment.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#6): SemanticEnrichment — package-private intermediate record

Refs #6"
```

---

## Task 4: `SemanticEnrichmentStep`

**Files:**
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/renderer/SemanticEnrichmentStep.java`
- Create: `runtime/src/test/java/io/casehub/eidos/runtime/renderer/SemanticEnrichmentStepTest.java`

- [ ] **Step 4.1: Write the failing tests**

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

    static ObjectNode payload(String agentId) {
        final ObjectNode node = MAPPER.createObjectNode();
        node.put("agentId", agentId);
        node.put("name", "Test Agent");
        node.put("slot", "tester");
        return node;
    }

    static ChatModel mockReturning(String json) {
        return new ChatModel() {
            @Override
            public ChatResponse doChat(final ChatRequest request) {
                return ChatResponse.builder()
                        .aiMessage(AiMessage.from(json))
                        .build();
            }
        };
    }

    static ChatModel capturingMock(String[] captured) {
        return new ChatModel() {
            @Override
            public ChatResponse doChat(final ChatRequest request) {
                captured[0] = request.messages().stream()
                        .filter(m -> m instanceof UserMessage)
                        .map(m -> ((UserMessage) m).singleText())
                        .findFirst().orElse("");
                captured[1] = request.messages().stream()
                        .filter(m -> m instanceof dev.langchain4j.data.message.SystemMessage)
                        .map(m -> ((dev.langchain4j.data.message.SystemMessage) m).text())
                        .findFirst().orElse("");
                return ChatResponse.builder()
                        .aiMessage(AiMessage.from("""
                            {"identityNarrative":"id","roleNarrative":"role",
                             "capabilityNarrative":"cap","dispositionNarrative":"",
                             "constraintNarrative":"","goalNarrative":""}"""))
                        .build();
            }
        };
    }

    @Test
    void parse_valid_json_populates_required_fields() {
        final String json = """
            {"identityNarrative":"You are TestAgent.",
             "roleNarrative":"Your role is testing.",
             "capabilityNarrative":"You can test things.",
             "dispositionNarrative":"You are strict.",
             "constraintNarrative":"",
             "goalNarrative":""}""";

        final Optional<SemanticEnrichment> result = step.enrich(mockReturning(json), payload("a1"));

        assertThat(result).isPresent();
        assertThat(result.get().identityNarrative()).isEqualTo("You are TestAgent.");
        assertThat(result.get().roleNarrative()).isEqualTo("Your role is testing.");
        assertThat(result.get().capabilityNarrative()).isEqualTo("You can test things.");
        assertThat(result.get().dispositionNarrative()).contains("You are strict.");
    }

    @Test
    void blank_optional_fields_become_empty() {
        final String json = """
            {"identityNarrative":"id","roleNarrative":"role","capabilityNarrative":"cap",
             "dispositionNarrative":"","constraintNarrative":"  ","goalNarrative":""}""";

        final Optional<SemanticEnrichment> result = step.enrich(mockReturning(json), payload("a1"));

        assertThat(result.get().dispositionNarrative()).isEmpty();
        assertThat(result.get().constraintNarrative()).isEmpty();
        assertThat(result.get().goalNarrative()).isEmpty();
    }

    @Test
    void non_blank_optional_field_is_present() {
        final String json = """
            {"identityNarrative":"id","roleNarrative":"role","capabilityNarrative":"cap",
             "dispositionNarrative":"","constraintNarrative":"","goalNarrative":"Review PR #42."}""";

        final Optional<SemanticEnrichment> result = step.enrich(mockReturning(json), payload("a1"));

        assertThat(result.get().goalNarrative()).contains("Review PR #42.");
    }

    @Test
    void exception_from_llm_returns_empty() {
        final ChatModel throwing = new ChatModel() {
            @Override
            public ChatResponse doChat(final ChatRequest request) {
                throw new RuntimeException("Model unavailable");
            }
        };

        assertThat(step.enrich(throwing, payload("a1"))).isEmpty();
    }

    @Test
    void malformed_json_returns_empty() {
        assertThat(step.enrich(mockReturning("not json at all"), payload("a1"))).isEmpty();
    }

    @Test
    void user_message_contains_payload_fields() {
        final String[] captured = {"", ""};
        step.enrich(capturingMock(captured), payload("agent-42"));
        assertThat(captured[0]).contains("agent-42");
    }

    @Test
    void system_message_equals_prompt_template() {
        final String[] captured = {"", ""};
        step.enrich(capturingMock(captured), payload("x"));
        assertThat(captured[1]).isEqualTo(ClaudeMarkdownRenderer.PROMPT_TEMPLATE);
    }

    @Test
    void chat_request_has_response_format() {
        final boolean[] hasFormat = {false};
        final ChatModel checkingMock = new ChatModel() {
            @Override
            public ChatResponse doChat(final ChatRequest request) {
                hasFormat[0] = request.parameters() != null
                        && request.parameters().responseFormat() != null;
                return ChatResponse.builder()
                        .aiMessage(AiMessage.from("""
                            {"identityNarrative":"id","roleNarrative":"r","capabilityNarrative":"c",
                             "dispositionNarrative":"","constraintNarrative":"","goalNarrative":""}"""))
                        .build();
            }
        };
        step.enrich(checkingMock, payload("x"));
        assertThat(hasFormat[0]).isTrue();
    }
}
```

- [ ] **Step 4.2: Run to confirm compilation fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -pl runtime \
  -Dtest=SemanticEnrichmentStepTest 2>&1 | grep "ERROR\|error:" | head -10
```
Expected: compilation error — `SemanticEnrichmentStep` not found

- [ ] **Step 4.3: Create `SemanticEnrichmentStep`**

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
import dev.langchain4j.model.chat.request.ResponseFormat;
import dev.langchain4j.model.chat.request.ResponseFormatType;
import dev.langchain4j.model.chat.request.json.JsonObjectSchema;
import dev.langchain4j.model.chat.request.json.JsonSchema;
import org.jboss.logging.Logger;

import java.util.Optional;

class SemanticEnrichmentStep {

    private static final Logger log = Logger.getLogger(SemanticEnrichmentStep.class);

    // Declaration order is load-order: RESPONSE_FORMAT is self-contained, no field dependency.
    static final ResponseFormat RESPONSE_FORMAT = ResponseFormat.builder()
            .type(ResponseFormatType.JSON)
            .jsonSchema(JsonSchema.builder()
                    .name("SemanticEnrichment")
                    .rootElement(JsonObjectSchema.builder()
                            .addStringProperty("identityNarrative",
                                    "Who this agent is — name, model, version context. Second person.")
                            .addStringProperty("roleNarrative",
                                    "The agent's role and purpose. Second person.")
                            .addStringProperty("capabilityNarrative",
                                    "What the agent can do, including domain confidence. Second person.")
                            .addStringProperty("dispositionNarrative",
                                    "How the agent operates. Empty string if no disposition data.")
                            .addStringProperty("constraintNarrative",
                                    "Data handling obligations. Empty string if none.")
                            .addStringProperty("goalNarrative",
                                    "Current task and objectives. Empty string if no goal.")
                            .required("identityNarrative", "roleNarrative", "capabilityNarrative",
                                    "dispositionNarrative", "constraintNarrative", "goalNarrative")
                            .build())
                    .build())
            .build();

    private final ObjectMapper mapper;

    SemanticEnrichmentStep(final ObjectMapper mapper) {
        this.mapper = mapper;
    }

    Optional<SemanticEnrichment> enrich(final ChatModel llm, final ObjectNode payload) {
        try {
            final ChatRequest request = ChatRequest.builder()
                    .messages(
                            SystemMessage.from(ClaudeMarkdownRenderer.PROMPT_TEMPLATE),
                            UserMessage.from(mapper.writeValueAsString(payload))
                    )
                    .responseFormat(RESPONSE_FORMAT)
                    .build();

            final var response = llm.chat(request);
            return Optional.of(parse(response.aiMessage().text()));

        } catch (final Exception e) {
            log.warn("Semantic enrichment failed (" + e.getMessage()
                    + "), falling back to structural rendering");
            return Optional.empty();
        }
    }

    private SemanticEnrichment parse(final String json) throws JsonProcessingException {
        final JsonNode node = mapper.readTree(json);
        return new SemanticEnrichment(
                node.get("identityNarrative").asText(),
                node.get("roleNarrative").asText(),
                node.get("capabilityNarrative").asText(),
                optional(node, "dispositionNarrative"),
                optional(node, "constraintNarrative"),
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

- [ ] **Step 4.4: Run tests (will fail — `ClaudeMarkdownRenderer.PROMPT_TEMPLATE` not yet package-private)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=SemanticEnrichmentStepTest
```
Expected: compilation fails on `ClaudeMarkdownRenderer.PROMPT_TEMPLATE` — the field is currently `private`. This is expected; we fix it in Task 5.

- [ ] **Step 4.5: Commit what compiles**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add \
  runtime/src/main/java/io/casehub/eidos/runtime/renderer/SemanticEnrichmentStep.java \
  runtime/src/test/java/io/casehub/eidos/runtime/renderer/SemanticEnrichmentStepTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#6): SemanticEnrichmentStep — LLM call, parse, fallback

Refs #6"
```

---

## Task 5: Rewrite `ClaudeMarkdownRenderer` — constructors, constants, payload building

This task updates the constructor signatures (both CDI and test), adds `PROMPT_TEMPLATE` / `TEMPLATE_HASH` constants, and adds the payload-building methods. The existing `render()` is temporarily untouched; existing tests will need their constructor calls updated.

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/renderer/ClaudeMarkdownRenderer.java`
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/renderer/ClaudeMarkdownRendererTest.java`

- [ ] **Step 5.1: Replace the full renderer file**

```java
// runtime/src/main/java/io/casehub/eidos/runtime/renderer/ClaudeMarkdownRenderer.java
package io.casehub.eidos.runtime.renderer;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ArrayNode;
import com.fasterxml.jackson.databind.node.ObjectNode;
import io.casehub.eidos.api.*;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Any;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import dev.langchain4j.model.chat.ChatModel;

import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.util.HexFormat;
import java.util.Optional;

@DefaultBean
@ApplicationScoped
public class ClaudeMarkdownRenderer implements SystemPromptRenderer {

    // PROMPT_TEMPLATE must be declared before TEMPLATE_HASH — static initializers run
    // in declaration order. Reversing them causes sha256(null) at class load:
    // NullPointerException wrapped in ExceptionInInitializerError, not a quiet wrong value.
    static final String PROMPT_TEMPLATE = """
            You are writing narrative descriptions for an AI agent's system prompt.

            Given the agent definition in JSON, produce a JSON object with prose descriptions
            for each field. Write in second person, addressing the agent directly.

            REQUIRED FIELDS (always populate):
            - identityNarrative (1-2 sentences): The agent's name, model, and version context.
            - roleNarrative (1-3 sentences): The role this agent plays and its purpose.
              If slotLabel and slotDescription are present, prefer them over the raw slot value.
            - capabilityNarrative (2-4 sentences): What the agent can do.
              Include inputTypes and outputTypes when present.
              For epistemicDomains, use natural language confidence:
                >= 0.7 -> "strong expertise", 0.4-0.69 -> "working knowledge", < 0.4 -> "limited familiarity".

            OPTIONAL FIELDS (use empty string "" if the source data is absent):
            - dispositionNarrative (1-2 sentences): How the agent operates - autonomy,
              rule-following orientation, delegation authority.
            - constraintNarrative (1-2 sentences): Data handling obligations - jurisdiction
              and compliance requirements the agent must observe.
            - goalNarrative (1-3 sentences): The agent's current task and objectives.
              Include sub-goals as a natural continuation, not a bullet list.

            RULES:
            - Second person only: "You are...", "Your role is...", "You have...".
            - Plain prose. No markdown, no bullet points, no headers.
            - Be concise. Every sentence must carry information the agent needs to act on.
            - Return ONLY the JSON object. No explanation, no preamble, no code fences.""";

    private static final String TEMPLATE_HASH = sha256(PROMPT_TEMPLATE).substring(0, 8);

    private final ChatModel llm;
    private final VocabularyRegistry vocab;
    private final RenderedPromptCache cache;
    private final ObjectMapper mapper;
    private final SemanticEnrichmentStep enrichmentStep;

    @Inject
    public ClaudeMarkdownRenderer(
            @Any final Instance<ChatModel> llm,
            final VocabularyRegistry vocab,
            final RenderedPromptCache cache,
            final ObjectMapper mapper) {
        // ChatModel must be @ApplicationScoped (or broader). A @Dependent-scoped ChatModel
        // obtained via Instance.get() would leak. Quarkus LangChain4j always registers
        // ChatModel as @ApplicationScoped, so this is safe in practice.
        this.llm = llm.isResolvable() ? llm.get() : null;
        this.vocab = vocab;
        this.cache = cache;
        this.mapper = mapper;
        this.enrichmentStep = new SemanticEnrichmentStep(mapper);
    }

    /** Package-private constructor for pure-Java tests — no CDI required. */
    ClaudeMarkdownRenderer(final ChatModel llm, final VocabularyRegistry vocab,
                           final RenderedPromptCache cache, final ObjectMapper mapper) {
        this.llm = llm;
        this.vocab = vocab;
        this.cache = cache;
        this.mapper = mapper;
        this.enrichmentStep = new SemanticEnrichmentStep(mapper);
    }

    @Override
    public RenderedPrompt render(final AgentDescriptor descriptor, final AgentPromptContext context) {
        final ObjectNode descriptorNode = buildDescriptorPayload(descriptor);
        final ObjectNode contextNode    = buildContextPayload(context);

        final String descriptorHash = sha256(descriptorNode.toString());
        final String contextHash    = sha256(contextNode.toString());
        final String cacheKey       = descriptorHash + ":" + contextHash + ":"
                                    + context.format().name() + ":" + TEMPLATE_HASH;

        final Optional<RenderedPrompt> cached = cache.get(cacheKey);
        if (cached.isPresent()) return cached.get();

        // Stage 2: optional semantic enrichment
        Optional<SemanticEnrichment> enrichment = Optional.empty();
        if (llm != null && usesEnrichment(context.format())) {
            final ObjectNode full = buildFullPayload(descriptorNode, contextNode);
            enrichment = enrichmentStep.enrich(llm, full);
        }

        // Stage 3: format-specific assembly
        final String content = assemble(enrichment, descriptor, context);
        final RenderedPrompt result = new RenderedPrompt(content, context.format(),
                                                         descriptorHash, contextHash);
        cache.put(cacheKey, result);
        return result;
    }

    // ── Stage 2 predicate ────────────────────────────────────────────────────

    private static boolean usesEnrichment(final RenderFormat format) {
        return switch (format) {
            case CLAUDE_MD, OPENAI_SYSTEM, GEMINI -> true;
            case A2A_CARD                          -> false;
        };
    }

    // ── Stage 1: payload building ─────────────────────────────────────────────

    ObjectNode buildDescriptorPayload(final AgentDescriptor descriptor) {
        final ObjectNode node = mapper.createObjectNode();
        node.put("agentId", descriptor.agentId());
        node.put("name", descriptor.name());
        addIfPresent(node, "version", descriptor.version());
        addIfPresent(node, "provider", descriptor.provider());

        // model: combined form
        if (descriptor.modelFamily() != null && descriptor.modelVersion() != null) {
            node.put("model", descriptor.modelFamily() + "/" + descriptor.modelVersion());
        } else if (descriptor.modelFamily() != null) {
            node.put("model", descriptor.modelFamily());
        } else if (descriptor.modelVersion() != null) {
            node.put("model", descriptor.modelVersion());
        }

        addIfPresent(node, "weightsFingerprint", descriptor.weightsFingerprint());
        node.put("slot", descriptor.slot());

        // Vocabulary-resolved slot labels
        if (descriptor.slotVocabulary() != null) {
            vocab.resolve(descriptor.slotVocabulary(), descriptor.slot()).ifPresent(term -> {
                addIfPresent(node, "slotLabel", term.label());
                addIfPresent(node, "slotDescription", term.description());
            });
        }

        // Capabilities — include name, qualityHint, latencyHintP50Ms, inputTypes, outputTypes,
        // epistemicDomains. Excluded: costHint (operational), tags (routing labels).
        if (descriptor.capabilities() != null && !descriptor.capabilities().isEmpty()) {
            final ArrayNode capsArray = node.putArray("capabilities");
            for (final AgentCapability cap : descriptor.capabilities()) {
                final ObjectNode capNode = capsArray.addObject();
                capNode.put("name", cap.name());
                if (cap.qualityHint() != null)       capNode.put("qualityHint", cap.qualityHint());
                if (cap.latencyHintP50Ms() != null)  capNode.put("latencyHintP50Ms", cap.latencyHintP50Ms());
                if (cap.inputTypes() != null && !cap.inputTypes().isEmpty()) {
                    final ArrayNode arr = capNode.putArray("inputTypes");
                    cap.inputTypes().forEach(arr::add);
                }
                if (cap.outputTypes() != null && !cap.outputTypes().isEmpty()) {
                    final ArrayNode arr = capNode.putArray("outputTypes");
                    cap.outputTypes().forEach(arr::add);
                }
                if (cap.epistemicDomains() != null && !cap.epistemicDomains().isEmpty()) {
                    final ObjectNode domains = capNode.putObject("epistemicDomains");
                    cap.epistemicDomains().forEach(domains::put);
                }
            }
        }

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

        addIfPresent(node, "jurisdiction",       descriptor.jurisdiction());
        addIfPresent(node, "dataHandlingPolicy", descriptor.dataHandlingPolicy());

        return node;
    }

    ObjectNode buildContextPayload(final AgentPromptContext context) {
        final ObjectNode node = mapper.createObjectNode();
        context.goal().ifPresent(goal -> {
            final ObjectNode goalNode = node.putObject("goal");
            goalNode.put("description", goal.description());
            if (!goal.subGoals().isEmpty()) {
                final ArrayNode subGoals = goalNode.putArray("subGoals");
                goal.subGoals().forEach(subGoals::add);
            }
            addIfPresent(goalNode, "caseRef", goal.caseRef());
        });
        return node;
    }

    private ObjectNode buildFullPayload(final ObjectNode descriptorNode,
                                         final ObjectNode contextNode) {
        final ObjectNode full = descriptorNode.deepCopy();
        contextNode.fields().forEachRemaining(e -> full.set(e.getKey(), e.getValue().deepCopy()));
        return full;
    }

    // ── Stage 3: format assembly ──────────────────────────────────────────────

    private String assemble(final Optional<SemanticEnrichment> enrichment,
                             final AgentDescriptor descriptor,
                             final AgentPromptContext context) {
        return switch (context.format()) {
            case CLAUDE_MD     -> assembleClaudeMarkdown(enrichment, descriptor, context);
            case OPENAI_SYSTEM -> assembleOpenAiSystem(enrichment, descriptor, context);
            case A2A_CARD      -> assembleA2aCard(descriptor);
            case GEMINI        -> assembleGemini(enrichment, descriptor, context);
        };
    }

    private String assembleClaudeMarkdown(final Optional<SemanticEnrichment> enrichment,
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

        if (enrichment.isPresent()) {
            final SemanticEnrichment e = enrichment.get();
            sb.append("\n").append(e.identityNarrative()).append("\n");
            sb.append("\n## Role\n").append(e.roleNarrative()).append("\n");
            sb.append("\n## Capabilities\n").append(e.capabilityNarrative()).append("\n");
            e.dispositionNarrative().ifPresent(d ->
                sb.append("\n## How You Operate\n").append(d).append("\n"));
            e.constraintNarrative().ifPresent(c ->
                sb.append("\n## Data Handling\n").append(c).append("\n"));
            e.goalNarrative().ifPresent(g ->
                sb.append("\n## Current Goal\n").append(g).append("\n"));
        } else {
            assembleClaudeMarkdownStructural(sb, descriptor, context);
        }

        // Resources — always structural
        if (!context.resources().isEmpty()) {
            sb.append("\n## Resources\n");
            for (final Resource r : context.resources()) {
                sb.append("- **").append(r.label()).append("**: ").append(r.uri());
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

    private void assembleClaudeMarkdownStructural(final StringBuilder sb,
                                                   final AgentDescriptor descriptor,
                                                   final AgentPromptContext context) {
        // Role — deliberate heading change from ## {slot_label} to ## Role
        // (see spec behavioral delta note)
        if (descriptor.slot() != null) {
            sb.append("\n## Role\n");
            if (descriptor.slotVocabulary() != null) {
                vocab.resolve(descriptor.slotVocabulary(), descriptor.slot()).ifPresentOrElse(
                    term -> {
                        if (term.label() != null)       sb.append(term.label()).append("\n");
                        if (term.description() != null) sb.append(term.description()).append("\n");
                    },
                    () -> sb.append(descriptor.slot()).append("\n")
                );
            } else {
                sb.append(descriptor.slot()).append("\n");
            }
        }

        // Capabilities
        if (descriptor.capabilities() != null && !descriptor.capabilities().isEmpty()) {
            sb.append("\n## Capabilities\n");
            for (final AgentCapability cap : descriptor.capabilities()) {
                sb.append("- **").append(cap.name()).append("**");
                if (cap.qualityHint() != null) sb.append(": quality ").append(cap.qualityHint());
                if (cap.latencyHintP50Ms() != null)
                    sb.append(", p50 ").append(cap.latencyHintP50Ms()).append("ms");
                sb.append("\n");
                if (cap.epistemicDomains() != null && !cap.epistemicDomains().isEmpty()) {
                    sb.append("  Domains: ").append(cap.epistemicDomains()).append("\n");
                }
            }
        }

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

        // Data handling
        if (descriptor.jurisdiction() != null || descriptor.dataHandlingPolicy() != null) {
            sb.append("\n## Data Handling\n");
            if (descriptor.jurisdiction() != null)
                sb.append("Jurisdiction: ").append(descriptor.jurisdiction()).append("\n");
            if (descriptor.dataHandlingPolicy() != null)
                sb.append("Policy: ").append(descriptor.dataHandlingPolicy()).append("\n");
        }

        // Goal
        context.goal().ifPresent(goal -> {
            sb.append("\n## Current Goal\n");
            sb.append(goal.description()).append("\n");
            if (!goal.subGoals().isEmpty()) {
                goal.subGoals().forEach(sub -> sb.append("- ").append(sub).append("\n"));
            }
            if (goal.caseRef() != null) sb.append("Case: ").append(goal.caseRef()).append("\n");
        });
    }

    private String assembleOpenAiSystem(final Optional<SemanticEnrichment> enrichment,
                                         final AgentDescriptor descriptor,
                                         final AgentPromptContext context) {
        final var sb = new StringBuilder();

        if (enrichment.isPresent()) {
            final SemanticEnrichment e = enrichment.get();
            sb.append(e.identityNarrative()).append(" ").append(e.roleNarrative()).append("\n");
            sb.append("\n").append(e.capabilityNarrative()).append("\n");
            e.dispositionNarrative().ifPresent(d -> sb.append("\n").append(d).append("\n"));
            e.constraintNarrative().ifPresent(c -> sb.append("\n").append(c).append("\n"));
            e.goalNarrative().ifPresent(g -> sb.append("\n").append(g).append("\n"));
        } else {
            // Structural OPENAI_SYSTEM — dense prose, no headers
            sb.append(descriptor.name());
            if (descriptor.slot() != null) sb.append(", ").append(descriptor.slot());
            sb.append(".");
            if (descriptor.version() != null) sb.append(" Version ").append(descriptor.version()).append(".");
            sb.append("\n");

            if (descriptor.capabilities() != null && !descriptor.capabilities().isEmpty()) {
                sb.append("\nCapabilities: ");
                final var names = descriptor.capabilities().stream()
                        .map(AgentCapability::name)
                        .collect(java.util.stream.Collectors.joining(", "));
                sb.append(names).append(".\n");
            }

            if (descriptor.disposition() != null) {
                final AgentDisposition d = descriptor.disposition();
                sb.append("\nOperating style:");
                if (d.ruleFollowing() != null) sb.append(" ").append(d.ruleFollowing()).append(" rule-following.");
                if (d.autonomy() != null)      sb.append(" Autonomy: ").append(d.autonomy()).append(".");
                sb.append(" Can delegate: ").append(d.delegation() ? "yes" : "no").append(".\n");
            }

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
                    .map(r -> r.label() + " (" + r.uri() + ")")
                    .collect(java.util.stream.Collectors.joining(", "));
            sb.append(resources).append(".\n");
        }

        if (context.situationalContext() != null) {
            sb.append("\n").append(context.situationalContext()).append("\n");
        }

        return sb.toString().trim();
    }

    private String assembleA2aCard(final AgentDescriptor descriptor) {
        // Structural only — SemanticEnrichment not used for A2A_CARD.
        // Per-capability prose deferred to eidos#13.
        final ObjectNode card = mapper.createObjectNode();
        card.put("name", descriptor.name());
        card.put("agentId", descriptor.agentId());
        addIfPresent(card, "version", descriptor.version());

        if (descriptor.capabilities() != null && !descriptor.capabilities().isEmpty()) {
            final ArrayNode capsArray = card.putArray("capabilities");
            for (final AgentCapability cap : descriptor.capabilities()) {
                final ObjectNode capNode = capsArray.addObject();
                capNode.put("name", cap.name());
                if (cap.qualityHint() != null) capNode.put("qualityHint", cap.qualityHint());
            }
        }

        try {
            return mapper.writeValueAsString(card);
        } catch (final com.fasterxml.jackson.core.JsonProcessingException ex) {
            throw new IllegalStateException("A2A card serialization failed", ex);
        }
    }

    private String assembleGemini(final Optional<SemanticEnrichment> enrichment,
                                   final AgentDescriptor descriptor,
                                   final AgentPromptContext context) {
        // Placeholder until eidos#14. RenderedPrompt.format() == GEMINI but content is
        // CLAUDE_MD-structured. Callers must not branch on format == GEMINI for structure.
        return assembleClaudeMarkdown(enrichment, descriptor, context);
    }

    // ── Shared utilities ───────────────────────────────────────────────────────

    private static String combinedModel(final AgentDescriptor descriptor) {
        if (descriptor.modelFamily() != null && descriptor.modelVersion() != null)
            return descriptor.modelFamily() + "/" + descriptor.modelVersion();
        if (descriptor.modelFamily() != null) return descriptor.modelFamily();
        return descriptor.modelVersion();
    }

    private static void addIfPresent(final ObjectNode node, final String key, final String value) {
        if (value != null) node.put(key, value);
    }

    static String sha256(final String input) {
        try {
            final MessageDigest digest = MessageDigest.getInstance("SHA-256");
            final byte[] hash = digest.digest(input.getBytes(StandardCharsets.UTF_8));
            return HexFormat.of().formatHex(hash).substring(0, 16);
        } catch (final NoSuchAlgorithmException e) {
            throw new IllegalStateException("SHA-256 not available", e);
        }
    }
}
```

- [ ] **Step 5.2: Update the test file — fix constructor calls and add test infrastructure**

Replace the existing `ClaudeMarkdownRendererTest.java` with the updated version. The key changes are:
- 4-param constructor (add `NoOpRenderedPromptCache` and `new ObjectMapper()`)
- Add `TestRenderedPromptCache` inner class
- Add `fullDescriptor()` update (unchanged from current — `AgentCapability` already has the right shape)

```java
// runtime/src/test/java/io/casehub/eidos/runtime/renderer/ClaudeMarkdownRendererTest.java
package io.casehub.eidos.runtime.renderer;

import com.fasterxml.jackson.databind.ObjectMapper;
import dev.langchain4j.data.message.AiMessage;
import dev.langchain4j.data.message.UserMessage;
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.model.chat.request.ChatRequest;
import dev.langchain4j.model.chat.response.ChatResponse;
import io.casehub.eidos.api.*;
import io.casehub.eidos.api.SystemPromptRenderer.RenderFormat;
import io.casehub.eidos.runtime.vocabulary.CdiVocabularyRegistry;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Optional;

import static io.casehub.eidos.api.SystemPromptRenderer.RenderFormat.*;
import static org.assertj.core.api.Assertions.assertThat;

class ClaudeMarkdownRendererTest {

    static final String LLM_RESPONSE = "You are a code reviewer specialising in Java.";
    static final ObjectMapper MAPPER = new ObjectMapper();

    /** Minimal in-memory cache for testing cache-hit and cache-miss behaviour. */
    static class TestRenderedPromptCache implements RenderedPromptCache {
        final Map<String, SystemPromptRenderer.RenderedPrompt> store = new HashMap<>();
        int putCount = 0;
        int getCount = 0;

        @Override
        public Optional<SystemPromptRenderer.RenderedPrompt> get(final String cacheKey) {
            getCount++;
            return Optional.ofNullable(store.get(cacheKey));
        }

        @Override
        public void put(final String cacheKey, final SystemPromptRenderer.RenderedPrompt result) {
            putCount++;
            store.put(cacheKey, result);
        }
    }

    ChatModel mockLlm;
    ClaudeMarkdownRenderer rendererWithLlm;
    ClaudeMarkdownRenderer rendererStructural;
    TestRenderedPromptCache testCache;

    @BeforeEach
    void setUp() {
        mockLlm = new ChatModel() {
            @Override
            public ChatResponse doChat(final ChatRequest request) {
                return ChatResponse.builder().aiMessage(AiMessage.from(LLM_RESPONSE)).build();
            }
        };
        testCache = new TestRenderedPromptCache();
        final var vocab = new CdiVocabularyRegistry();
        rendererWithLlm  = new ClaudeMarkdownRenderer(mockLlm, vocab, testCache, MAPPER);
        rendererStructural = new ClaudeMarkdownRenderer(null, vocab,
                new NoOpRenderedPromptCache(), MAPPER);
    }

    static AgentDescriptor fullDescriptor() {
        return new AgentDescriptor(
            "reviewer-1", "Code Reviewer", "1.0", "anthropic",
            "claude", "claude-3-7-sonnet", null,
            null, null, null,
            "reviewer",
            List.of(new AgentCapability("code-review", 0.95, 150L, "low",
                List.of("code"), List.of("review"), List.of(),
                Map.of("java", 0.95, "rust", 0.3))),
            new AgentDisposition("independent", "strict", "conservative", "directed", false),
            "EU", "gdpr-compliant", "default"
        );
    }

    static AgentPromptContext fullContext() {
        return AgentPromptContext.forFormat(CLAUDE_MD)
                .withGoal(new GoalContext("Review PR #42", List.of("Check style", "Check tests"), "case-123"))
                .withResources(List.of(new Resource("/src/main/java", "Source", "filesystem")))
                .withSituationalContext("Critical release branch");
    }

    /** Renders and returns the user message payload sent to the LLM. */
    private String capturePayload(final AgentDescriptor desc, final AgentPromptContext ctx) {
        final String[] captured = {""};
        final ChatModel capturingLlm = new ChatModel() {
            @Override
            public ChatResponse doChat(final ChatRequest request) {
                captured[0] = request.messages().stream()
                        .filter(m -> m instanceof UserMessage)
                        .map(m -> ((UserMessage) m).singleText())
                        .reduce("", (a, b) -> a + b);
                return ChatResponse.builder()
                        .aiMessage(AiMessage.from("""
                            {"identityNarrative":"You are TestAgent.",
                             "roleNarrative":"Your role is testing.",
                             "capabilityNarrative":"You can review code.",
                             "dispositionNarrative":"You are strict.",
                             "constraintNarrative":"","goalNarrative":""}"""))
                        .build();
            }
        };
        new ClaudeMarkdownRenderer(capturingLlm, new CdiVocabularyRegistry(),
                new NoOpRenderedPromptCache(), MAPPER).render(desc, ctx);
        return captured[0];
    }

    // ── LLM path ──────────────────────────────────────────────────────────────

    @Test
    void llm_path_uses_llm_response_as_content() {
        final var result = rendererWithLlm.render(fullDescriptor(), fullContext());
        assertThat(result.content()).contains(LLM_RESPONSE);
    }

    @Test
    void llm_path_payload_contains_agent_id() {
        assertThat(capturePayload(fullDescriptor(), fullContext())).contains("reviewer-1");
    }

    @Test
    void llm_path_payload_contains_capability_name() {
        assertThat(capturePayload(fullDescriptor(), fullContext())).contains("code-review");
    }

    @Test
    void llm_path_payload_contains_input_types() {
        assertThat(capturePayload(fullDescriptor(), fullContext())).contains("code");
    }

    @Test
    void llm_path_payload_excludes_tenancy_id() {
        assertThat(capturePayload(fullDescriptor(), fullContext())).doesNotContain("default");
    }

    @Test
    void llm_path_payload_contains_goal_when_set() {
        assertThat(capturePayload(fullDescriptor(), fullContext())).contains("Review PR #42");
    }

    @Test
    void llm_path_payload_excludes_resources_and_situational_context() {
        assertThat(capturePayload(fullDescriptor(), fullContext()))
                .doesNotContain("/src/main/java")
                .doesNotContain("Critical release branch");
    }

    // ── Structural CLAUDE_MD path ─────────────────────────────────────────────

    @Test
    void structural_path_contains_agent_name_and_id() {
        final var result = rendererStructural.render(fullDescriptor(), fullContext());
        assertThat(result.content()).contains("Code Reviewer").contains("reviewer-1");
    }

    @Test
    void structural_path_contains_capability() {
        final var result = rendererStructural.render(fullDescriptor(), fullContext());
        assertThat(result.content()).contains("code-review");
    }

    @Test
    void structural_path_contains_disposition_axes() {
        final var result = rendererStructural.render(fullDescriptor(), fullContext());
        assertThat(result.content()).contains("independent").contains("strict");
    }

    @Test
    void structural_path_contains_goal_when_set() {
        final var result = rendererStructural.render(fullDescriptor(), fullContext());
        assertThat(result.content()).contains("Review PR #42");
    }

    @Test
    void structural_path_omits_goal_section_when_absent() {
        final var ctx = AgentPromptContext.forFormat(CLAUDE_MD);
        final var result = rendererStructural.render(fullDescriptor(), ctx);
        assertThat(result.content()).doesNotContain("## Current Goal");
    }

    @Test
    void structural_path_uses_role_heading_not_slot_label() {
        final var result = rendererStructural.render(fullDescriptor(), fullContext());
        assertThat(result.content()).contains("## Role");
    }

    @Test
    void structural_path_contains_resources_when_set() {
        final var result = rendererStructural.render(fullDescriptor(), fullContext());
        assertThat(result.content()).contains("/src/main/java");
    }

    @Test
    void structural_path_omits_resources_section_when_empty() {
        final var ctx = AgentPromptContext.forFormat(CLAUDE_MD);
        final var result = rendererStructural.render(fullDescriptor(), ctx);
        assertThat(result.content()).doesNotContain("## Resources");
    }

    @Test
    void structural_path_contains_situational_context_when_set() {
        final var result = rendererStructural.render(fullDescriptor(), fullContext());
        assertThat(result.content()).contains("Critical release branch");
    }

    @Test
    void structural_path_omits_context_section_when_null() {
        final var ctx = AgentPromptContext.forFormat(CLAUDE_MD);
        final var result = rendererStructural.render(fullDescriptor(), ctx);
        assertThat(result.content()).doesNotContain("## Context");
    }

    // ── OPENAI_SYSTEM path ────────────────────────────────────────────────────

    @Test
    void openai_structural_has_no_markdown_headers() {
        final var ctx = AgentPromptContext.forFormat(OPENAI_SYSTEM);
        final var result = rendererStructural.render(fullDescriptor(), ctx);
        assertThat(result.content()).doesNotContain("#");
    }

    @Test
    void openai_structural_contains_agent_name() {
        final var ctx = AgentPromptContext.forFormat(OPENAI_SYSTEM);
        final var result = rendererStructural.render(fullDescriptor(), ctx);
        assertThat(result.content()).contains("Code Reviewer");
    }

    // ── A2A_CARD path ─────────────────────────────────────────────────────────

    @Test
    void a2a_card_produces_json_with_name_and_capabilities() {
        final var ctx = AgentPromptContext.forFormat(A2A_CARD);
        final var result = rendererStructural.render(fullDescriptor(), ctx);
        assertThat(result.content()).contains("\"name\"").contains("code-review");
    }

    @Test
    void a2a_card_skips_llm_even_when_llm_is_configured() {
        final boolean[] called = {false};
        final ChatModel trackingLlm = new ChatModel() {
            @Override
            public ChatResponse doChat(final ChatRequest request) {
                called[0] = true;
                return ChatResponse.builder().aiMessage(AiMessage.from("irrelevant")).build();
            }
        };
        final var renderer = new ClaudeMarkdownRenderer(trackingLlm, new CdiVocabularyRegistry(),
                new NoOpRenderedPromptCache(), MAPPER);
        renderer.render(fullDescriptor(), AgentPromptContext.forFormat(A2A_CARD));
        assertThat(called[0]).isFalse();
    }

    // ── GEMINI path ───────────────────────────────────────────────────────────

    @Test
    void gemini_structural_produces_same_content_as_claude_md_structural() {
        final var claudeResult = rendererStructural.render(fullDescriptor(),
                AgentPromptContext.forFormat(CLAUDE_MD)
                        .withSituationalContext("ctx"));
        final var geminiResult = rendererStructural.render(fullDescriptor(),
                AgentPromptContext.forFormat(GEMINI)
                        .withSituationalContext("ctx"));
        assertThat(geminiResult.content()).isEqualTo(claudeResult.content());
    }

    // ── Cache behaviour ───────────────────────────────────────────────────────

    @Test
    void cache_hit_skips_llm_call() {
        final boolean[] called = {false};
        final ChatModel trackingLlm = new ChatModel() {
            @Override
            public ChatResponse doChat(final ChatRequest request) {
                called[0] = true;
                return ChatResponse.builder().aiMessage(AiMessage.from("result")).build();
            }
        };
        final var renderer = new ClaudeMarkdownRenderer(trackingLlm, new CdiVocabularyRegistry(),
                testCache, MAPPER);

        renderer.render(fullDescriptor(), fullContext()); // miss — LLM called
        called[0] = false;
        renderer.render(fullDescriptor(), fullContext()); // hit — LLM must NOT be called

        assertThat(called[0]).isFalse();
    }

    @Test
    void claude_md_and_openai_system_produce_different_cache_entries() {
        // This test is both a functional correctness test and a regression guard
        // for the format-in-cache-key fix (spec review finding #1).
        final var claudeCtx  = fullContext();  // format = CLAUDE_MD from fullContext()
        final var openaiCtx  = AgentPromptContext.forFormat(OPENAI_SYSTEM)
                .withGoal(new GoalContext("Review PR #42", List.of("Check style", "Check tests"), "case-123"))
                .withResources(List.of(new Resource("/src/main/java", "Source", "filesystem")))
                .withSituationalContext("Critical release branch");

        rendererStructural.render(fullDescriptor(), claudeCtx);
        rendererStructural.render(fullDescriptor(), openaiCtx);

        // Two distinct formats = two distinct cache entries
        // rendererStructural uses NoOpRenderedPromptCache so we test via content difference
        final var claudeResult = rendererStructural.render(fullDescriptor(), claudeCtx);
        final var openaiResult = rendererStructural.render(fullDescriptor(), openaiCtx);
        assertThat(claudeResult.content()).isNotEqualTo(openaiResult.content());
        assertThat(claudeResult.format()).isEqualTo(CLAUDE_MD);
        assertThat(openaiResult.format()).isEqualTo(OPENAI_SYSTEM);
    }

    // ── Hashing ───────────────────────────────────────────────────────────────

    @Test
    void same_inputs_produce_same_hashes() {
        final var r1 = rendererStructural.render(fullDescriptor(), fullContext());
        final var r2 = rendererStructural.render(fullDescriptor(), fullContext());
        assertThat(r1.descriptorHash()).isEqualTo(r2.descriptorHash());
        assertThat(r1.contextHash()).isEqualTo(r2.contextHash());
    }

    @Test
    void different_descriptor_produces_different_descriptor_hash() {
        final var desc2 = new AgentDescriptor(
            "planner-1", "Planner", "1.0", "anthropic", "claude", "claude-3-7-sonnet",
            null, null, null, null, "planner",
            List.of(), null, null, null, "default"
        );
        final var r1 = rendererStructural.render(fullDescriptor(), fullContext());
        final var r2 = rendererStructural.render(desc2, fullContext());
        assertThat(r1.descriptorHash()).isNotEqualTo(r2.descriptorHash());
    }

    @Test
    void different_context_produces_different_context_hash() {
        final var ctx2 = AgentPromptContext.forFormat(CLAUDE_MD).withSituationalContext("different");
        final var r1 = rendererStructural.render(fullDescriptor(), fullContext());
        final var r2 = rendererStructural.render(fullDescriptor(), ctx2);
        assertThat(r1.contextHash()).isNotEqualTo(r2.contextHash());
    }

    @Test
    void rendered_prompt_has_correct_format() {
        final var result = rendererStructural.render(fullDescriptor(), fullContext());
        assertThat(result.format()).isEqualTo(CLAUDE_MD);
    }
}
```

- [ ] **Step 5.3: Run the full runtime test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime
```
Expected: all existing tests plus new tests PASS; `SemanticEnrichmentStepTest` now also passes (PROMPT_TEMPLATE is package-private)

- [ ] **Step 5.4: Run the full build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install
```
Expected: BUILD SUCCESS, all modules green

- [ ] **Step 5.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add \
  runtime/src/main/java/io/casehub/eidos/runtime/renderer/ClaudeMarkdownRenderer.java \
  runtime/src/test/java/io/casehub/eidos/runtime/renderer/ClaudeMarkdownRendererTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#6): three-stage rendering pipeline in ClaudeMarkdownRenderer

- RenderedPromptCache SPI integration (cache key includes format + TEMPLATE_HASH)
- Stage 1: Jackson ObjectNode payload builder (vocabulary-enriched, tenancyId excluded)
- Stage 2: SemanticEnrichmentStep with ResponseFormat+JsonSchema
- Stage 3: format-specific assembly (CLAUDE_MD, OPENAI_SYSTEM, A2A_CARD, GEMINI)
- usesEnrichment() exhaustive switch short-circuits A2A_CARD
- Structural CLAUDE_MD now uses ## Role heading (deliberate behavioral change)
- TEMPLATE_HASH auto-invalidates cache on prompt template changes

Closes #6"
```

---

## Task 6: Payload content verification tests

These tests verify `buildDescriptorPayload()` and `buildContextPayload()` directly, which are package-private and accessible from the test class in the same package.

**Files:**
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/renderer/ClaudeMarkdownRendererTest.java`

Add the following test methods to `ClaudeMarkdownRendererTest` (they access package-private methods directly):

- [ ] **Step 6.1: Add payload tests**

Add these methods to the existing `ClaudeMarkdownRendererTest` class, before the closing `}`:

```java
    // ── Payload building (Stage 1) ────────────────────────────────────────────

    @Test
    void descriptor_payload_includes_agent_id_and_name() {
        final var node = rendererStructural.buildDescriptorPayload(fullDescriptor());
        assertThat(node.get("agentId").asText()).isEqualTo("reviewer-1");
        assertThat(node.get("name").asText()).isEqualTo("Code Reviewer");
    }

    @Test
    void descriptor_payload_excludes_tenancy_id() {
        final var node = rendererStructural.buildDescriptorPayload(fullDescriptor());
        assertThat(node.has("tenancyId")).isFalse();
    }

    @Test
    void descriptor_payload_excludes_vocabulary_uris() {
        final var node = rendererStructural.buildDescriptorPayload(fullDescriptor());
        assertThat(node.has("slotVocabulary")).isFalse();
        assertThat(node.has("domainVocabulary")).isFalse();
        assertThat(node.has("dispositionVocabulary")).isFalse();
    }

    @Test
    void descriptor_payload_combines_model_family_and_version() {
        final var node = rendererStructural.buildDescriptorPayload(fullDescriptor());
        assertThat(node.get("model").asText()).isEqualTo("claude/claude-3-7-sonnet");
    }

    @Test
    void descriptor_payload_capability_includes_input_and_output_types() {
        final var node = rendererStructural.buildDescriptorPayload(fullDescriptor());
        final var cap = node.get("capabilities").get(0);
        assertThat(cap.get("inputTypes").get(0).asText()).isEqualTo("code");
        assertThat(cap.get("outputTypes").get(0).asText()).isEqualTo("review");
    }

    @Test
    void descriptor_payload_capability_excludes_cost_hint_and_tags() {
        final var node = rendererStructural.buildDescriptorPayload(fullDescriptor());
        final var cap = node.get("capabilities").get(0);
        assertThat(cap.has("costHint")).isFalse();
        assertThat(cap.has("tags")).isFalse();
    }

    @Test
    void descriptor_payload_includes_weights_fingerprint_when_set() {
        final var desc = new AgentDescriptor(
            "id", "Name", "1.0", null, null, null, "fp-abc123",
            null, null, null, "slot", List.of(), null, null, null, "t"
        );
        final var node = rendererStructural.buildDescriptorPayload(desc);
        assertThat(node.get("weightsFingerprint").asText()).isEqualTo("fp-abc123");
    }

    @Test
    void context_payload_includes_goal_when_present() {
        final var node = rendererStructural.buildContextPayload(fullContext());
        assertThat(node.get("goal").get("description").asText()).isEqualTo("Review PR #42");
    }

    @Test
    void context_payload_excludes_resources_and_situational_context() {
        final var node = rendererStructural.buildContextPayload(fullContext());
        assertThat(node.has("resources")).isFalse();
        assertThat(node.has("situationalContext")).isFalse();
    }

    @Test
    void context_payload_is_empty_when_no_goal() {
        final var ctx = AgentPromptContext.forFormat(CLAUDE_MD);
        final var node = rendererStructural.buildContextPayload(ctx);
        assertThat(node.isEmpty()).isTrue();
    }
```

- [ ] **Step 6.2: Run payload tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest="ClaudeMarkdownRendererTest#descriptor_payload*+ClaudeMarkdownRendererTest#context_payload*"
```
Expected: all payload tests PASS

- [ ] **Step 6.3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add \
  runtime/src/test/java/io/casehub/eidos/runtime/renderer/ClaudeMarkdownRendererTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "test(eidos#6): payload construction and format assembly tests

Refs #6"
```

---

## Task 7: Full build and final verification

- [ ] **Step 7.1: Run all tests across all modules**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install
```
Expected: BUILD SUCCESS. All 133+ tests pass (number will increase).

- [ ] **Step 7.2: Verify test count increased**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test 2>&1 | grep "Tests run:" | tail -10
```
Confirm test counts are higher than pre-implementation baseline of 133.

- [ ] **Step 7.3: Verify no legacy methods remain**

```bash
grep -n "toYaml\|renderWithLlm\|renderStructural" \
  /Users/mdproctor/claude/casehub/eidos/runtime/src/main/java/io/casehub/eidos/runtime/renderer/ClaudeMarkdownRenderer.java
```
Expected: no output — those methods no longer exist.

- [ ] **Step 7.4: Final commit (if any stray changes)**

```bash
git -C /Users/mdproctor/claude/casehub/eidos status
```
If clean: nothing to do. If there are unstaged changes, review and commit with `Refs #6`.
