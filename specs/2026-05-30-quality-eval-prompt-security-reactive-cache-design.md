# Design: Quality Eval, Prompt Security, Reactive Cache

**Date:** 2026-05-30  
**Branch:** issue-016-quality-eval-security  
**Issues:** eidos#16 (offline quality eval), eidos#15 (prompt injection), eidos#19 (ReactiveRenderedPromptCache)  
**Deferred:** eidos#20 (optional field validation), eidos#21 (multi-format eval coverage)

---

## 0. Shared Foundation: `EidosRenderPipeline` Refactor

All three issues benefit from first making `EidosRenderPipeline` stateless.

**Current state:** pipeline injects `RenderedPromptCache` and exposes `cacheGet()` + `assembleAndCache()`. Both renderers duplicate Stage 1 (5 identical lines; `StageOneResult` is a private record trapped in the reactive renderer).

**After refactor:**

- `RenderedPromptCache` injection removed from `EidosRenderPipeline`
- `cacheGet()` and `assembleAndCache()` removed
- `buildStage1(AgentDescriptor, AgentPromptContext) → StageOneResult` added as a shared pipeline method
- `StageOneResult` promoted to package-private record in `renderer/`:

```java
record StageOneResult(
    ObjectNode descriptorNode,
    ObjectNode contextNode,
    String descriptorHash,
    String contextHash,
    String cacheKey
)
```

- `assemble(...)` unchanged — returns `String`
- Pipeline now depends only on `VocabularyRegistry` (pure lookup) and `ObjectMapper` (stateless) — trivially unit-testable, thread-safe by construction, no observable side effects

**`EidosSystemPromptRenderer` after:**
1. Injects `RenderedPromptCache` directly
2. Calls `pipeline.buildStage1(descriptor, context)`
3. Checks `cache.get(s1.cacheKey())`; returns hit if present
4. Runs enrichment steps (unchanged)
5. Calls `pipeline.assemble(...)`; wraps in `RenderedPrompt`; calls `cache.put(...)`

**`DefaultReactiveSystemPromptRenderer` after:**
1. Injects `RenderedPromptCache` directly (blocking; Stage 1 runs on worker pool — correct by design)
2. Stage 1: `Uni.item(() -> pipeline.buildStage1(...)).runSubscriptionOn(workerPool)`, then inline cache check
3. Stage 2+3: unchanged threading model
4. Stage 3: `pipeline.assemble(...)` → `new RenderedPrompt(...)` → `cache.put(...)`

---

## 1. eidos#15 — Prompt Injection: `AgentDescriptor` Validation

### Field Audit

| Field | Status | Reason |
|-------|--------|--------|
| `agentId` | **Required** | `node.put("agentId", ...)` — no null guard in pipeline |
| `name` | **Required** | `node.put("name", ...)` — no null guard in pipeline |
| `slot` | **Required** | `node.put("slot", ...)` — no null guard in pipeline |
| `tenancyId` | **Required** | All registry queries are tenancy-scoped; null breaks them |
| All others | Optional | Guarded with `addIfPresent` or null checks throughout pipeline |

### Design

**`AgentDescriptor` compact constructor:**

```java
public AgentDescriptor {
    AgentDescriptorValidator.validate(agentId, name, slot, tenancyId);
}
```

**`AgentDescriptorValidator`** — package-private static utility in `io.casehub.eidos.api`:

- `static void validate(String agentId, String name, String slot, String tenancyId)`
- Throws `AgentDescriptorValidationException` (field name + violation message)

Rules per required field:

| Rule | Detail |
|------|--------|
| Non-null, non-blank | `null` or whitespace-only → rejected |
| Length limit | `agentId`/`tenancyId` ≤ 255; `name` ≤ 200; `slot` ≤ 100 |
| No C0/C1 control chars | Codepoints 0–31 and 127–159 rejected |
| No Unicode direction overrides | U+200E, U+200F, U+202A–202E, U+2066–2069 rejected |
| No zero-width characters | U+200B, U+FEFF, U+200C, U+200D rejected |

No keyword blocklist — length + character set is the correct defence for open-string fields. Domain-defined slot values cannot be whitelisted at the platform level.

**`AgentDescriptorValidationException extends IllegalArgumentException`** — new in `api/`. Constructor takes field name + violation description.

### Why compact constructor (not registry boundary)

Every `AgentDescriptor` in the system is guaranteed valid at construction time. No registry implementation needs to call a separate validator. Callers catching `IllegalArgumentException` catch `AgentDescriptorValidationException` automatically.

### Optional field validation

Deferred to eidos#20. The four required fields are the active injection surface (sent to LLM unconditionally). Optional fields are included in the payload only when present, but character-set validation on them is worth adding in a follow-up.

---

## 2. eidos#19 — `ReactiveRenderedPromptCache` SPI

### New in `casehub-eidos-api`

```java
public interface ReactiveRenderedPromptCache {
    Uni<Optional<RenderedPrompt>> get(String cacheKey);

    /** Must not throw — implementations handle errors internally. */
    Uni<Void> put(String cacheKey, RenderedPrompt result);
}
```

### New in `runtime/renderer/` — `ReactiveRenderedPromptCacheBridge`

```java
@DefaultBean
@ApplicationScoped
class ReactiveRenderedPromptCacheBridge implements ReactiveRenderedPromptCache {
    // Injects RenderedPromptCache (blocking)
    // get: Uni.createFrom().item(() -> blocking.get(key))
    //          .runSubscriptionOn(Infrastructure.getDefaultWorkerPool())
    // put: Uni.createFrom().item(() -> { blocking.put(key,r); return null; })
    //          .replaceWithVoid()
    //          .runSubscriptionOn(Infrastructure.getDefaultWorkerPool())
}
```

No `@IfBuildProperty` gate — per PP-20260529-5745c1. No Hibernate Reactive dependency; safe to activate unconditionally.

### `DefaultReactiveSystemPromptRenderer` updated

Stage 1 becomes a reactive chain using `ReactiveRenderedPromptCache`:

```java
Uni.createFrom()
   .item(() -> pipeline.buildStage1(descriptor, context))
   .runSubscriptionOn(workerPool)
   .chain(s1 -> reactiveCache.get(s1.cacheKey())
                              .chain(hit -> hit.isPresent()
                                  ? Uni.createFrom().item(hit.get())
                                  : executeStagesTwoAndThree(s1, descriptor, context)))
```

Stage 3 ends with `reactiveCache.put(cacheKey, result).replaceWith(result)`.

### CDI resolution

| Type | Bean | Activation |
|------|------|-----------|
| `RenderedPromptCache` | `NoOpRenderedPromptCache @DefaultBean` | Always (unchanged) |
| `ReactiveRenderedPromptCache` | `ReactiveRenderedPromptCacheBridge @DefaultBean` | Always |
| Future reactive impl (Redis) | `@Alternative @Priority(1)` | Classpath presence |

`EidosSystemPromptRenderer` continues to inject `RenderedPromptCache` (blocking) — unaffected.

---

## 3. eidos#16 — Offline Quality Evaluation Harness

### Module

New top-level Maven module: `eval/` (`casehub-eidos-eval`).

- Depends on `casehub-eidos` (runtime) + LangChain4j `ChatModel` for judge
- `maven-deploy-plugin` configured with `<skip>true</skip>` — not released, not in consumer transitive graph
- Activated with `-Peval` Maven profile; excluded from CI

### Core Types (`io.casehub.eidos.eval`)

```java
record EvalCase(String name, AgentDescriptor descriptor, AgentPromptContext context)

enum EvalDimension {
    COMPLETENESS,     // all declared capabilities and non-null fields represented
    SECOND_PERSON,    // "you"/"your" throughout; no third person
    CONCISENESS,      // no redundancy, no filler, dense information
    FACTUAL_FIDELITY, // nothing claimed that isn't in descriptor + context
    TONE              // reads as instructions to an AI, not documentation about one
}

record EvalScore(int score, String reasoning)  // 0–5

record EvalResult(
    EvalCase evalCase,
    RenderedPrompt rendered,
    Map<EvalDimension, EvalScore> scores,
    double overall,
    List<String> issues
)

record EvalSummary(
    Map<EvalDimension, Double> meanByDimension,
    EvalDimension lowestScoringDimension,
    double meanOverall
)

record EvalReport(
    Instant timestamp,
    String judgeModel,
    RenderFormat format,
    List<EvalResult> results,
    EvalSummary summary
)
```

### Judge Design

`PromptJudge` — `@ApplicationScoped` bean in `eval/`:

- Single LLM call per case
- System message: rubric definition (each `EvalDimension`, 0–5 scale, scoring guidance)
- User message: `{ "descriptor": {...}, "context": {...}, "rendered": "..." }`
- Response format: LangChain4j `ResponseFormat` with JSON schema — one `{score, reasoning}` per dimension + `issues` array
- Throws (does not swallow) on parse failure — judge failure is a configuration problem, not a graceful degradation

The judge model is configured independently of the rendering model via `%eval` profile. Misconfigured judge → CDI startup failure → clear error, not silent fallback.

### Eval Dataset

`EvalDataset` provides the initial set of `EvalCase` instances:

- Reuses descriptors from `examples/agent-scenarios/` (devtown team, cross-vocab, epistemic, tenancy, disposition)
- Adds a minimal case (only required fields: `agentId`, `name`, `slot`, `tenancyId`)
- Adds a maximal case (all fields populated including all capability fields and full disposition)

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
        EvalReportWriter.printSummary(report, System.out);

        assertThat(report.summary().meanOverall()).isGreaterThanOrEqualTo(3.5);
    }
}
```

`EvalProfile` configures a real `ChatModel` (judge). CI never runs `@Tag("eval")` tests.

`EvalReportWriter` writes `target/eval-report.json`. The file is committed manually when checkpointing scores.

### Threading

`PromptJudge.evaluate()` is blocking — it uses a `ChatModel`, not `StreamingChatModel`. Called from a `@QuarkusTest` on the test thread. No reactive concerns.

---

## 4. Testing Strategy

### #15 — `AgentDescriptorValidator`

- Unit tests in `api/` (pure Java, no CDI): happy path, each required field failing each rule independently
- Null `agentId`, blank `name`, control char in `slot`, direction override in `tenancyId`
- `AgentDescriptorValidationException` carries correct field name in each case
- Existing `AgentDescriptorTest` extended (or new `AgentDescriptorValidatorTest`)

### #19 — `ReactiveRenderedPromptCache`

- `ReactiveRenderedPromptCacheBridge` unit test: verify `get()` delegates to blocking cache on worker pool; verify `put()` delegates; verify errors are swallowed in `put()` (per `RenderedPromptCache` contract)
- `DefaultReactiveSystemPromptRenderer` test: existing tests verify cache hit path; add test for reactive cache miss → enrichment → cache put
- Parity test: blocking and reactive renderers produce identical output for same input (extends `BlockingReactiveParityTest`)

### Pipeline refactor

- `EidosRenderPipelineTest` extended: `buildStage1()` returns correct hashes and key; verifies `StageOneResult` fields
- Existing rendering tests remain green — refactor is transparent to callers

### #16 — Eval harness

- `PromptJudge` unit test with a stub `ChatModel` — verifies JSON schema parsing and `EvalResult` construction
- `EvalDataset` unit test — verifies all cases have non-null required fields
- `EvalReportWriter` unit test — round-trip JSON serialisation
- The `@Tag("eval")` integration test is the acceptance test: run manually with a real LLM configured

---

## 5. Deferred Issues

| Issue | Title | Why deferred |
|-------|-------|--------------|
| eidos#20 | Optional field validation in `AgentDescriptorValidator` | Required fields cover the active LLM injection surface; optional fields are guarded in the pipeline |
| eidos#21 | Multi-format eval coverage | A2A_CARD needs a different rubric; OPENAI_SYSTEM/GEMINI need format-specific criteria; scope kept tight for initial harness |
