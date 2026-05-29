---
layout: post
title: "Reactive bridges, a naming debt, and an enrichment schema flaw"
date: 2026-05-29
type: phase-update
entry_type: note
subtype: diary
projects: [eidos]
---

The most useful part of this branch was catching a schema design problem before it reached code.

The plan for A2A per-capability narratives proposed adding `capabilityNarratives` as a
required field to the existing `SemanticEnrichmentStep` schema. Three formats already
use that schema — CLAUDE_MD, OPENAI_SYSTEM, GEMINI — and none of them need per-capability
prose. For an agent with five capabilities, every CLAUDE_MD and OPENAI_SYSTEM render would
generate five sentences and throw them away. Token costs scale with the agent roster.

There was also a payload problem. `buildLlmPayload` includes goal context from the caller —
correct for narrative enrichment, where the current goal shapes how role and capability
sections are framed. For A2A cards it's wrong: an A2A card describes what an agent can do
in general, not what it's doing right now. Goal context in an A2A enrichment call would
contaminate stable descriptor metadata with transient request state.

I asked Claude to review the design before implementation started. It caught both. The
fix: a dedicated `A2ASemanticEnrichmentStep` with its own schema (only `capabilityNarratives`),
its own prompt, and a descriptor-only payload. `render()` now has two enrichment stages —
Stage 2a for narrative enrichment (unchanged), Stage 2b for A2A enrichment (new). Two
schemas, two payloads.

What struck me was that neither problem would have been caught by a test.
`usesEnrichment(A2A_CARD)` had always returned false, so A2A tests never touched the LLM
path; the waste would only have appeared on a real bill. The payload contamination would
have been invisible — capability descriptions slightly coloured by goal context, probably
not wrong enough to notice but structurally incorrect.

## runSubscriptionOn, not emitOn

The other technically interesting piece: reactive bridges for `AgentStateStore` and
`SystemPromptRenderer`. Both delegates do blocking work — one JPA, one HTTP to an LLM.

`emitOn(Infrastructure.getDefaultWorkerPool())` looks like the right call for shifting
blocking work off the Vert.x event loop. For `Uni.createFrom().item(supplier)` it isn't.
The supplier evaluates during subscription, before `emitOn` has any effect on the pipeline.
`emitOn` dispatches the resulting signal downstream; `runSubscriptionOn` controls where
the subscription request runs — and therefore where the supplier evaluates.

```java
// Wrong — supplier still runs on the subscription thread
return Uni.createFrom()
    .item(() -> delegate.render(descriptor, context))
    .emitOn(Infrastructure.getDefaultWorkerPool());

// Right — supplier runs on the worker pool
return Uni.createFrom()
    .item(() -> delegate.render(descriptor, context))
    .runSubscriptionOn(Infrastructure.getDefaultWorkerPool());
```

We used `runSubscriptionOn` throughout. The mistake produces no compile error, no test
failure — just a blocked-thread event in production when the first real LLM call arrives.

## A name four formats too late

`ClaudeMarkdownRenderer` was always misnamed — it was just less obviously wrong when it
produced one format. It now produces four: CLAUDE_MD, OPENAI_SYSTEM, GEMINI, A2A_CARD.
We renamed it to `EidosSystemPromptRenderer`.

GEMINI had been delegating to `assembleClaudeMarkdown` as a placeholder. Gemini system
instructions want plain prose, no `#` headers. The enriched path concatenates narratives
with blank-line separators; the structural path mirrors OPENAI_SYSTEM's dense-prose shape
but in a separate method, so format divergence later requires no structural change. One
deliberate difference: resource format. OPENAI_SYSTEM writes `label (uri)` with a space
before the paren; GEMINI writes `label(uri)` without. Gemini's instruction register
tolerates less punctuation.
