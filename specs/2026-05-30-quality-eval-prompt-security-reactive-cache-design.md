# Design: Quality Eval, Prompt Security, Reactive Cache

**Date:** 2026-05-30  
**Branch:** issue-016-quality-eval-security  
**Issues:** eidos#16 (offline quality eval), eidos#15 (prompt injection), eidos#19 (ReactiveRenderedPromptCache)  
**Deferred:** eidos#20 (optional + capability name field validation), eidos#21 (multi-format eval coverage)

---

## 0. Shared Foundation: `EidosRenderPipeline` Refactor

All three issues benefit from first making `EidosRenderPipeline` stateless.

**Current state:** pipeline injects `RenderedPromptCache` and exposes `cacheGet()` + `assembleAndCache()`. Both renderers duplicate Stage 1 (5 identical lines; `StageOneResult` is a private record trapped in the reactive renderer).

**After refactor:**

- `RenderedPromptCache` injection removed from `EidosRenderPipeline`
- `cacheGet()` and `assembleAndCache()` removed
- `buildStage1(AgentDescriptor, AgentPromptContext) → StageOneResult` added as a shared pipeline method
- `assemble(...)` signature extended to accept `StageOneResult` and returns `RenderedPrompt` directly — eliminates boilerplate `RenderedPrompt` construction in callers

`StageOneResult` — promoted to package-private record in `renderer/`:

```java
record StageOneResult(
    ObjectNode descriptorNode,
    ObjectNode contextNode,
    String descriptorHash,
    String contextHash,
    String cacheKey
)
```

`assemble(...)` new signature:

```java
RenderedPrompt assemble(
    StageOneResult s1,
    Optional<SemanticEnrichment> enrichment,
    Optional<A2AEnrichment> a2aEnrichment,
    AgentDescriptor descriptor,
    AgentPromptContext context
)
// Returns: new RenderedPrompt(content, context.format(), s1.descriptorHash(), s1.contextHash())
```

Pipeline now depends only on `VocabularyRegistry` (pure lookup) and `ObjectMapper` (stateless) — trivially unit-testable, thread-safe by construction, no observable side effects.

**`EidosRenderPipelineTest` constructor update:** `new EidosRenderPipeline(vocab, mapper)` — no `RenderedPromptCache` arg.

---

## 1. eidos#15 — Prompt Injection: `AgentDescriptor` Validation

### Field Audit

| Field | Status | Evidence |
|-------|--------|----------|
| `agentId` | **Required** | `node.put("agentId", ...)` — no null guard, pipeline:143 |
| `name` | **Required** | `node.put("name", ...)` — no null guard, pipeline:144 |
| `slot` | **Required** | `node.put("slot", ...)` — no null guard, pipeline:154 |
| `tenancyId` | **Required** | All registry queries tenancy-scoped; null breaks them |
| `capabilities[*].name()` | **Injection surface** | Appended raw: `sb.append("- **").append(cap.name()).append("**")` at pipeline:362. A name containing `\n## Injected Section\n` injects Markdown structure. Scoped to eidos#20. |
| `jurisdiction`, `dataHandlingPolicy` | **Unescaped** | Appended raw in `assembleClaudeMarkdownStructural`. Injection surface pending eidos#20. Do not describe as "guarded." |
| Other optional fields | Optional | Guarded by `addIfPresent` / null checks in payload builder, but appended raw in structural assembly in some cases — pending eidos#20. |

### Design

**`AgentDescriptor` compact constructor:**

```java
public AgentDescriptor {
    AgentDescriptorValidator.validate(agentId, name, slot, tenancyId);
}
```

**`AgentDescriptorValidator`** — package-private static utility in `io.casehub.eidos.api`. Single static `validate()` method keeps the compact constructor to one line and makes validation rules independently readable and testable.

**`AgentDescriptorValidationException`** — **public** `extends IllegalArgumentException` in `api/`. Callers in `api/` consumers need to catch it specifically. Constructor takes field name + violation description.

Validation rules per required field:

| Rule | Detail |
|------|--------|
| Non-null, non-blank | `null` or whitespace-only → rejected |
| Length limit | `agentId`/`tenancyId` ≤ 255; `name` ≤ 200; `slot` ≤ 100. Bounds chosen to cap cache key length and LLM payload size. |
| No C0/C1 control chars | Codepoints 0–31 and 127–159 rejected |
| No Unicode direction overrides | U+200E, U+200F, U+202A–202E, U+2066–2069 rejected |
| No zero-width / line-separator chars | U+200B, U+FEFF, U+200C, U+200D, U+2028 (LINE SEPARATOR), U+2029 (PARAGRAPH SEPARATOR) rejected. U+2028/U+2029 sit above U+009F (not C1) but function as line breaks in many LLM tokenizers. |

No keyword blocklist — length + character set is the correct defence for open-string fields. Domain-defined slot values cannot be whitelisted at the platform level.

### Capability name and optional field injection (eidos#20 scope clarification)

Capability names are **not** optional — they are a concrete injection surface appended raw into structural renders. `jurisdiction` and `dataHandlingPolicy` are also appended raw. eidos#20 must cover:
- `AgentCapability.name()` — apply same character-set + length rules
- All optional `AgentDescriptor` string fields sent to rendered output unescaped

### Testing

Parameterized test `AgentDescriptorValidatorTest` — `@ParameterizedTest` over cases:
- Each required field: null, blank, over length limit, each banned character class
- Each case asserts `AgentDescriptorValidationException` with correct field name
- Happy path: valid descriptor constructs without exception

---

## 2. eidos#19 — `ReactiveRenderedPromptCache` SPI

### Canonical SPI Decision

`ReactiveRenderedPromptCache` is the canonical cache SPI. Both renderers inject it. This eliminates the coherence problem that arises when two independent SPIs (`RenderedPromptCache` and `ReactiveRenderedPromptCache`) could be backed by different stores simultaneously.

`RenderedPromptCache` remains in `api/` as a **convenience blocking SPI** for implementations that don't want Mutiny. A single adapter (`BlockingToReactiveRenderedPromptCacheAdapter`) wraps whichever `RenderedPromptCache` is active, satisfying the `ReactiveRenderedPromptCache` injection point. Both renderers always see the same backing store.

### New in `casehub-eidos-api`

```java
public interface ReactiveRenderedPromptCache {

    /**
     * Returns the cached prompt for the given key.
     * On failure, implementations must recover and return Uni<Optional.empty()> —
     * a cache miss must never abort a render.
     */
    Uni<Optional<RenderedPrompt>> get(String cacheKey);

    /**
     * Stores a rendered prompt. Must not emit a failure — implementations recover
     * internally so a cache write failure never aborts a render.
     */
    Uni<Void> put(String cacheKey, RenderedPrompt result);
}
```

### New in `runtime/cache/` — `BlockingToReactiveRenderedPromptCacheAdapter`

Location: `runtime/cache/` — not `runtime/renderer/`. This is general infrastructure, not renderer-specific.

```java
@DefaultBean
@ApplicationScoped
class BlockingToReactiveRenderedPromptCacheAdapter implements ReactiveRenderedPromptCache {
    // Injects RenderedPromptCache (blocking — NoOp by default, or @Alternative @Priority(1))

    // get: Uni.createFrom().item(() -> blocking.get(key))
    //          .onFailure().recoverWith(e -> Uni.createFrom().item(Optional.empty()))
    // NO runSubscriptionOn — both callers (blocking renderer, reactive renderer Stage 1)
    // are already on the worker pool. Adding runSubscriptionOn(workerPool) here would
    // schedule the supplier onto the same pool the calling thread already occupies —
    // deadlock under saturation.

    // put: Uni.createFrom().item(() -> { blocking.put(key, result); return null; })
    //          .onFailure().recoverWithNull()
    //          .replaceWithVoid()
    // NO runSubscriptionOn — same reasoning as get().
}
```

No `@IfBuildProperty` gate — per PP-20260529-5745c1. No Hibernate Reactive dependency; safe unconditionally.

### CDI Resolution

| Scenario | `RenderedPromptCache` | `ReactiveRenderedPromptCache` | Backing store |
|----------|-----------------------|-------------------------------|---------------|
| Default | `NoOpRenderedPromptCache @DefaultBean` | `BlockingToReactiveRenderedPromptCacheAdapter @DefaultBean` (wraps NoOp) | NoOp — coherent |
| Custom blocking (e.g. LRU) | `MyLruCache @Alternative @Priority(1)` | Adapter `@DefaultBean` (wraps MyLruCache) | MyLruCache — coherent |
| Custom native reactive (e.g. Redis) | `NoOpRenderedPromptCache @DefaultBean` | `MyRedisCache @Alternative @Priority(1)` | Redis — coherent |

### Renderer Updates

**`EidosSystemPromptRenderer`:**
- Injects `ReactiveRenderedPromptCache` directly
- Cache check:
  ```java
  Optional<RenderedPrompt> cached;
  try {
      cached = reactiveCache.get(s1.cacheKey()).await().atMost(Duration.ofSeconds(5));
  } catch (Exception e) {
      cached = Optional.empty(); // cache miss on any failure — render continues
  }
  ```
- For the default adapter, `await()` resolves synchronously (no thread switch) — `atMost` cannot fire. The try-catch defends against future async implementations that violate the error contract or genuinely time out.
- Safe: blocking renderer runs on request thread / worker pool, not event loop

**`DefaultReactiveSystemPromptRenderer`:**
- Injects `ReactiveRenderedPromptCache` directly
- Stage 1 reactive chain:

```java
Uni.createFrom()
   .item(() -> pipeline.buildStage1(descriptor, context))
   .runSubscriptionOn(workerPool)
   .chain(s1 -> reactiveCache.get(s1.cacheKey())
                              .chain(hit -> hit.isPresent()
                                  ? Uni.createFrom().item(hit.get())
                                  : executeStagesTwoAndThree(s1, descriptor, context)))
```

Stage 3 ends with `reactiveCache.put(s1.cacheKey(), result).replaceWith(result)`.

### Threading note

The adapter has no `runSubscriptionOn`. Both callers are already on the worker pool when they call `reactiveCache.get()`:
- Reactive renderer: Stage 1 is scheduled via `runSubscriptionOn(workerPool)` before chaining into `reactiveCache.get()` — one hop total for Stage 1.
- Blocking renderer: the request thread is already a worker pool thread; `await()` resolves the `Uni` synchronously with no scheduling.

For a future native async `ReactiveRenderedPromptCache` (`@Alternative @Priority(1)`), the `Uni` returned by `get()` is inherently non-blocking and needs no `runSubscriptionOn` either.

### Test restructuring for cache-hit invariant

`DefaultReactiveSystemPromptRendererStreamingTest.cache_hit_returns_without_any_llm_call()` currently wires `TestRenderedPromptCache` through the pipeline. After refactor:

1. Add `ReactiveRenderedPromptCache cache` param to the package-private test constructor of `DefaultReactiveSystemPromptRenderer`
2. Test pre-computes the cache key: call `pipeline.buildStage1(descriptor, context)` to get `s1.cacheKey()`
3. Pre-populate a `TestReactiveRenderedPromptCache` (simple `ConcurrentHashMap`-backed impl in `src/test/`) with that key
4. Inject the pre-populated cache into the renderer via the test constructor
5. Call `render()` — assert the `StreamingChatModel` is never called

`EidosRenderPipelineTest.setUp()`: change from `new EidosRenderPipeline(vocab, cache, mapper)` to `new EidosRenderPipeline(vocab, mapper)`.

`EidosSystemPromptRendererTest`: add `ReactiveRenderedPromptCache` param to package-private test constructor.

---

## 3. eidos#16 — Offline Quality Evaluation Harness

### Module

New top-level Maven module: `eval/` (`casehub-eidos-eval`), sibling to `api/`, `runtime/`, `deployment/`.

- Depends on `casehub-eidos` (runtime) + LangChain4j `ChatModel` for judge
- `maven-deploy-plugin` configured with `<skip>true</skip>` — not released, not in consumer transitive graph
- CI exclusion: parent POM or `eval/` module POM sets `<excludedGroups>eval</excludedGroups>` in Surefire — prevents CI from attempting to run `@Tag("eval")` tests that require a real LLM
- Activated manually: `mvn test -Peval -pl eval`
- `Instant` serialization: Quarkus Jackson extension registers `JavaTimeModule` automatically — no additional dep needed in a `@QuarkusTest` context

### EvalDataset — module dependency resolution

`eval/` cannot import from `examples/agent-scenarios/` without a declared Maven dependency. **Chosen approach: define descriptors inline in `EvalDataset`** — keeps `eval/` self-contained with no cross-module test dependency. The fixture descriptors are small; duplication is acceptable. `EvalDataset` is the canonical eval fixture; `examples/agent-scenarios/` remains the CDI demo fixture.

### Core Types (`io.casehub.eidos.eval`)

```java
record EvalCase(String name, AgentDescriptor descriptor, AgentPromptContext context)

// COMPLETENESS removed from LLM dimensions — see below
enum EvalDimension {
    SECOND_PERSON,    // "you"/"your" throughout; no "the agent", "it", third person
    CONCISENESS,      // no redundancy, no filler, dense information per sentence
    FACTUAL_FIDELITY, // nothing claimed that is absent from descriptor + context
    TONE              // reads as instructions to an agent, not documentation about one
}

record EvalScore(int score, String reasoning)  // 0–5

record EvalResult(
    EvalCase evalCase,
    RenderedPrompt rendered,
    boolean completenessPass,         // structural check — see below
    List<String> missingCapabilities, // capabilities not found in rendered.content()
    Map<EvalDimension, EvalScore> scores,
    double overall,                   // 0.0–5.0; mean of EvalDimension scores (COMPLETENESS excluded)
    List<String> issues
)

record EvalSummary(
    boolean allCasesComplete,
    Map<EvalDimension, Double> meanByDimension,
    EvalDimension lowestScoringDimension, // if tied, first in EvalDimension declaration order
    double meanOverall
)

record EvalReport(
    Instant timestamp,
    String judgeModel,
    // No top-level format field — format is per-case via evalCase.context().format()
    List<EvalResult> results,
    EvalSummary summary
)
```

### COMPLETENESS: programmatic structural check, not LLM judgment

Completeness is deterministic and cheap:

```java
boolean complete = descriptor.capabilities().stream()
    .allMatch(cap -> rendered.content().contains(cap.name()));
List<String> missing = descriptor.capabilities().stream()
    .map(AgentCapability::name)
    .filter(name -> !rendered.content().contains(name))
    .toList();
```

Removed from `EvalDimension` — LLM judges answer it inconsistently across runs. Reported separately as `completenessPass` / `missingCapabilities` in `EvalResult`. The test asserts `allCasesComplete == true` independently of the LLM score floor.

### Judge Design

`PromptJudge` — `@ApplicationScoped` bean in `eval/`:

- One LLM call per case (after completeness check)
- Injects `@ModelName("judge") ChatModel judgeModel` — **must be a separate named model** from the renderer's default `ChatModel`. Using the same model instance means the judge evaluates output it produced, defeating independence.
- **Temperature: must be 0.0** on the judge model — otherwise scores differ across runs and the threshold assertion is non-deterministic.

**`%eval` profile configuration** (`application-eval.properties`):
```properties
# Renderer model — default ChatModel used by EidosSystemPromptRenderer
quarkus.langchain4j.openai.chat-model.model-name=gpt-4o

# Judge model — named "judge", temperature 0 for determinism
quarkus.langchain4j.judge.openai.chat-model.model-name=gpt-4o-mini
quarkus.langchain4j.judge.openai.chat-model.temperature=0.0
```
(Provider names are examples; real values are caller-supplied secrets in `application-eval.properties`, git-ignored.)

**Judge request:**
```
System: <rubric definition — each EvalDimension, scale 0-5, scoring guidance>
User: { "descriptor": {...}, "context": {...}, "rendered": "<rendered content>" }
```

**Judge response schema** (LangChain4j `ResponseFormat` with JSON schema):

```json
{
  "SECOND_PERSON":    { "score": 5, "reasoning": "..." },
  "CONCISENESS":      { "score": 4, "reasoning": "..." },
  "FACTUAL_FIDELITY": { "score": 5, "reasoning": "..." },
  "TONE":             { "score": 4, "reasoning": "..." },
  "issues": ["..."]
}
```

**Parsing:** `mapper.readTree(json)`, then iterate `EvalDimension.values()` only — use each enum name as the key. Extract `issues` separately via `node.get("issues")`. Do not attempt to deserialize unknown keys as `EvalScore`. Throws (does not swallow) on parse failure — judge misconfiguration is not a graceful degradation scenario.

**Error contract:** if judge LLM not configured → CDI startup failure → clear error.

### EvalDataset

Inline fixture descriptors (not imported from `examples/`):
- Devtown planner (multi-capability, epistemic domains, disposition)
- Cross-vocabulary agent (two vocabulary URIs)
- Minimal (only required fields: `agentId`, `name`, `slot`, `tenancyId`)
- Maximal (all fields: all capabilities, full disposition, goal, resources, situational context)
- Epistemic weak (capability with low domain confidence)

Initial scope: `CLAUDE_MD` format only. Multi-format coverage deferred to eidos#21.

### Test Class

```java
@QuarkusTest
@TestProfile(EvalProfile.class)
class PromptEvalTest {
    @Inject SystemPromptRenderer renderer;
    @Inject PromptJudge judge;

    @Test
    @Tag("eval")
    void evaluateAllScenarios() throws Exception {
        List<EvalResult> results = EvalDataset.all().stream()
            .map(c -> judge.evaluate(c, renderer.render(c.descriptor(), c.context())))
            .toList();

        EvalReport report = EvalReport.build(results);
        EvalReportWriter.writeJson(report, Path.of("target/eval-report.json"));
        System.out.println(EvalReportWriter.summaryTable(report)); // returns String

        // Structural check: all capabilities must appear in every rendered prompt
        assertThat(report.summary().allCasesComplete()).isTrue();

        // Score floor: set after first baseline run
        // Initial value TBD — run eval once, observe scores, set to max(3.0, baseline − 0.5)
        // Commit target/eval-report.json alongside the threshold decision
        assertThat(report.summary().meanOverall()).isGreaterThanOrEqualTo(SCORE_FLOOR);
    }

    private static final double SCORE_FLOOR = 3.5; // update after first run
}
```

`EvalProfile` sets real `ChatModel` (renderer) + judge at temperature 0.0. CI Surefire excludes `@Tag("eval")`.

`EvalReportWriter.summaryTable(report)` returns `String` — caller decides output stream.

---

## 4. Testing Strategy

### #15 — `AgentDescriptorValidator`

`AgentDescriptorValidatorTest` (pure Java, no CDI):
- `@ParameterizedTest` over ~20 cases: each required field × each rule (null, blank, over-length, each banned codepoint class: C0, C1, BiDi override, zero-width, U+2028, U+2029)
- Each case: asserts `AgentDescriptorValidationException` with correct field name in message
- Happy path: valid descriptor constructs without exception

### Pipeline refactor

- `EidosRenderPipelineTest` constructor: `new EidosRenderPipeline(vocab, mapper)` — no cache arg
- `buildStage1()` unit test: correct hashes, correct key derivation, `StageOneResult` fields
- All existing rendering tests must remain green — refactor is transparent to rendered output

### #19 — `ReactiveRenderedPromptCache`

- `BlockingToReactiveRenderedPromptCacheAdapterTest`: `get()` delegates to blocking; `get()` returns `Optional.empty()` on failure (does not propagate); `put()` delegates; `put()` recovers on failure (Uni completes, does not fail)
- `DefaultReactiveSystemPromptRendererStreamingTest` — cache-hit test restructured per §2 above: pre-populate `TestReactiveRenderedPromptCache` via test constructor; assert `StreamingChatModel` never called
- `EidosSystemPromptRendererTest`: add `ReactiveRenderedPromptCache` to test constructor; existing cache-hit test updated
- Parity test: blocking and reactive renderers produce identical output for same input (extends `BlockingReactiveParityTest`)

### #16 — Eval harness unit tests

- `PromptJudgeTest` with stub `ChatModel` returning valid judge JSON — verifies `EvalResult` construction, dimension scores, `overall` calculation
- `EvalDatasetTest` — all cases have valid descriptors (non-null required fields)
- `EvalReportWriterTest` — JSON round-trip; `summaryTable()` returns non-empty String
- `EvalResult` completeness check: unit test with known descriptor + rendered content with one missing capability name

---

## 5. Deferred Issues

| Issue | Title | Scope clarification |
|-------|-------|---------------------|
| eidos#20 | Optional + capability name field validation | `AgentCapability.name()` is an active injection surface (appended raw in structural render). `jurisdiction`, `dataHandlingPolicy` are also appended raw. Scope eidos#20 to cover these explicitly, not just "optional fields." |
| eidos#21 | Multi-format eval coverage | A2A_CARD needs structural rubric (JSON, not prose); OPENAI_SYSTEM and GEMINI need format-specific dimensions. `EvalReport` has no top-level `format` field — safe to extend. |
