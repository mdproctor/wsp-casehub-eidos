---
layout: post
title: "Three Issues, One Pipeline"
date: 2026-05-30
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-eidos]
tags: [reactive, validation, eval, prompt-injection, cache-coherence]
---

# CaseHub Eidos — Three Issues, One Pipeline

**Date:** 2026-05-30
**Type:** phase-update

---

## What I expected vs. what was actually one problem

Three open issues: a reactive cache SPI (eidos#19), AgentDescriptor field validation against prompt injection (eidos#15), and an offline quality evaluation harness (eidos#16). I expected them to be independent work. They weren't.

The cache question collapsed everything else. `EidosRenderPipeline` had been injecting `RenderedPromptCache` since phase 3 — keeping cache logic close to assembly felt natural. The problem becomes visible only when you ask: if a user installs an `@Alternative ReactiveRenderedPromptCache` backed by Redis, the blocking renderer still reads from `RenderedPromptCache`. Two backing stores serving the same render surface. Cache entries written via one renderer invisible to the other. A coherence bug waiting behind a `@DefaultBean`.

I brought Claude in to design and build all three. We pulled cache out of the pipeline entirely. `EidosRenderPipeline` now depends only on `VocabularyRegistry` (pure lookup) and `ObjectMapper` (stateless). It's trivially testable and thread-safe by construction. Both renderers inject `ReactiveRenderedPromptCache` directly; `ReactiveRenderedPromptCache` is the canonical SPI. A `BlockingToReactiveRenderedPromptCacheAdapter @DefaultBean` wraps whatever `RenderedPromptCache` is active — so simple blocking implementations still work without touching Mutiny.

## The deadlock hiding in the adapter

The first adapter design was:

```java
return Uni.createFrom()
          .item(() -> blocking.get(cacheKey))
          .onFailure().recoverWith(e -> Uni.createFrom().item(Optional.empty()))
          .runSubscriptionOn(Infrastructure.getDefaultWorkerPool());
```

`runSubscriptionOn(workerPool)` is the standard way to keep blocking work off the event loop. It's wrong here.

Both callers are already on the worker pool. The blocking renderer runs on the request thread, which is a worker pool thread. The reactive renderer's Stage 1 is scheduled via its own `runSubscriptionOn(workerPool)` before the cache call. When the adapter then schedules the `get()` supplier onto the same pool, the calling thread blocks waiting for a slot — on the pool it already occupies. Under saturation all threads are waiting on each other. The symptom would be intermittent hangs under load, not startup failures. The kind of bug that survives staging.

Claude's code review caught it. The fix:

```java
// No runSubscriptionOn — callers are already on the worker pool
return Uni.createFrom()
          .item(() -> blocking.get(cacheKey))
          .onFailure().recoverWith(e -> Uni.createFrom().item(Optional.empty()));
```

The supplier runs synchronously on the calling thread, which is already correct.

## Validation at the record boundary

Eidos#15 was straightforward on the surface: add field-level protection against adversarial strings in `AgentDescriptor`. Fields like `name`, `slot`, and capability names flow directly into the LLM payload as JSON — an adversarial string in a capability name lands in the enrichment prompt. JSON structure is not a barrier; the LLM reads all of it.

The design decision was where to validate. Service boundary (registry `register()`) means every registry implementation has to remember to call a validator, and descriptors can be constructed for testing with null or invalid required fields. Compact constructor means every descriptor in the system is guaranteed valid at construction time, full stop. Registries need zero validation code.

```java
public AgentDescriptor {
    AgentDescriptorValidator.validate(agentId, name, slot, tenancyId);
}
```

Two tests in `InMemoryAgentRegistryTest` that constructed descriptors with null slots had to go — they were testing behaviour that's now impossible. That's the correct outcome.

The validator rejects C0/C1 control characters (0–31, 127–159), BiDi override and isolate characters (LRE, RLE, PDF, LRO, RLO, LRI, RLI, FSI, PDI), zero-width joiners, and U+2028/U+2029 (line and paragraph separators). Claude's review caught a gap: U+061C — the Arabic Letter Mark, added in Unicode 6.3. It sits in the Arabic block, nowhere near the ranges everyone checks. One line:

```java
if (cp == 0x061C) return true;   // ALM — Arabic Letter Mark (BiDi control)
```

## What the eval harness actually measures

Eidos#16 was the most open-ended. The brief was quality evaluation for rendered system prompts using an LLM-as-judge approach — give a second LLM the descriptor, the context, and the rendered output, score it on a rubric.

The first design question was what "completeness" means. LLM judges are inconsistent on structured checks — whether all declared capabilities appear in the rendered prose varies by run and by enrichment path. We moved completeness to a deterministic string check:

```java
boolean complete = descriptor.capabilities().stream()
    .allMatch(cap -> rendered.content().contains(cap.name()));
```

Structural rendering emits capability names verbatim; the check is reliable there. For v1 the harness evaluates structural rendering only — the eval profile configures no renderer `ChatModel`, so `EidosSystemPromptRenderer` falls back to its structural path. One LLM call per scenario, for the judge, not the renderer.

Four rubric dimensions: second-person voice, conciseness, factual fidelity, and tone. Five fixture descriptors: minimal (required fields only), maximal (all fields), devtown planner, cross-vocabulary agent, epistemic weak. The module lives in `eval/`, is never deployed (`maven-deploy-plugin skip=true`), and is excluded from CI via Surefire `<excludedGroups>eval</excludedGroups>`.

The project uses plain `langchain4j-core` — no Quarkus LangChain4j extension — so `@ModelName("judge")` is not available. `PromptJudge` uses `@Any Instance<ChatModel>` with a clear startup failure message when no judge is configured. That's the same pattern the existing renderers use; it's the right approach for this codebase.
