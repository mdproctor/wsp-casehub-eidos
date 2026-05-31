# Quality Eval, Prompt Security, Reactive Cache — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement eidos#16 (offline quality eval harness), eidos#15 (AgentDescriptor field validation), and eidos#19 (ReactiveRenderedPromptCache SPI) with a shared pipeline refactor as the foundation.

**Architecture:** Extract `StageOneResult` and `buildStage1()` to `EidosRenderPipeline` (eliminating code duplication), remove cache from the pipeline entirely, add `ReactiveRenderedPromptCache` as the canonical cache SPI backed by a `BlockingToReactiveRenderedPromptCacheAdapter`, and add `AgentDescriptor` compact-constructor validation. The eval harness lives in a new top-level `eval/` Maven module that is never deployed.

**Tech Stack:** Java 21, Quarkus 3.32.2, LangChain4j 1.14.1, Mutiny, JUnit 5, AssertJ, ArchUnit, Jackson

**Spec:** `specs/2026-05-30-quality-eval-prompt-security-reactive-cache-design.md` (workspace)

---

## File Map

### New files
- `api/src/main/java/io/casehub/eidos/api/ReactiveRenderedPromptCache.java` — canonical reactive cache SPI
- `api/src/main/java/io/casehub/eidos/api/AgentDescriptorValidationException.java` — public validation exception
- `api/src/main/java/io/casehub/eidos/api/AgentDescriptorValidator.java` — package-private validator
- `runtime/src/main/java/io/casehub/eidos/runtime/renderer/StageOneResult.java` — package-private record (promoted from private inner class)
- `runtime/src/main/java/io/casehub/eidos/runtime/cache/BlockingToReactiveRenderedPromptCacheAdapter.java` — `@DefaultBean` adapter
- `runtime/src/test/java/io/casehub/eidos/runtime/cache/BlockingToReactiveRenderedPromptCacheAdapterTest.java`
- `runtime/src/test/java/io/casehub/eidos/runtime/renderer/TestReactiveRenderedPromptCache.java` — test double
- `api/src/test/java/io/casehub/eidos/api/AgentDescriptorValidatorTest.java`
- `eval/pom.xml`
- `eval/src/main/java/io/casehub/eidos/eval/EvalCase.java`
- `eval/src/main/java/io/casehub/eidos/eval/EvalDimension.java`
- `eval/src/main/java/io/casehub/eidos/eval/EvalScore.java`
- `eval/src/main/java/io/casehub/eidos/eval/EvalResult.java`
- `eval/src/main/java/io/casehub/eidos/eval/EvalSummary.java`
- `eval/src/main/java/io/casehub/eidos/eval/EvalReport.java`
- `eval/src/main/java/io/casehub/eidos/eval/EvalDataset.java`
- `eval/src/main/java/io/casehub/eidos/eval/PromptJudge.java`
- `eval/src/main/java/io/casehub/eidos/eval/EvalReportWriter.java`
- `eval/src/test/java/io/casehub/eidos/eval/PromptJudgeTest.java`
- `eval/src/test/java/io/casehub/eidos/eval/EvalDatasetTest.java`
- `eval/src/test/java/io/casehub/eidos/eval/EvalReportWriterTest.java`
- `eval/src/test/java/io/casehub/eidos/eval/EvalResultCompletenessTest.java`
- `eval/src/test/java/io/casehub/eidos/eval/EvalProfile.java`
- `eval/src/test/java/io/casehub/eidos/eval/PromptEvalTest.java`
- `eval/src/test/resources/application.properties`
- `eval/src/test/resources/application-eval.properties`

### Modified files
- `pom.xml` — add `<module>eval</module>`
- `runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosRenderPipeline.java` — remove cache, add `buildStage1()`, change `assemble()` signature
- `runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosSystemPromptRenderer.java` — inject `ReactiveRenderedPromptCache`, use `buildStage1()`
- `runtime/src/main/java/io/casehub/eidos/runtime/renderer/DefaultReactiveSystemPromptRenderer.java` — inject `ReactiveRenderedPromptCache`, use `buildStage1()`
- `api/src/main/java/io/casehub/eidos/api/AgentDescriptor.java` — add compact constructor
- `runtime/src/test/java/io/casehub/eidos/runtime/renderer/EidosRenderPipelineTest.java` — drop cache arg, add `buildStage1()` tests
- `runtime/src/test/java/io/casehub/eidos/runtime/renderer/EidosSystemPromptRendererTest.java` — inject `TestReactiveRenderedPromptCache`
- `runtime/src/test/java/io/casehub/eidos/runtime/renderer/DefaultReactiveSystemPromptRendererStreamingTest.java` — restructure cache-hit test

---

## Task 1: `ReactiveRenderedPromptCache` SPI

**Files:**
- Create: `api/src/main/java/io/casehub/eidos/api/ReactiveRenderedPromptCache.java`

- [ ] **Step 1: Create the interface**

```java
package io.casehub.eidos.api;

import io.smallrye.mutiny.Uni;
import java.util.Optional;

public interface ReactiveRenderedPromptCache {

    /**
     * Returns the cached prompt for the given key.
     * On failure, implementations must recover and return Uni of Optional.empty() —
     * a cache miss must never abort a render.
     */
    Uni<Optional<SystemPromptRenderer.RenderedPrompt>> get(String cacheKey);

    /**
     * Stores a rendered prompt. Must not emit a failure — implementations recover
     * internally so a cache write failure never aborts a render.
     */
    Uni<Void> put(String cacheKey, SystemPromptRenderer.RenderedPrompt result);
}
```

- [ ] **Step 2: Build to verify it compiles**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api -q
```

Expected: BUILD SUCCESS

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add api/src/main/java/io/casehub/eidos/api/ReactiveRenderedPromptCache.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#19): add ReactiveRenderedPromptCache SPI

Refs #19"
```

---

## Task 2: `BlockingToReactiveRenderedPromptCacheAdapter` + tests

**Files:**
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/cache/BlockingToReactiveRenderedPromptCacheAdapter.java`
- Create: `runtime/src/test/java/io/casehub/eidos/runtime/cache/BlockingToReactiveRenderedPromptCacheAdapterTest.java`

- [ ] **Step 1: Write failing tests**

```java
package io.casehub.eidos.runtime.cache;

import io.casehub.eidos.api.RenderedPromptCache;
import io.casehub.eidos.api.ReactiveRenderedPromptCache;
import io.casehub.eidos.api.SystemPromptRenderer.RenderedPrompt;
import io.casehub.eidos.api.SystemPromptRenderer.RenderFormat;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;

class BlockingToReactiveRenderedPromptCacheAdapterTest {

    static final RenderedPrompt PROMPT = new RenderedPrompt("content", RenderFormat.CLAUDE_MD, "dh", "ch");

    ReactiveRenderedPromptCache adapter;

    @BeforeEach
    void setUp() {
        adapter = new BlockingToReactiveRenderedPromptCacheAdapter(new InMemoryBlockingCache());
    }

    @Test
    void get_returns_empty_on_miss() {
        assertThat(adapter.get("missing").await().indefinitely()).isEmpty();
    }

    @Test
    void put_then_get_returns_stored_value() {
        adapter.put("key", PROMPT).await().indefinitely();
        assertThat(adapter.get("key").await().indefinitely()).contains(PROMPT);
    }

    @Test
    void get_returns_empty_when_blocking_cache_throws() {
        final ReactiveRenderedPromptCache failingAdapter =
            new BlockingToReactiveRenderedPromptCacheAdapter(new ThrowingBlockingCache());
        assertThat(failingAdapter.get("key").await().indefinitely()).isEmpty();
    }

    @Test
    void put_completes_when_blocking_cache_throws() {
        final ReactiveRenderedPromptCache failingAdapter =
            new BlockingToReactiveRenderedPromptCacheAdapter(new ThrowingBlockingCache());
        // must not throw — Uni must complete normally
        failingAdapter.put("key", PROMPT).await().indefinitely();
    }

    // ── test doubles ──────────────────────────────────────────────────────────

    static class InMemoryBlockingCache implements RenderedPromptCache {
        private final java.util.Map<String, RenderedPrompt> store = new java.util.HashMap<>();

        @Override public Optional<RenderedPrompt> get(String k) { return Optional.ofNullable(store.get(k)); }
        @Override public void put(String k, RenderedPrompt v)   { store.put(k, v); }
    }

    static class ThrowingBlockingCache implements RenderedPromptCache {
        @Override public Optional<RenderedPrompt> get(String k) { throw new RuntimeException("cache down"); }
        @Override public void put(String k, RenderedPrompt v)   { throw new RuntimeException("cache down"); }
    }
}
```

- [ ] **Step 2: Run to verify FAIL**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=BlockingToReactiveRenderedPromptCacheAdapterTest -q 2>&1 | tail -5
```

Expected: compilation error — `BlockingToReactiveRenderedPromptCacheAdapter` does not exist

- [ ] **Step 3: Create the adapter**

```java
package io.casehub.eidos.runtime.cache;

import io.casehub.eidos.api.ReactiveRenderedPromptCache;
import io.casehub.eidos.api.RenderedPromptCache;
import io.casehub.eidos.api.SystemPromptRenderer.RenderedPrompt;
import io.quarkus.arc.DefaultBean;
import io.smallrye.mutiny.Uni;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.util.Optional;

@DefaultBean
@ApplicationScoped
class BlockingToReactiveRenderedPromptCacheAdapter implements ReactiveRenderedPromptCache {

    private final RenderedPromptCache blocking;

    @Inject
    BlockingToReactiveRenderedPromptCacheAdapter(final RenderedPromptCache blocking) {
        this.blocking = blocking;
    }

    // Package-private constructor for tests (no CDI).
    BlockingToReactiveRenderedPromptCacheAdapter(final RenderedPromptCache blocking,
                                                  @SuppressWarnings("unused") boolean testMarker) {
        this.blocking = blocking;
    }

    @Override
    public Uni<Optional<RenderedPrompt>> get(final String cacheKey) {
        // No runSubscriptionOn — callers (blocking renderer, reactive Stage 1) are already
        // on the worker pool. Adding runSubscriptionOn(workerPool) here risks deadlock
        // under saturation (calling thread blocks waiting for a hop on the same pool).
        return Uni.createFrom()
                  .item(() -> blocking.get(cacheKey))
                  .onFailure().recoverWith(e -> Uni.createFrom().item(Optional.empty()));
    }

    @Override
    public Uni<Void> put(final String cacheKey, final RenderedPrompt result) {
        return Uni.createFrom()
                  .<Void>item(() -> { blocking.put(cacheKey, result); return null; })
                  .onFailure().recoverWithNull()
                  .replaceWithVoid();
    }
}
```

The test uses a package-private constructor via an overload. Update the test's `setUp()` to use it:

```java
adapter = new BlockingToReactiveRenderedPromptCacheAdapter(new InMemoryBlockingCache(), true);
// and in get_returns_empty_when_blocking_cache_throws and put_completes:
new BlockingToReactiveRenderedPromptCacheAdapter(new ThrowingBlockingCache(), true);
```

- [ ] **Step 4: Run tests — verify PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=BlockingToReactiveRenderedPromptCacheAdapterTest -q
```

Expected: BUILD SUCCESS, 4 tests passing

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add runtime/src/main/java/io/casehub/eidos/runtime/cache/ runtime/src/test/java/io/casehub/eidos/runtime/cache/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#19): add BlockingToReactiveRenderedPromptCacheAdapter

@DefaultBean wrapping RenderedPromptCache. No runSubscriptionOn —
callers are already on worker pool. Errors swallowed per contract.

Refs #19"
```

---

## Task 3: Pipeline refactor — `StageOneResult` + `EidosRenderPipeline`

**Files:**
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/renderer/StageOneResult.java`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosRenderPipeline.java`
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/renderer/EidosRenderPipelineTest.java`

- [ ] **Step 1: Create `StageOneResult` package-private record**

```java
package io.casehub.eidos.runtime.renderer;

import com.fasterxml.jackson.databind.node.ObjectNode;

record StageOneResult(
        ObjectNode descriptorNode,
        ObjectNode contextNode,
        String descriptorHash,
        String contextHash,
        String cacheKey
) {}
```

- [ ] **Step 2: Write new `buildStage1()` test in `EidosRenderPipelineTest`**

Add these tests (they will fail to compile until the method exists):

```java
@Test
void buildStage1_returns_matching_hashes_and_key() {
    final var desc = EidosRenderPipelineTest.fullDescriptor();
    final var ctx  = EidosRenderPipelineTest.fullContext();
    final StageOneResult s1 = pipeline.buildStage1(desc, ctx);
    assertThat(s1.descriptorHash()).hasSize(16);
    assertThat(s1.contextHash()).hasSize(16);
    assertThat(s1.cacheKey()).contains(s1.descriptorHash());
    assertThat(s1.cacheKey()).contains(s1.contextHash());
    assertThat(s1.cacheKey()).contains("CLAUDE_MD");
}

@Test
void buildStage1_is_deterministic() {
    final var desc = EidosRenderPipelineTest.fullDescriptor();
    final var ctx  = EidosRenderPipelineTest.fullContext();
    assertThat(pipeline.buildStage1(desc, ctx).cacheKey())
        .isEqualTo(pipeline.buildStage1(desc, ctx).cacheKey());
}
```

- [ ] **Step 3: Refactor `EidosRenderPipeline`**

Changes to make in `EidosRenderPipeline.java`:

**3a. Remove `RenderedPromptCache` field and injection.** Delete the `cache` field and the `RenderedPromptCache cache` constructor parameter. Update the constructor to:

```java
@Inject
EidosRenderPipeline(final VocabularyRegistry vocab,
                    final ObjectMapper mapper) {
    this.vocab = vocab;
    this.mapper = mapper;
}
```

**3b. Add `buildStage1()` method** (replacing the 5-line block duplicated in both renderers):

```java
StageOneResult buildStage1(final AgentDescriptor descriptor, final AgentPromptContext context) {
    final ObjectNode descriptorNode = buildDescriptorPayload(descriptor);
    final ObjectNode contextNode    = buildContextPayload(context);
    final String descriptorHash     = fingerprint(descriptorNode.toString());
    final String contextHash        = fingerprint(contextNode.toString());
    final String key                = cacheKey(descriptorHash, contextHash, context.format());
    return new StageOneResult(descriptorNode, contextNode, descriptorHash, contextHash, key);
}
```

**3c. Change `assemble()` signature** — accept `StageOneResult`, return `RenderedPrompt`:

```java
RenderedPrompt assemble(final StageOneResult s1,
                         final Optional<SemanticEnrichment> enrichment,
                         final Optional<A2AEnrichment> a2aEnrichment,
                         final AgentDescriptor descriptor,
                         final AgentPromptContext context) {
    final String content = switch (context.format()) {
        case CLAUDE_MD     -> assembleClaudeMarkdown(enrichment, descriptor, context);
        case OPENAI_SYSTEM -> assembleOpenAiSystem(enrichment, descriptor, context);
        case A2A_CARD      -> assembleA2aCard(a2aEnrichment, descriptor);
        case GEMINI        -> assembleGemini(enrichment, descriptor, context);
    };
    return new RenderedPrompt(content, context.format(), s1.descriptorHash(), s1.contextHash());
}
```

**3d. Delete `cacheGet()` and `assembleAndCache()` methods entirely.**

- [ ] **Step 4: Fix `EidosRenderPipelineTest` constructor**

Change `setUp()`:
```java
@BeforeEach
void setUp() {
    pipeline = new EidosRenderPipeline(new CdiVocabularyRegistry(), MAPPER);
}
```

The existing tests in this file don't call `assemble()` or `cacheGet()`, so only the constructor line needs updating.

- [ ] **Step 5: Run pipeline tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=EidosRenderPipelineTest -q
```

Expected: BUILD SUCCESS, all pipeline tests pass including the two new `buildStage1` tests

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add \
  runtime/src/main/java/io/casehub/eidos/runtime/renderer/StageOneResult.java \
  runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosRenderPipeline.java \
  runtime/src/test/java/io/casehub/eidos/runtime/renderer/EidosRenderPipelineTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "refactor(eidos#19): extract StageOneResult; pipeline becomes stateless

Remove RenderedPromptCache from EidosRenderPipeline. Add buildStage1()
eliminating 5-line duplication across both renderers. assemble() now
returns RenderedPrompt directly via StageOneResult.

Refs #19"
```

---

## Task 4: `TestReactiveRenderedPromptCache` test double

**Files:**
- Create: `runtime/src/test/java/io/casehub/eidos/runtime/renderer/TestReactiveRenderedPromptCache.java`

- [ ] **Step 1: Create the test double**

```java
package io.casehub.eidos.runtime.renderer;

import io.casehub.eidos.api.ReactiveRenderedPromptCache;
import io.casehub.eidos.api.SystemPromptRenderer.RenderedPrompt;
import io.smallrye.mutiny.Uni;

import java.util.HashMap;
import java.util.Map;
import java.util.Optional;

class TestReactiveRenderedPromptCache implements ReactiveRenderedPromptCache {
    final Map<String, RenderedPrompt> store = new HashMap<>();
    int putCount = 0;
    int getCount = 0;

    @Override
    public Uni<Optional<RenderedPrompt>> get(final String cacheKey) {
        getCount++;
        return Uni.createFrom().item(Optional.ofNullable(store.get(cacheKey)));
    }

    @Override
    public Uni<Void> put(final String cacheKey, final RenderedPrompt result) {
        putCount++;
        store.put(cacheKey, result);
        return Uni.createFrom().voidItem();
    }
}
```

- [ ] **Step 2: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add runtime/src/test/java/io/casehub/eidos/runtime/renderer/TestReactiveRenderedPromptCache.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "test(eidos#19): add TestReactiveRenderedPromptCache test double

Refs #19"
```

---

## Task 5: Update `EidosSystemPromptRenderer` + fix its tests

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosSystemPromptRenderer.java`
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/renderer/EidosSystemPromptRendererTest.java`

- [ ] **Step 1: Rewrite `EidosSystemPromptRenderer`**

Replace the entire file:

```java
package io.casehub.eidos.runtime.renderer;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import dev.langchain4j.model.chat.ChatModel;
import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.eidos.api.AgentPromptContext;
import io.casehub.eidos.api.ReactiveRenderedPromptCache;
import io.casehub.eidos.api.SystemPromptRenderer;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Any;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;

import java.time.Duration;
import java.util.Optional;

@DefaultBean
@ApplicationScoped
public class EidosSystemPromptRenderer implements SystemPromptRenderer {

    private final ChatModel llm;
    private final EidosRenderPipeline pipeline;
    private final SemanticEnrichmentStep enrichmentStep;
    private final A2ASemanticEnrichmentStep a2aEnrichmentStep;
    private final ReactiveRenderedPromptCache cache;

    @Inject
    public EidosSystemPromptRenderer(
            @Any final Instance<ChatModel> llm,
            final EidosRenderPipeline pipeline,
            final ReactiveRenderedPromptCache cache,
            final ObjectMapper mapper) {
        this.llm = llm.isResolvable() ? llm.get() : null;
        this.pipeline = pipeline;
        this.cache = cache;
        this.enrichmentStep = new SemanticEnrichmentStep(mapper);
        this.a2aEnrichmentStep = new A2ASemanticEnrichmentStep(mapper);
    }

    /** Package-private constructor for pure-Java tests — no CDI required. */
    EidosSystemPromptRenderer(final ChatModel llm,
                              final EidosRenderPipeline pipeline,
                              final ReactiveRenderedPromptCache cache,
                              final ObjectMapper mapper) {
        this.llm = llm;
        this.pipeline = pipeline;
        this.cache = cache;
        this.enrichmentStep = new SemanticEnrichmentStep(mapper);
        this.a2aEnrichmentStep = new A2ASemanticEnrichmentStep(mapper);
    }

    @Override
    public RenderedPrompt render(final AgentDescriptor descriptor, final AgentPromptContext context) {
        final StageOneResult s1 = pipeline.buildStage1(descriptor, context);

        // await() resolves synchronously for the default adapter (no runSubscriptionOn).
        // try-catch defends against future async implementations or adapter contract violations.
        final Optional<RenderedPrompt> cached;
        try {
            cached = cache.get(s1.cacheKey()).await().atMost(Duration.ofSeconds(5));
        } catch (final Exception e) {
            return renderFresh(s1, descriptor, context);
        }
        if (cached.isPresent()) return cached.get();
        return renderFresh(s1, descriptor, context);
    }

    private RenderedPrompt renderFresh(final StageOneResult s1,
                                        final AgentDescriptor descriptor,
                                        final AgentPromptContext context) {
        Optional<SemanticEnrichment> enrichment = Optional.empty();
        if (llm != null && EidosRenderPipeline.usesEnrichment(context.format())) {
            final ObjectNode llmPayload = pipeline.buildLlmPayload(s1.descriptorNode(), s1.contextNode());
            enrichment = enrichmentStep.enrich(llm, llmPayload);
        }

        Optional<A2AEnrichment> a2aEnrichment = Optional.empty();
        if (context.format() == RenderFormat.A2A_CARD && llm != null) {
            a2aEnrichment = a2aEnrichmentStep.enrich(llm, s1.descriptorNode());
        }

        final RenderedPrompt result = pipeline.assemble(s1, enrichment, a2aEnrichment, descriptor, context);
        cache.put(s1.cacheKey(), result).await().indefinitely(); // synchronous for default adapter
        return result;
    }
}
```

- [ ] **Step 2: Fix `EidosSystemPromptRendererTest`**

The test currently wires cache through the pipeline. After the refactor, cache goes directly to the renderer. Update `setUp()` and helpers:

```java
TestReactiveRenderedPromptCache testCache;
EidosRenderPipeline pipeline;

@BeforeEach
void setUp() {
    mockLlm = new ChatModel() {
        @Override
        public ChatResponse doChat(final ChatRequest request) {
            return ChatResponse.builder().aiMessage(AiMessage.from(LLM_JSON_RESPONSE)).build();
        }
    };
    testCache = new TestReactiveRenderedPromptCache();
    final var vocab = new CdiVocabularyRegistry();
    pipeline = new EidosRenderPipeline(vocab, MAPPER);
    rendererWithLlm   = new EidosSystemPromptRenderer(mockLlm, pipeline, testCache, MAPPER);
    rendererStructural = new EidosSystemPromptRenderer((ChatModel) null, pipeline,
                             new TestReactiveRenderedPromptCache(), MAPPER);
}
```

Update `rendererWithA2aLlm()`:
```java
static EidosSystemPromptRenderer rendererWithA2aLlm() {
    final ChatModel a2aLlm = new ChatModel() {
        @Override
        public ChatResponse doChat(final ChatRequest request) {
            return ChatResponse.builder().aiMessage(AiMessage.from(A2A_LLM_JSON_RESPONSE)).build();
        }
    };
    return new EidosSystemPromptRenderer(a2aLlm,
            new EidosRenderPipeline(new CdiVocabularyRegistry(), MAPPER),
            new TestReactiveRenderedPromptCache(), MAPPER);
}
```

Update `capturePayload()`:
```java
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
    new EidosSystemPromptRenderer(capturingLlm,
            new EidosRenderPipeline(new CdiVocabularyRegistry(), MAPPER),
            new TestReactiveRenderedPromptCache(), MAPPER).render(desc, ctx);
    return captured[0];
}
```

Find any remaining cache-hit tests in `EidosSystemPromptRendererTest` that test `testCache.getCount` / `testCache.putCount` and verify they still reference `testCache` (now a `TestReactiveRenderedPromptCache`). The field type changes from `TestRenderedPromptCache` to `TestReactiveRenderedPromptCache`.

- [ ] **Step 3: Run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=EidosSystemPromptRendererTest -q
```

Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add \
  runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosSystemPromptRenderer.java \
  runtime/src/test/java/io/casehub/eidos/runtime/renderer/EidosSystemPromptRendererTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "refactor(eidos#19): EidosSystemPromptRenderer injects ReactiveRenderedPromptCache

Cache moves from pipeline to renderer. Uses buildStage1() shared method.
Cache check wrapped in try-catch for defensive timeout handling.

Refs #19"
```

---

## Task 6: Update `DefaultReactiveSystemPromptRenderer` + fix its tests

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/renderer/DefaultReactiveSystemPromptRenderer.java`
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/renderer/DefaultReactiveSystemPromptRendererStreamingTest.java`

- [ ] **Step 1: Rewrite `DefaultReactiveSystemPromptRenderer`**

```java
package io.casehub.eidos.runtime.renderer;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import dev.langchain4j.model.chat.StreamingChatModel;
import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.eidos.api.AgentPromptContext;
import io.casehub.eidos.api.ReactiveRenderedPromptCache;
import io.casehub.eidos.api.ReactiveSystemPromptRenderer;
import io.casehub.eidos.api.SystemPromptRenderer;
import io.casehub.eidos.api.SystemPromptRenderer.RenderFormat;
import io.casehub.eidos.api.SystemPromptRenderer.RenderedPrompt;
import io.quarkus.arc.DefaultBean;
import io.smallrye.mutiny.Uni;
import io.smallrye.mutiny.infrastructure.Infrastructure;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Any;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;

import java.util.Optional;

@DefaultBean
@ApplicationScoped
public class DefaultReactiveSystemPromptRenderer implements ReactiveSystemPromptRenderer {

    private final StreamingChatModel streamingLlm;
    private final SystemPromptRenderer blockingDelegate;
    private final EidosRenderPipeline pipeline;
    private final ReactiveRenderedPromptCache cache;
    private final ReactiveSemanticEnrichmentStep reactiveEnrichStep;
    private final ReactiveA2ASemanticEnrichmentStep reactiveA2aStep;

    @Inject
    public DefaultReactiveSystemPromptRenderer(
            @Any final Instance<StreamingChatModel> streamingLlmInstance,
            final SystemPromptRenderer blockingDelegate,
            final EidosRenderPipeline pipeline,
            final ReactiveRenderedPromptCache cache,
            final ObjectMapper mapper) {
        this.streamingLlm = streamingLlmInstance.isResolvable() ? streamingLlmInstance.get() : null;
        this.blockingDelegate = blockingDelegate;
        this.pipeline = pipeline;
        this.cache = cache;
        this.reactiveEnrichStep = new ReactiveSemanticEnrichmentStep(mapper);
        this.reactiveA2aStep = new ReactiveA2ASemanticEnrichmentStep(mapper);
    }

    /** Package-private constructor for tests. */
    DefaultReactiveSystemPromptRenderer(
            final StreamingChatModel streamingLlm,
            final SystemPromptRenderer blockingDelegate,
            final EidosRenderPipeline pipeline,
            final ReactiveRenderedPromptCache cache,
            final ObjectMapper mapper) {
        this.streamingLlm = streamingLlm;
        this.blockingDelegate = blockingDelegate;
        this.pipeline = pipeline;
        this.cache = cache;
        this.reactiveEnrichStep = new ReactiveSemanticEnrichmentStep(mapper);
        this.reactiveA2aStep = new ReactiveA2ASemanticEnrichmentStep(mapper);
    }

    @Override
    public Uni<RenderedPrompt> render(final AgentDescriptor descriptor,
                                      final AgentPromptContext context) {
        if (streamingLlm == null) {
            return Uni.createFrom()
                      .item(() -> blockingDelegate.render(descriptor, context))
                      .runSubscriptionOn(Infrastructure.getDefaultWorkerPool());
        }

        // Stage 1 on worker pool — payload building + cache check
        return Uni.createFrom()
                  .item(() -> pipeline.buildStage1(descriptor, context))
                  .runSubscriptionOn(Infrastructure.getDefaultWorkerPool())
                  .chain(s1 -> cache.get(s1.cacheKey())
                                    .chain(hit -> hit.isPresent()
                                        ? Uni.createFrom().item(hit.get())
                                        : executeStagesTwoAndThree(s1, descriptor, context)));
    }

    private Uni<RenderedPrompt> executeStagesTwoAndThree(
            final StageOneResult s1,
            final AgentDescriptor descriptor,
            final AgentPromptContext context) {
        final RenderFormat format = context.format();
        final boolean needsEnrichment = EidosRenderPipeline.usesEnrichment(format);
        final boolean needsA2A = format == RenderFormat.A2A_CARD;

        final ObjectNode llmPayload = needsEnrichment
                ? pipeline.buildLlmPayload(s1.descriptorNode(), s1.contextNode())
                : null;

        final Uni<Optional<SemanticEnrichment>> enrichUni = needsEnrichment
                ? reactiveEnrichStep.enrich(streamingLlm, llmPayload)
                : Uni.createFrom().item(Optional.empty());
        final Uni<Optional<A2AEnrichment>> a2aUni = needsA2A
                ? reactiveA2aStep.enrich(streamingLlm, s1.descriptorNode())
                : Uni.createFrom().item(Optional.empty());

        // Stage 3: assemble on worker pool, then reactive cache put
        return Uni.combine().all().unis(enrichUni, a2aUni).asTuple()
                  .emitOn(Infrastructure.getDefaultWorkerPool())
                  .chain(t -> {
                      final RenderedPrompt result = pipeline.assemble(
                          s1, t.getItem1(), t.getItem2(), descriptor, context);
                      return cache.put(s1.cacheKey(), result).replaceWith(result);
                  });
    }
}
```

- [ ] **Step 2: Fix `DefaultReactiveSystemPromptRendererStreamingTest`**

Update `setUp()`:
```java
@BeforeEach
void setUp() {
    pipeline = new EidosRenderPipeline(new CdiVocabularyRegistry(), MAPPER);
    blockingDelegate = (descriptor, context) ->
        new RenderedPrompt("blocking:" + descriptor.name(), context.format(), "dh", "ch");
}
```

Update all renderer constructions to add the cache parameter. For tests that don't need caching, pass `new TestReactiveRenderedPromptCache()`:

```java
// renders_with_streaming_llm_when_present
final var renderer = new DefaultReactiveSystemPromptRenderer(
        successMock(), blockingDelegate, pipeline, new TestReactiveRenderedPromptCache(), MAPPER);

// uses_streaming_api_not_blocking_overload
final var renderer = new DefaultReactiveSystemPromptRenderer(
        trackingMock, blockingDelegate, pipeline, new TestReactiveRenderedPromptCache(), MAPPER);

// falls_back_to_structural_when_streaming_llm_on_error
final var renderer = new DefaultReactiveSystemPromptRenderer(
        errorMock(), blockingDelegate, pipeline, new TestReactiveRenderedPromptCache(), MAPPER);

// falls_back_to_blocking_delegate_when_streaming_llm_absent
final var renderer = new DefaultReactiveSystemPromptRenderer(
        (StreamingChatModel) null, blockingDelegate, pipeline, new TestReactiveRenderedPromptCache(), MAPPER);
```

Rewrite `cache_hit_returns_without_any_llm_call()`:

```java
@Test
void cache_hit_returns_without_any_llm_call() {
    // Pre-populate the cache with the key for our descriptor+context combo.
    // We need the key, so call buildStage1() to get it.
    final var ctx = AgentPromptContext.forFormat(CLAUDE_MD);
    final StageOneResult s1 = pipeline.buildStage1(descriptor(), ctx);
    final RenderedPrompt cachedResult = new RenderedPrompt(
        "cached-content", CLAUDE_MD, s1.descriptorHash(), s1.contextHash());

    final TestReactiveRenderedPromptCache prePopulated = new TestReactiveRenderedPromptCache();
    prePopulated.store.put(s1.cacheKey(), cachedResult);

    // Renderer with a throwing LLM — must not be called on cache hit
    final StreamingChatModel throwingMock = new StreamingChatModel() {
        @Override
        public void doChat(ChatRequest request, StreamingChatResponseHandler handler) {
            throw new AssertionError("LLM must not be called on cache hit");
        }
    };
    final var renderer = new DefaultReactiveSystemPromptRenderer(
            throwingMock, blockingDelegate, pipeline, prePopulated, MAPPER);

    final RenderedPrompt result = renderer.render(descriptor(), ctx).await().indefinitely();

    assertThat(result.content()).isEqualTo("cached-content");
    assertThat(prePopulated.getCount).isEqualTo(1);
    assertThat(prePopulated.putCount).isEqualTo(0); // cache hit — no put
}
```

- [ ] **Step 3: Run all renderer tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="DefaultReactiveSystemPromptRendererStreamingTest,EidosSystemPromptRendererTest,EidosRenderPipelineTest" -q
```

Expected: BUILD SUCCESS

- [ ] **Step 4: Run full module**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q
```

Expected: BUILD SUCCESS, all tests pass

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add \
  runtime/src/main/java/io/casehub/eidos/runtime/renderer/DefaultReactiveSystemPromptRenderer.java \
  runtime/src/test/java/io/casehub/eidos/runtime/renderer/DefaultReactiveSystemPromptRendererStreamingTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "refactor(eidos#19): DefaultReactiveSystemPromptRenderer injects ReactiveRenderedPromptCache

Stage 1 now calls pipeline.buildStage1(). Cache get/put via reactive SPI.
Cache-hit test restructured to pre-populate TestReactiveRenderedPromptCache.

Refs #19"
```

---

## Task 7: Run full build — verify eidos#19 complete

- [ ] **Step 1: Full build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -q
```

Expected: BUILD SUCCESS, all tests green

- [ ] **Step 2: Commit parity note**

Add `ReactiveRenderedPromptCache` to `BlockingReactiveParityTest` to enforce the `Uni<>` return type convention. Open `BlockingReactiveParityTest.java` and add:

```java
import io.casehub.eidos.api.ReactiveRenderedPromptCache;

@Test
void reactive_cache_methods_return_uni() {
    ArchRule rule = methods()
        .that().areDeclaredIn(ReactiveRenderedPromptCache.class)
        .should().haveRawReturnType(assignableTo(Uni.class));
    rule.check(API_CLASSES);
}
```

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=BlockingReactiveParityTest -q
git -C /Users/mdproctor/claude/casehub/eidos add runtime/src/test/java/io/casehub/eidos/runtime/registry/BlockingReactiveParityTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "test(eidos#19): enforce Uni return type on ReactiveRenderedPromptCache

Refs #19"
```

---

## Task 8: `AgentDescriptorValidationException` + `AgentDescriptorValidator`

**Files:**
- Create: `api/src/main/java/io/casehub/eidos/api/AgentDescriptorValidationException.java`
- Create: `api/src/main/java/io/casehub/eidos/api/AgentDescriptorValidator.java`
- Create: `api/src/test/java/io/casehub/eidos/api/AgentDescriptorValidatorTest.java`

- [ ] **Step 1: Write failing tests**

```java
package io.casehub.eidos.api;

import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.Arguments;
import org.junit.jupiter.params.provider.MethodSource;

import java.util.stream.Stream;

import static org.assertj.core.api.Assertions.assertThatNoException;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class AgentDescriptorValidatorTest {

    static final String VALID_ID    = "agent-1";
    static final String VALID_NAME  = "My Agent";
    static final String VALID_SLOT  = "reviewer";
    static final String VALID_TID   = "tenant-1";

    @ParameterizedTest(name = "{0}")
    @MethodSource("invalidCases")
    void invalid_field_throws_with_field_name(String label, String agentId, String name,
                                               String slot, String tenancyId,
                                               String expectedField) {
        assertThatThrownBy(() -> AgentDescriptorValidator.validate(agentId, name, slot, tenancyId))
            .isInstanceOf(AgentDescriptorValidationException.class)
            .satisfies(ex -> {
                AgentDescriptorValidationException e = (AgentDescriptorValidationException) ex;
                assertThat(e.fieldName()).isEqualTo(expectedField);
            });
    }

    static Stream<Arguments> invalidCases() {
        return Stream.of(
            // null checks
            Arguments.of("agentId null",    null,       VALID_NAME, VALID_SLOT, VALID_TID,  "agentId"),
            Arguments.of("name null",       VALID_ID,   null,       VALID_SLOT, VALID_TID,  "name"),
            Arguments.of("slot null",       VALID_ID,   VALID_NAME, null,       VALID_TID,  "slot"),
            Arguments.of("tenancyId null",  VALID_ID,   VALID_NAME, VALID_SLOT, null,       "tenancyId"),
            // blank checks
            Arguments.of("agentId blank",   "   ",      VALID_NAME, VALID_SLOT, VALID_TID,  "agentId"),
            Arguments.of("name blank",      VALID_ID,   "",         VALID_SLOT, VALID_TID,  "name"),
            // length checks
            Arguments.of("agentId too long",  "a".repeat(256), VALID_NAME,    VALID_SLOT,  VALID_TID, "agentId"),
            Arguments.of("name too long",     VALID_ID,        "n".repeat(201), VALID_SLOT, VALID_TID, "name"),
            Arguments.of("slot too long",     VALID_ID,        VALID_NAME,    "s".repeat(101), VALID_TID, "slot"),
            Arguments.of("tenancyId too long",VALID_ID,        VALID_NAME,    VALID_SLOT,  "t".repeat(256), "tenancyId"),
            // C0 control chars
            Arguments.of("name C0 tab",     VALID_ID, "name\twith\ttab", VALID_SLOT, VALID_TID, "name"),
            Arguments.of("slot newline",    VALID_ID, VALID_NAME, "slot\nwith\nnl", VALID_TID, "slot"),
            // C1 control chars
            Arguments.of("name C1 U+0085", VALID_ID, "name", VALID_SLOT, VALID_TID, "name"),
            Arguments.of("name DEL",        VALID_ID, "name", VALID_SLOT, VALID_TID, "name"),
            // BiDi overrides
            Arguments.of("name RLM",       VALID_ID, "name‏", VALID_SLOT, VALID_TID, "name"),
            Arguments.of("slot LRE",       VALID_ID, VALID_NAME,   "slot‪", VALID_TID, "slot"),
            Arguments.of("name LRI",       VALID_ID, "name⁦", VALID_SLOT, VALID_TID, "name"),
            // Zero-width chars
            Arguments.of("name ZWSP",      VALID_ID, "name​", VALID_SLOT, VALID_TID, "name"),
            Arguments.of("name BOM",       VALID_ID, "name﻿", VALID_SLOT, VALID_TID, "name"),
            // Line/paragraph separators
            Arguments.of("name LS U+2028", VALID_ID, "name ", VALID_SLOT, VALID_TID, "name"),
            Arguments.of("name PS U+2029", VALID_ID, "name ", VALID_SLOT, VALID_TID, "name")
        );
    }

    @ParameterizedTest(name = "{0}")
    @MethodSource("validCases")
    void valid_fields_do_not_throw(String label, String agentId, String name,
                                    String slot, String tenancyId) {
        assertThatNoException().isThrownBy(
            () -> AgentDescriptorValidator.validate(agentId, name, slot, tenancyId));
    }

    static Stream<Arguments> validCases() {
        return Stream.of(
            Arguments.of("all simple",    VALID_ID, VALID_NAME, VALID_SLOT, VALID_TID),
            Arguments.of("max agentId",   "a".repeat(255), VALID_NAME, VALID_SLOT, VALID_TID),
            Arguments.of("max name",      VALID_ID, "n".repeat(200), VALID_SLOT, VALID_TID),
            Arguments.of("max slot",      VALID_ID, VALID_NAME, "s".repeat(100), VALID_TID),
            Arguments.of("unicode ok",    VALID_ID, "Agent 中文", VALID_SLOT, VALID_TID),
            Arguments.of("hyphen-dash",   "my-agent-v2", VALID_NAME, VALID_SLOT, VALID_TID)
        );
    }
}
```

Also add the import for `assertThat` inside the satisfies lambda:
```java
import static org.assertj.core.api.Assertions.assertThat;
```

- [ ] **Step 2: Run — verify FAIL (classes don't exist yet)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=AgentDescriptorValidatorTest -q 2>&1 | tail -3
```

Expected: compilation error

- [ ] **Step 3: Create `AgentDescriptorValidationException`**

```java
package io.casehub.eidos.api;

public final class AgentDescriptorValidationException extends IllegalArgumentException {

    private final String fieldName;

    public AgentDescriptorValidationException(final String fieldName, final String message) {
        super("AgentDescriptor field '" + fieldName + "': " + message);
        this.fieldName = fieldName;
    }

    public String fieldName() {
        return fieldName;
    }
}
```

- [ ] **Step 4: Create `AgentDescriptorValidator`**

```java
package io.casehub.eidos.api;

class AgentDescriptorValidator {

    // Bounds chosen to cap cache key length and LLM payload size.
    private static final int MAX_AGENT_ID   = 255;
    private static final int MAX_NAME       = 200;
    private static final int MAX_SLOT       = 100;
    private static final int MAX_TENANCY_ID = 255;

    static void validate(final String agentId, final String name,
                          final String slot, final String tenancyId) {
        validateField("agentId",   agentId,   MAX_AGENT_ID);
        validateField("name",      name,      MAX_NAME);
        validateField("slot",      slot,      MAX_SLOT);
        validateField("tenancyId", tenancyId, MAX_TENANCY_ID);
    }

    private static void validateField(final String fieldName, final String value,
                                       final int maxLength) {
        if (value == null) {
            throw new AgentDescriptorValidationException(fieldName, "must not be null");
        }
        if (value.isBlank()) {
            throw new AgentDescriptorValidationException(fieldName, "must not be blank");
        }
        if (value.length() > maxLength) {
            throw new AgentDescriptorValidationException(fieldName,
                "exceeds maximum length " + maxLength + " (was " + value.length() + ")");
        }
        for (int i = 0; i < value.length(); ) {
            final int cp = value.codePointAt(i);
            if (isBanned(cp)) {
                throw new AgentDescriptorValidationException(fieldName,
                    "contains banned character U+" + String.format("%04X", cp));
            }
            i += Character.charCount(cp);
        }
    }

    private static boolean isBanned(final int cp) {
        if (cp <= 0x001F) return true;                      // C0 control chars
        if (cp >= 0x007F && cp <= 0x009F) return true;     // DEL + C1 control chars
        if (cp == 0x200E || cp == 0x200F) return true;     // LRM, RLM
        if (cp >= 0x202A && cp <= 0x202E) return true;     // LRE, RLE, PDF, LRO, RLO
        if (cp >= 0x2066 && cp <= 0x2069) return true;     // LRI, RLI, FSI, PDI
        if (cp == 0x200B || cp == 0xFEFF) return true;     // ZWSP, BOM/ZWNBSP
        if (cp == 0x200C || cp == 0x200D) return true;     // ZWNJ, ZWJ
        if (cp == 0x2028 || cp == 0x2029) return true;     // LINE SEP, PARA SEP
        return false;
    }
}
```

- [ ] **Step 5: Run tests — verify PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=AgentDescriptorValidatorTest -q
```

Expected: BUILD SUCCESS, all cases pass

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add \
  api/src/main/java/io/casehub/eidos/api/AgentDescriptorValidationException.java \
  api/src/main/java/io/casehub/eidos/api/AgentDescriptorValidator.java \
  api/src/test/java/io/casehub/eidos/api/AgentDescriptorValidatorTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#15): add AgentDescriptorValidator and AgentDescriptorValidationException

Character-set + length validation for the 4 implicitly-required fields.
Rejects C0/C1, BiDi overrides, zero-width chars, U+2028/U+2029.

Refs #15"
```

---

## Task 9: `AgentDescriptor` compact constructor + update existing tests

**Files:**
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentDescriptor.java`
- Verify: existing `AgentDescriptorTest` + all other test files that construct `AgentDescriptor`

- [ ] **Step 1: Add compact constructor to `AgentDescriptor`**

Open `AgentDescriptor.java` and add the compact constructor (the record already has its field declarations):

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
    }
}
```

- [ ] **Step 2: Build api to surface all broken callsites**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -pl api 2>&1 | grep "error:" | head -20
```

Expected: compile errors in test files that pass null/blank for required fields. Identify each one.

- [ ] **Step 3: Fix broken test callsites in `api/`**

Any `AgentDescriptor` construction in `api/src/test/` that passes `null` for `agentId`, `name`, `slot`, or `tenancyId` must be updated to use valid values, OR the test that was testing null-resistance moves to `AgentDescriptorValidatorTest` (where it belongs now).

Example fix pattern — change:
```java
// Before: testing null agentId
new AgentDescriptor(null, "Name", ..., "slot", ..., "tenant")
```
to a validator test (already covered in `AgentDescriptorValidatorTest`) and remove the null-construction from this file.

For tests that simply need a descriptor and were passing `null` for convenience:
```java
// Before: convenience null for optional fields — agentId must be valid
new AgentDescriptor("id", "Name", null, null, null, null, null, null, null, null, "slot", List.of(), null, null, null, "tenant")
```
This is already valid (nulls are for optional fields, not required ones).

- [ ] **Step 4: Build and test the full stack**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -q
```

Expected: BUILD SUCCESS — all modules build, all tests pass

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add api/src/main/java/io/casehub/eidos/api/AgentDescriptor.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#15): AgentDescriptor compact constructor — validates required fields

Every AgentDescriptor in the system is guaranteed valid at construction.
Delegates to AgentDescriptorValidator.validate(agentId, name, slot, tenancyId).

Closes #15"
```

---

## Task 10: `eval/` module — pom + root pom update

**Files:**
- Create: `eval/pom.xml`
- Modify: `pom.xml` (root)

- [ ] **Step 1: Add `eval` module to root `pom.xml`**

In `pom.xml`, find the `<modules>` block (currently ends with `<module>examples</module>`) and add:

```xml
<module>eval</module>
```

After `<module>examples</module>`.

- [ ] **Step 2: Create `eval/pom.xml`**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-eidos-parent</artifactId>
        <version>1.0.0-SNAPSHOT</version>
        <relativePath>../pom.xml</relativePath>
    </parent>

    <artifactId>casehub-eidos-eval</artifactId>
    <name>CaseHub Eidos — Offline Quality Evaluation Harness</name>

    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-eidos</artifactId>
            <version>${project.version}</version>
        </dependency>

        <!-- LangChain4j for judge ChatModel -->
        <dependency>
            <groupId>dev.langchain4j</groupId>
            <artifactId>langchain4j-core</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkiverse.langchain4j</groupId>
            <artifactId>quarkus-langchain4j-core</artifactId>
        </dependency>

        <!-- Test -->
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-junit5</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <!-- Eval harness must never be deployed as a library artifact -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-deploy-plugin</artifactId>
                <configuration>
                    <skip>true</skip>
                </configuration>
            </plugin>
            <!-- CI must never run @Tag("eval") tests — they require a real LLM -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <configuration>
                    <excludedGroups>eval</excludedGroups>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

- [ ] **Step 3: Verify module structure builds**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl eval -q 2>&1 | tail -5
```

Expected: BUILD SUCCESS (empty module compiles)

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add pom.xml eval/pom.xml
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#16): add eval/ Maven module — offline quality evaluation harness

Never deployed (maven-deploy-plugin skip=true).
CI excludes @Tag(\"eval\") tests via Surefire excludedGroups.

Refs #16"
```

---

## Task 11: Eval core types

**Files:**
- Create: `eval/src/main/java/io/casehub/eidos/eval/EvalCase.java`
- Create: `eval/src/main/java/io/casehub/eidos/eval/EvalDimension.java`
- Create: `eval/src/main/java/io/casehub/eidos/eval/EvalScore.java`
- Create: `eval/src/main/java/io/casehub/eidos/eval/EvalResult.java`
- Create: `eval/src/main/java/io/casehub/eidos/eval/EvalSummary.java`
- Create: `eval/src/main/java/io/casehub/eidos/eval/EvalReport.java`

- [ ] **Step 1: Create all core types**

`EvalCase.java`:
```java
package io.casehub.eidos.eval;

import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.eidos.api.AgentPromptContext;

public record EvalCase(String name, AgentDescriptor descriptor, AgentPromptContext context) {}
```

`EvalDimension.java`:
```java
package io.casehub.eidos.eval;

public enum EvalDimension {
    SECOND_PERSON,    // "you"/"your" throughout; no third-person ("the agent", "it")
    CONCISENESS,      // no redundancy, no filler, dense information per sentence
    FACTUAL_FIDELITY, // nothing claimed that is absent from descriptor + context
    TONE              // reads as instructions to an AI agent, not documentation about one
}
```

`EvalScore.java`:
```java
package io.casehub.eidos.eval;

public record EvalScore(int score, String reasoning) {  // score 0–5
    public EvalScore {
        if (score < 0 || score > 5) throw new IllegalArgumentException("score must be 0–5, was " + score);
    }
}
```

`EvalResult.java`:
```java
package io.casehub.eidos.eval;

import io.casehub.eidos.api.SystemPromptRenderer.RenderedPrompt;
import java.util.List;
import java.util.Map;

public record EvalResult(
        EvalCase evalCase,
        RenderedPrompt rendered,
        boolean completenessPass,
        List<String> missingCapabilities,
        Map<EvalDimension, EvalScore> scores,
        double overall,      // 0.0–5.0; mean of EvalDimension scores (COMPLETENESS excluded)
        List<String> issues
) {}
```

`EvalSummary.java`:
```java
package io.casehub.eidos.eval;

import java.util.Map;

public record EvalSummary(
        boolean allCasesComplete,
        Map<EvalDimension, Double> meanByDimension,
        EvalDimension lowestScoringDimension, // ties broken by EvalDimension declaration order
        double meanOverall
) {}
```

`EvalReport.java`:
```java
package io.casehub.eidos.eval;

import java.time.Instant;
import java.util.EnumMap;
import java.util.List;
import java.util.Map;

public record EvalReport(
        Instant timestamp,
        String judgeModel,
        List<EvalResult> results,
        EvalSummary summary
) {
    public static EvalReport build(final List<EvalResult> results, final String judgeModel) {
        final boolean allComplete = results.stream().allMatch(EvalResult::completenessPass);

        final Map<EvalDimension, Double> meanByDim = new EnumMap<>(EvalDimension.class);
        for (final EvalDimension d : EvalDimension.values()) {
            final double mean = results.stream()
                .mapToInt(r -> r.scores().get(d).score())
                .average()
                .orElse(0.0);
            meanByDim.put(d, mean);
        }

        // Lowest scoring — ties broken by EvalDimension declaration order (stream preserves it)
        final EvalDimension lowest = meanByDim.entrySet().stream()
            .min(Map.Entry.comparingByValue())
            .map(Map.Entry::getKey)
            .orElse(EvalDimension.values()[0]);

        final double meanOverall = results.stream()
            .mapToDouble(EvalResult::overall)
            .average()
            .orElse(0.0);

        final EvalSummary summary = new EvalSummary(allComplete, meanByDim, lowest, meanOverall);
        return new EvalReport(Instant.now(), judgeModel, results, summary);
    }
}
```

- [ ] **Step 2: Compile to verify**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl eval -q
```

Expected: BUILD SUCCESS

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add eval/src/main/java/io/casehub/eidos/eval/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#16): add eval core types — EvalCase, EvalDimension, EvalScore, EvalResult, EvalSummary, EvalReport

Refs #16"
```

---

## Task 12: `EvalDataset` + unit test

**Files:**
- Create: `eval/src/main/java/io/casehub/eidos/eval/EvalDataset.java`
- Create: `eval/src/test/java/io/casehub/eidos/eval/EvalDatasetTest.java`

- [ ] **Step 1: Write failing test**

```java
package io.casehub.eidos.eval;

import io.casehub.eidos.api.SystemPromptRenderer.RenderFormat;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class EvalDatasetTest {

    @Test
    void all_returns_non_empty_list() {
        assertThat(EvalDataset.all()).isNotEmpty();
    }

    @Test
    void all_cases_have_valid_descriptors() {
        EvalDataset.all().forEach(c -> {
            assertThat(c.descriptor().agentId()).isNotBlank();
            assertThat(c.descriptor().name()).isNotBlank();
            assertThat(c.descriptor().slot()).isNotBlank();
            assertThat(c.descriptor().tenancyId()).isNotBlank();
        });
    }

    @Test
    void all_cases_use_claude_md_format() {
        EvalDataset.all().forEach(c ->
            assertThat(c.context().format()).isEqualTo(RenderFormat.CLAUDE_MD));
    }

    @Test
    void includes_minimal_and_maximal_cases() {
        final var names = EvalDataset.all().stream().map(EvalCase::name).toList();
        assertThat(names).contains("minimal", "maximal");
    }
}
```

- [ ] **Step 2: Create `EvalDataset`**

```java
package io.casehub.eidos.eval;

import io.casehub.eidos.api.*;
import io.casehub.eidos.api.SystemPromptRenderer.RenderFormat;

import java.util.List;
import java.util.Map;

public class EvalDataset {

    public static List<EvalCase> all() {
        return List.of(
            devtownPlanner(),
            crossVocab(),
            epistemicWeak(),
            minimal(),
            maximal()
        );
    }

    private static EvalCase devtownPlanner() {
        final var descriptor = new AgentDescriptor(
            "planner-1", "Devtown Planner", "1.0", "anthropic",
            "claude", "claude-3-5-sonnet", null,
            "https://vocab.casehub.io/devtown", "https://vocab.casehub.io/devtown", null,
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
        final var context = AgentPromptContext.forFormat(RenderFormat.CLAUDE_MD)
            .withGoal(new GoalContext("Plan sprint 42", List.of("Prioritise backlog", "Assign capacity"), "case-sprint-42"));
        return new EvalCase("devtown-planner", descriptor, context);
    }

    private static EvalCase crossVocab() {
        final var descriptor = new AgentDescriptor(
            "reviewer-1", "Code Reviewer", "2.0", "anthropic",
            "claude", "claude-3-7-sonnet", null,
            "https://vocab.casehub.io/svo", "https://vocab.casehub.io/devtown", null,
            "reviewer",
            List.of(
                new AgentCapability("code-review", 0.95, 150L, "low",
                    List.of("pull-request"), List.of("review-comment"), List.of(),
                    Map.of("java", 0.95, "rust", 0.4, "python", 0.7))
            ),
            new AgentDisposition("independent", "strict", "conservative", "directed", false),
            "EU", "gdpr-compliant", "devtown-1"
        );
        final var context = AgentPromptContext.forFormat(RenderFormat.CLAUDE_MD);
        return new EvalCase("cross-vocab", descriptor, context);
    }

    private static EvalCase epistemicWeak() {
        final var descriptor = new AgentDescriptor(
            "ml-agent-1", "ML Researcher", "1.0", "openai",
            "gpt", "gpt-4o", null, null, null, null,
            "researcher",
            List.of(
                new AgentCapability("literature-review", 0.6, 500L, "high",
                    List.of("papers"), List.of("summary"), List.of(),
                    Map.of("reinforcement-learning", 0.25, "supervised-learning", 0.8))
            ),
            null, null, null, null, "research-1"
        );
        final var context = AgentPromptContext.forFormat(RenderFormat.CLAUDE_MD)
            .withSituationalContext("Reviewing recent RL papers for quarterly report");
        return new EvalCase("epistemic-weak", descriptor, context);
    }

    private static EvalCase minimal() {
        final var descriptor = new AgentDescriptor(
            "min-1", "Minimal Agent", null, null, null, null, null,
            null, null, null,
            "worker",
            List.of(), null, null, null, "tenant-1"
        );
        final var context = AgentPromptContext.forFormat(RenderFormat.CLAUDE_MD);
        return new EvalCase("minimal", descriptor, context);
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
        final var context = AgentPromptContext.forFormat(RenderFormat.CLAUDE_MD)
            .withGoal(new GoalContext(
                "Coordinate quarterly planning cycle",
                List.of("Gather input from all teams", "Synthesise priorities", "Produce roadmap"),
                "case-q3-planning"))
            .withResources(List.of(
                new Resource("https://internal.company.io/roadmap", "Current Roadmap", "document"),
                new Resource("https://internal.company.io/okrs", "OKRs", "spreadsheet")
            ))
            .withSituationalContext("End of Q2 — all teams must submit priorities by Friday.");
        return new EvalCase("maximal", descriptor, context);
    }
}
```

- [ ] **Step 3: Run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -Dtest=EvalDatasetTest -q
```

Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add eval/src/main/java/io/casehub/eidos/eval/EvalDataset.java eval/src/test/java/io/casehub/eidos/eval/EvalDatasetTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#16): add EvalDataset — 5 inline fixture descriptors

Includes minimal, maximal, devtown-planner, cross-vocab, epistemic-weak.
All in CLAUDE_MD format (v1 scope — eidos#21 adds other formats).

Refs #16"
```

---

## Task 13: `EvalReportWriter` + tests

**Files:**
- Create: `eval/src/main/java/io/casehub/eidos/eval/EvalReportWriter.java`
- Create: `eval/src/test/java/io/casehub/eidos/eval/EvalReportWriterTest.java`

- [ ] **Step 1: Write failing tests**

```java
package io.casehub.eidos.eval;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.eidos.api.AgentPromptContext;
import io.casehub.eidos.api.SystemPromptRenderer.RenderFormat;
import io.casehub.eidos.api.SystemPromptRenderer.RenderedPrompt;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.time.Instant;
import java.util.EnumMap;
import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class EvalReportWriterTest {

    static EvalReport sampleReport() {
        final var desc = new AgentDescriptor(
            "a", "Agent", null, null, null, null, null,
            null, null, null, "worker", List.of(), null, null, null, "t");
        final var ctx = AgentPromptContext.forFormat(RenderFormat.CLAUDE_MD);
        final var rendered = new RenderedPrompt("You are Agent.", RenderFormat.CLAUDE_MD, "dh", "ch");
        final Map<EvalDimension, EvalScore> scores = new EnumMap<>(EvalDimension.class);
        for (final EvalDimension d : EvalDimension.values()) scores.put(d, new EvalScore(4, "good"));
        final var result = new EvalResult(new EvalCase("case1", desc, ctx), rendered,
            true, List.of(), scores, 4.0, List.of());
        return EvalReport.build(List.of(result), "test-judge");
    }

    @Test
    void summaryTable_returns_non_empty_string() {
        assertThat(EvalReportWriter.summaryTable(sampleReport())).isNotBlank();
    }

    @Test
    void summaryTable_contains_dimension_names() {
        final String table = EvalReportWriter.summaryTable(sampleReport());
        for (final EvalDimension d : EvalDimension.values()) {
            assertThat(table).containsIgnoringCase(d.name().replace('_', ' ').toLowerCase());
        }
    }

    @Test
    void writeJson_creates_valid_json_file(@TempDir Path dir) throws IOException {
        final Path out = dir.resolve("report.json");
        EvalReportWriter.writeJson(sampleReport(), out);
        assertThat(Files.exists(out)).isTrue();
        // Must be parseable JSON
        new ObjectMapper().findAndRegisterModules().readTree(out.toFile());
    }

    @Test
    void writeJson_round_trips_judge_model(@TempDir Path dir) throws IOException {
        final Path out = dir.resolve("report.json");
        EvalReportWriter.writeJson(sampleReport(), out);
        final var node = new ObjectMapper().findAndRegisterModules().readTree(out.toFile());
        assertThat(node.get("judgeModel").asText()).isEqualTo("test-judge");
    }
}
```

- [ ] **Step 2: Create `EvalReportWriter`**

```java
package io.casehub.eidos.eval;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.SerializationFeature;

import java.io.IOException;
import java.io.UncheckedIOException;
import java.nio.file.Path;
import java.util.Comparator;
import java.util.Map;

public class EvalReportWriter {

    private static final ObjectMapper MAPPER = new ObjectMapper()
            .findAndRegisterModules()
            .enable(SerializationFeature.INDENT_OUTPUT);

    public static String summaryTable(final EvalReport report) {
        final var sb = new StringBuilder();
        sb.append(String.format("%-20s %-20s %-8s%n", "Scenario", "Dimension", "Score"));
        sb.append("-".repeat(50)).append("\n");

        final EvalSummary summary = report.summary();
        summary.meanByDimension().entrySet().stream()
            .sorted(Comparator.comparingDouble(Map.Entry<EvalDimension, Double>::getValue)
                               .thenComparingInt(e -> e.getKey().ordinal()))
            .forEach(e -> sb.append(String.format("%-20s %-20s %.2f%n",
                "", e.getKey().name().toLowerCase().replace('_', ' '), e.getValue())));

        sb.append("\n");
        sb.append(String.format("Overall mean:      %.2f / 5.0%n", summary.meanOverall()));
        sb.append(String.format("All cases complete: %s%n", summary.allCasesComplete() ? "YES" : "NO"));
        sb.append(String.format("Lowest dimension:  %s%n",
            summary.lowestScoringDimension().name().toLowerCase().replace('_', ' ')));
        return sb.toString();
    }

    public static void writeJson(final EvalReport report, final Path path) {
        try {
            MAPPER.writeValue(path.toFile(), report);
        } catch (final IOException e) {
            throw new UncheckedIOException("Failed to write eval report to " + path, e);
        }
    }
}
```

- [ ] **Step 3: Run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -Dtest=EvalReportWriterTest -q
```

Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add eval/src/main/java/io/casehub/eidos/eval/EvalReportWriter.java eval/src/test/java/io/casehub/eidos/eval/EvalReportWriterTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#16): add EvalReportWriter — JSON and summary table output

Refs #16"
```

---

## Task 14: Completeness check unit test

**Files:**
- Create: `eval/src/test/java/io/casehub/eidos/eval/EvalResultCompletenessTest.java`

- [ ] **Step 1: Create test (validates the programmatic completeness check logic)**

```java
package io.casehub.eidos.eval;

import io.casehub.eidos.api.*;
import io.casehub.eidos.api.SystemPromptRenderer.RenderFormat;
import io.casehub.eidos.api.SystemPromptRenderer.RenderedPrompt;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class EvalResultCompletenessTest {

    static AgentDescriptor descriptorWithCaps(String... capNames) {
        final var caps = java.util.Arrays.stream(capNames)
            .map(n -> new AgentCapability(n, null, null, null, List.of(), List.of(), List.of(), Map.of()))
            .toList();
        return new AgentDescriptor("id", "Name", null, null, null, null, null,
            null, null, null, "worker", caps, null, null, null, "tenant");
    }

    static boolean computeCompleteness(AgentDescriptor descriptor, String renderedContent) {
        return descriptor.capabilities().stream()
            .allMatch(cap -> renderedContent.contains(cap.name()));
    }

    static List<String> missingCapabilities(AgentDescriptor descriptor, String renderedContent) {
        return descriptor.capabilities().stream()
            .map(AgentCapability::name)
            .filter(name -> !renderedContent.contains(name))
            .toList();
    }

    @Test
    void all_caps_present_returns_true() {
        final var desc = descriptorWithCaps("code-review", "estimation");
        final String content = "You can perform code-review and estimation tasks.";
        assertThat(computeCompleteness(desc, content)).isTrue();
        assertThat(missingCapabilities(desc, content)).isEmpty();
    }

    @Test
    void missing_cap_returns_false() {
        final var desc = descriptorWithCaps("code-review", "estimation");
        final String content = "You can perform code-review tasks."; // estimation missing
        assertThat(computeCompleteness(desc, content)).isFalse();
        assertThat(missingCapabilities(desc, content)).containsExactly("estimation");
    }

    @Test
    void no_caps_returns_true() {
        final var desc = descriptorWithCaps();
        assertThat(computeCompleteness(desc, "any content")).isTrue();
        assertThat(missingCapabilities(desc, "any content")).isEmpty();
    }
}
```

- [ ] **Step 2: Run test**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -Dtest=EvalResultCompletenessTest -q
```

Expected: BUILD SUCCESS (pure logic test, no CDI)

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add eval/src/test/java/io/casehub/eidos/eval/EvalResultCompletenessTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "test(eidos#16): unit-test completeness check logic

Refs #16"
```

---

## Task 15: `PromptJudge` + unit test

**Files:**
- Create: `eval/src/main/java/io/casehub/eidos/eval/PromptJudge.java`
- Create: `eval/src/test/java/io/casehub/eidos/eval/PromptJudgeTest.java`

- [ ] **Step 1: Write failing test**

```java
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

import static org.assertj.core.api.Assertions.assertThat;

class PromptJudgeTest {

    static final String VALID_JUDGE_JSON = """
        {
          "SECOND_PERSON":    { "score": 5, "reasoning": "Uses you/your throughout." },
          "CONCISENESS":      { "score": 4, "reasoning": "Mostly concise." },
          "FACTUAL_FIDELITY": { "score": 5, "reasoning": "No hallucinated data." },
          "TONE":             { "score": 4, "reasoning": "Reads like instructions." },
          "issues": ["Minor: capability latency not mentioned"]
        }""";

    PromptJudge judge;
    EvalCase evalCase;
    RenderedPrompt rendered;

    @BeforeEach
    void setUp() {
        final ChatModel stubJudge = new ChatModel() {
            @Override public ChatResponse doChat(ChatRequest request) {
                return ChatResponse.builder().aiMessage(AiMessage.from(VALID_JUDGE_JSON)).build();
            }
        };
        judge = new PromptJudge(stubJudge, new ObjectMapper());

        final var desc = new AgentDescriptor(
            "id", "Name", null, null, null, null, null, null, null, null,
            "worker",
            List.of(new AgentCapability("code-review", null, null, null,
                List.of(), List.of(), List.of(), Map.of())),
            null, null, null, "tenant");
        evalCase = new EvalCase("test", desc, AgentPromptContext.forFormat(RenderFormat.CLAUDE_MD));
        rendered = new RenderedPrompt("- **code-review**", RenderFormat.CLAUDE_MD, "dh", "ch");
    }

    @Test
    void evaluate_parses_scores_correctly() {
        final EvalResult result = judge.evaluate(evalCase, rendered);
        assertThat(result.scores().get(EvalDimension.SECOND_PERSON).score()).isEqualTo(5);
        assertThat(result.scores().get(EvalDimension.CONCISENESS).score()).isEqualTo(4);
        assertThat(result.scores().get(EvalDimension.FACTUAL_FIDELITY).score()).isEqualTo(5);
        assertThat(result.scores().get(EvalDimension.TONE).score()).isEqualTo(4);
    }

    @Test
    void evaluate_computes_correct_overall() {
        final EvalResult result = judge.evaluate(evalCase, rendered);
        assertThat(result.overall()).isEqualTo((5.0 + 4.0 + 5.0 + 4.0) / 4.0);
    }

    @Test
    void evaluate_detects_completeness_when_cap_present() {
        final EvalResult result = judge.evaluate(evalCase, rendered);
        assertThat(result.completenessPass()).isTrue();
        assertThat(result.missingCapabilities()).isEmpty();
    }

    @Test
    void evaluate_detects_missing_cap() {
        final RenderedPrompt noCaps = new RenderedPrompt("no caps here", RenderFormat.CLAUDE_MD, "dh", "ch");
        final EvalResult result = judge.evaluate(evalCase, noCaps);
        assertThat(result.completenessPass()).isFalse();
        assertThat(result.missingCapabilities()).containsExactly("code-review");
    }

    @Test
    void evaluate_always_calls_judge_regardless_of_completeness() {
        final boolean[] called = {false};
        final ChatModel trackingJudge = new ChatModel() {
            @Override public ChatResponse doChat(ChatRequest request) {
                called[0] = true;
                return ChatResponse.builder().aiMessage(AiMessage.from(VALID_JUDGE_JSON)).build();
            }
        };
        final RenderedPrompt noCaps = new RenderedPrompt("no caps here", RenderFormat.CLAUDE_MD, "dh", "ch");
        new PromptJudge(trackingJudge, new ObjectMapper()).evaluate(evalCase, noCaps);
        assertThat(called[0]).isTrue();
    }

    @Test
    void evaluate_extracts_issues_list() {
        final EvalResult result = judge.evaluate(evalCase, rendered);
        assertThat(result.issues()).containsExactly("Minor: capability latency not mentioned");
    }
}
```

- [ ] **Step 2: Run — verify FAIL**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -Dtest=PromptJudgeTest -q 2>&1 | tail -3
```

Expected: compilation error — `PromptJudge` does not exist

- [ ] **Step 3: Create `PromptJudge`**

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
import io.casehub.eidos.api.SystemPromptRenderer.RenderedPrompt;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.eclipse.microprofile.config.inject.ConfigProperty;

import java.util.ArrayList;
import java.util.EnumMap;
import java.util.List;
import java.util.Map;
import java.util.OptionalDouble;

import static io.quarkiverse.langchain4j.ModelName.MODEL_NAME_EXPRESSION;

@ApplicationScoped
public class PromptJudge {

    static final String SYSTEM_PROMPT = """
        You are evaluating the quality of an AI agent's system prompt.
        
        Given the agent's descriptor (JSON), its context, and the rendered system prompt text,
        score each dimension from 0 to 5 and provide brief reasoning.
        
        Dimensions:
        - SECOND_PERSON (0-5): Uses "you"/"your" consistently. 5 = every sentence is second person.
          0 = all third person ("the agent", "it"). Deduct 1 per third-person occurrence.
        - CONCISENESS (0-5): Every sentence carries unique information. No filler.
          5 = dense and efficient. 0 = heavily padded.
        - FACTUAL_FIDELITY (0-5): Nothing claimed that is absent from the descriptor or context.
          5 = every claim is grounded. 0 = significant hallucinations.
        - TONE (0-5): Reads as instructions to an AI agent, not documentation about one.
          5 = imperative, action-oriented throughout. 0 = reads like a bio or README.
        
        Return a JSON object with keys SECOND_PERSON, CONCISENESS, FACTUAL_FIDELITY, TONE
        (each with "score" int 0-5 and "reasoning" string) and an "issues" string array.
        """;

    static final ResponseFormat RESPONSE_FORMAT = ResponseFormat.builder()
        .type(ResponseFormatType.JSON)
        .jsonSchema(JsonSchema.builder()
            .name("EvalJudgment")
            .rootElement(JsonObjectSchema.builder()
                .addProperty("SECOND_PERSON",    scoreSchema())
                .addProperty("CONCISENESS",      scoreSchema())
                .addProperty("FACTUAL_FIDELITY", scoreSchema())
                .addProperty("TONE",             scoreSchema())
                .addProperty("issues", JsonArraySchema.builder()
                    .description("List of specific quality issues found, if any.")
                    .items(JsonStringSchema.builder().build())
                    .build())
                .required("SECOND_PERSON", "CONCISENESS", "FACTUAL_FIDELITY", "TONE", "issues")
                .build())
            .build())
        .build();

    private static JsonObjectSchema scoreSchema() {
        return JsonObjectSchema.builder()
            .addIntegerProperty("score", "Score 0-5")
            .addStringProperty("reasoning", "Brief reasoning for the score")
            .required("score", "reasoning")
            .build();
    }

    private final ChatModel judgeModel;
    private final ObjectMapper mapper;

    @Inject
    public PromptJudge(@io.quarkiverse.langchain4j.ModelName("judge") final ChatModel judgeModel,
                       final ObjectMapper mapper) {
        this.judgeModel = judgeModel;
        this.mapper = mapper;
    }

    // Package-private constructor for tests.
    PromptJudge(final ChatModel judgeModel, final ObjectMapper mapper) {
        this.judgeModel = judgeModel;
        this.mapper = mapper;
    }

    public EvalResult evaluate(final EvalCase evalCase, final RenderedPrompt rendered) {
        // 1. Programmatic completeness check (deterministic — not sent to LLM)
        final List<String> missing = evalCase.descriptor().capabilities().stream()
            .map(AgentCapability::name)
            .filter(n -> !rendered.content().contains(n))
            .toList();
        final boolean complete = missing.isEmpty();

        // 2. Judge LLM call — always made, regardless of completeness result
        final ObjectNode userPayload = buildUserPayload(evalCase, rendered);
        final Map<EvalDimension, EvalScore> scores;
        final List<String> issues;
        try {
            final var request = ChatRequest.builder()
                .messages(
                    SystemMessage.from(SYSTEM_PROMPT),
                    UserMessage.from(mapper.writeValueAsString(userPayload)))
                .responseFormat(RESPONSE_FORMAT)
                .build();
            final var response = judgeModel.chat(request);
            final var parsed = parseResponse(response.aiMessage().text());
            scores = parsed.scores();
            issues = parsed.issues();
        } catch (final Exception e) {
            throw new IllegalStateException("Judge LLM call failed — check judge model configuration", e);
        }

        // 3. Compute overall
        final double overall = scores.values().stream()
            .mapToInt(EvalScore::score)
            .average()
            .orElse(0.0);

        return new EvalResult(evalCase, rendered, complete, missing, scores, overall, issues);
    }

    private ObjectNode buildUserPayload(final EvalCase evalCase, final RenderedPrompt rendered) {
        final ObjectNode node = mapper.createObjectNode();
        try {
            node.set("descriptor", mapper.valueToTree(evalCase.descriptor()));
            node.put("rendered", rendered.content());
        } catch (final Exception e) {
            throw new IllegalStateException("Failed to build judge payload", e);
        }
        return node;
    }

    private record ParsedResponse(Map<EvalDimension, EvalScore> scores, List<String> issues) {}

    private ParsedResponse parseResponse(final String json) throws JsonProcessingException {
        final JsonNode root = mapper.readTree(json);
        final Map<EvalDimension, EvalScore> scores = new EnumMap<>(EvalDimension.class);

        // Iterate only known dimensions — skip "issues" and any unknown keys
        for (final EvalDimension d : EvalDimension.values()) {
            final JsonNode dimNode = root.get(d.name());
            if (dimNode == null) throw new IllegalStateException("Judge response missing dimension: " + d.name());
            scores.put(d, new EvalScore(dimNode.get("score").asInt(), dimNode.get("reasoning").asText()));
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

- [ ] **Step 4: Run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -Dtest=PromptJudgeTest -q
```

Expected: BUILD SUCCESS, 5 tests passing

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add eval/src/main/java/io/casehub/eidos/eval/PromptJudge.java eval/src/test/java/io/casehub/eidos/eval/PromptJudgeTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#16): add PromptJudge — LLM-as-judge with structured response schema

Completeness check is programmatic (before judge call). Judge always called.
Parses EvalDimension enum keys only; skips 'issues' during dimension loop.

Refs #16"
```

---

## Task 16: `EvalProfile` + `PromptEvalTest` + application.properties

**Files:**
- Create: `eval/src/test/java/io/casehub/eidos/eval/EvalProfile.java`
- Create: `eval/src/test/java/io/casehub/eidos/eval/PromptEvalTest.java`
- Create: `eval/src/test/resources/application.properties`
- Create: `eval/src/test/resources/application-eval.properties`

- [ ] **Step 1: Create `application.properties` (minimal Quarkus test config)**

```properties
# No renderer ChatModel configured — EidosSystemPromptRenderer uses structural fallback (v1 scope)
# Judge model configured in application-eval.properties via EvalProfile
quarkus.log.level=WARN
```

- [ ] **Step 2: Create `application-eval.properties` (template — git-ignored, user supplies real keys)**

```properties
# Eval profile — configure the named "judge" model.
# Set API key via environment variable: QUARKUS_LANGCHAIN4J_JUDGE_OPENAI_API_KEY=your-key
# Or use other providers — see Quarkus LangChain4j docs.

# Judge model must be at temperature 0.0 for deterministic, comparable results.
# quarkus.langchain4j.judge.openai.chat-model.model-name=gpt-4o-mini
# quarkus.langchain4j.judge.openai.chat-model.temperature=0.0
```

Add `application-eval.properties` to `.gitignore` (real keys must not be committed).

- [ ] **Step 3: Create `EvalProfile`**

```java
package io.casehub.eidos.eval;

import io.quarkus.test.junit.QuarkusTestProfile;

public class EvalProfile implements QuarkusTestProfile {
    @Override
    public String getConfigProfile() {
        return "eval";
    }
}
```

- [ ] **Step 4: Create `PromptEvalTest`**

```java
package io.casehub.eidos.eval;

import io.casehub.eidos.api.SystemPromptRenderer;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.TestProfile;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;

import java.nio.file.Path;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
@TestProfile(EvalProfile.class)
@Tag("eval")
class PromptEvalTest {

    private static final double SCORE_FLOOR = 3.5; // Set after first baseline run

    @Inject
    SystemPromptRenderer renderer;

    @Inject
    PromptJudge judge;

    @Test
    void evaluateAllScenarios() throws Exception {
        final List<EvalResult> results = EvalDataset.all().stream()
            .map(c -> judge.evaluate(c, renderer.render(c.descriptor(), c.context())))
            .toList();

        final EvalReport report = EvalReport.build(results, "judge"); // "judge" matches @ModelName
        EvalReportWriter.writeJson(report, Path.of("target/eval-report.json"));
        System.out.println(EvalReportWriter.summaryTable(report));

        // Structural check: every declared capability must appear in every rendered prompt
        assertThat(report.summary().allCasesComplete())
            .as("All rendered prompts must include every declared capability name")
            .isTrue();

        // Quality floor: update SCORE_FLOOR after first run
        // Run: JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -Peval -Dgroups=eval
        // Then: commit target/eval-report.json and set SCORE_FLOOR = max(3.0, baseline - 0.5)
        assertThat(report.summary().meanOverall())
            .as("Mean judge score across all cases and dimensions")
            .isGreaterThanOrEqualTo(SCORE_FLOOR);
    }
}
```

- [ ] **Step 5: Compile eval module**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -pl eval -q
```

Expected: BUILD SUCCESS (unit tests compile; PromptEvalTest excluded from CI)

- [ ] **Step 6: Run unit tests only (no LLM)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -q
```

Expected: BUILD SUCCESS — `PromptEvalTest` excluded by `<excludedGroups>eval</excludedGroups>` in Surefire; only unit tests run

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add \
  eval/src/test/java/io/casehub/eidos/eval/ \
  eval/src/test/resources/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#16): add PromptEvalTest — @QuarkusTest + %eval profile

EvalProfile configures named judge model only; no renderer ChatModel.
Structural fallback path ensures completenessPass is reliable.
CI excludes via Surefire excludedGroups=eval.

Closes #16"
```

---

## Task 17: Full build + promote spec

- [ ] **Step 1: Full clean build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -q
```

Expected: BUILD SUCCESS, all modules including `eval/`

- [ ] **Step 2: Promote spec to project repo**

```bash
cp /Users/mdproctor/claude/public/casehub/eidos/specs/2026-05-30-quality-eval-prompt-security-reactive-cache-design.md \
   /Users/mdproctor/claude/casehub/eidos/docs/superpowers/specs/
git -C /Users/mdproctor/claude/casehub/eidos add docs/superpowers/specs/2026-05-30-quality-eval-prompt-security-reactive-cache-design.md
git -C /Users/mdproctor/claude/casehub/eidos commit -m "docs: promote spec for eidos#16, #15, #19 to project repo

Refs #15 #16 #19"
```

- [ ] **Step 3: Push project branch**

```bash
git -C /Users/mdproctor/claude/casehub/eidos push -u origin issue-016-quality-eval-security
```
