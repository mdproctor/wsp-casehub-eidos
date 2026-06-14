# Independent Judge Comparison — Design Spec

**Date:** 2026-06-14  
**Issue:** eidos#51 (Phase 2b)  
**Status:** Approved

## Problem

`evaluateRealWorldScenarios()` uses the same `ChatModel` for both rendering and judging. When Claude renders and Claude judges, high scores might reflect style affinity, not quality. We need one non-Anthropic judge run on Claude-rendered outputs to detect self-evaluation bias.

## Design: Two-Step Workflow (Option A)

### Step 1 — Render and cache (existing run, extended)

At the end of `evaluateRealWorldScenarios()`, serialize the `renders` map to `target/renders-cache.json` as a list of `RenderCacheEntry` objects:

```
record RenderCacheEntry(String caseName, String format, String content)
```

`caseName` matches `ProfiledEvalCase.name()` — used to look up the full case on reload.

### Step 2 — Independent judge run (new test method)

`evaluateWithIndependentJudge()`:
1. Load `target/renders-cache.json` → `List<RenderCacheEntry>`
2. Match each entry back to `ProfiledEvalCase` from `realWorldCases` by `caseName`
3. Reconstruct `RenderedPrompt(content, format, null, null)` — hashes not needed for re-judging
4. Run `PromptJudge` + `ProximityJudge` on the saved renders
5. Write `target/independent-judge-report.json` and `target/independent-proximity-report.json`
6. Print a comparison table (independent judge scores vs Claude baseline from `real-world-eval-report.json`)
7. No assertion — diagnostic only; user reads the divergence

### Inject ObjectMapper

`PromptEvalTest` needs `@Inject ObjectMapper mapper` to write/read the cache. Jackson `ObjectMapper` is a Quarkus CDI bean — inject at field level.

## What we learn

- Scores similar → self-evaluation bias not material; Claude baseline is valid
- Independent scores lower → Claude baseline is optimistic; floors need recalibration against a neutral judge

## Run sequence

```bash
# Step 1: render with Claude, save cache
mvn test -pl eval -Peval -Dtest=PromptEvalTest#evaluateRealWorldScenarios

# Step 2: judge with Qwen 35B on Claude's renders
mvn clean test -pl eval -Peval,eval-ollama \
  -Dtest=PromptEvalTest#evaluateWithIndependentJudge \
  -Dcasehub.eval.claude-provider.enabled=false \
  "-Dquarkus.langchain4j.ollama.chat-model.model-name=qwen3.6:35b-a3b" \
  "-Dquarkus.langchain4j.ollama.chat-model.format=json" \
  "-Dquarkus.langchain4j.ollama.timeout=300s" \
  -Dcasehub.eval.model.label=qwen3-35b
```

## Out of scope

- Committing render cache (transient; regenerated each Phase 2a run)
- Dual `@Judge`/`@Renderer` qualifier wiring (permanent complexity for a one-off diagnostic)
- Expressiveness/trait/pair-contrast judges (bias check only needs quality + proximity)
