---
layout: post
title: "Hold no threads"
date: 2026-05-29
type: phase-update
entry_type: note
subtype: diary
projects: [eidos]
tags: [reactive, streaming, langchain4j, mutiny]
---

Two issues on this branch: one a single line, one an architectural redesign.

The single line first. `DefaultReactiveCapabilityHealth.probe()` was wrapping its
blocking delegate in `Uni.createFrom().item(supplier)` without
`.runSubscriptionOn(Infrastructure.getDefaultWorkerPool())`. Every other reactive
bridge in the codebase has it. This one was missed when the reactive layer was added
in the previous session. Consistent with the protocol now.

---

The second issue was more interesting. `DefaultReactiveSystemPromptRenderer` was reactive
in name only. It offloaded the full `EidosSystemPromptRenderer.render()` call to a worker
thread via `runSubscriptionOn` — which meant the LLM call inside (`ChatModel.chat()`) held
that thread for the duration of inference. Could be several seconds. With a real deployment
running multiple concurrent renders, that's a thread pool you're burning through on LLM wait.

The fix was `StreamingChatModel`. LangChain4j's streaming API is callback-based:
you call `chat(request, handler)`, the call returns immediately, and `onCompleteResponse`
fires when the LLM finishes. No thread held.

But before I started building it, we ran the spec through a review pass. Three threading
errors surfaced that I'd missed in the initial design.

**Stage 1 was going to run on the event loop.** The plan had `buildDescriptorPayload` and
`buildContextPayload` — Jackson JSON tree construction, vocabulary lookups — executing
synchronously inside `render()` before returning any `Uni`. That's event loop work. Not I/O
blocking, but enough CPU for Vert.x to complain at load. The fix: wrap Stage 1 in
`Uni.createFrom().item(...).runSubscriptionOn(workerPool)`.

**Stage 3 was going to run on the streaming callback thread.** After combining the two
enrichment `Uni`s via `Uni.combine().all().unis(...).asTuple()`, the `.map()` that runs
format assembly and cache writes would execute on whichever thread completed the last `Uni`.
For the streaming path that's the provider's callback thread — not somewhere you want to
do Jackson serialisation. The fix: `.emitOn(Infrastructure.getDefaultWorkerPool())` before
`.map()`.

**`emitter()` was the wrong bridging primitive.** The initial spec proposed
`Uni.createFrom().emitter()` to wrap the streaming callback. It works in the happy path —
but `emitter()` has no built-in timeout. A provider that never fires `onCompleteResponse`
or `onError` hangs the `Uni` indefinitely. The right primitive for a one-shot async result
is `CompletableFuture` with `orTimeout()`, bridged to Mutiny via `completionStage()`:

```java
CompletableFuture<Optional<SemanticEnrichment>> future = new CompletableFuture<>();
llm.chat(request, new StreamingChatResponseHandler() {
    public void onCompleteResponse(ChatResponse r) {
        future.complete(parseOrEmpty(r.aiMessage().text()));
    }
    public void onError(Throwable e) { future.completeExceptionally(e); }
});
return Uni.createFrom().completionStage(
        future.orTimeout(EidosRenderPipeline.STREAMING_TIMEOUT_SECONDS, TimeUnit.SECONDS))
    .onFailure().recoverWithItem(e -> {
        log.warn("Reactive enrichment failed, falling back");
        return Optional.empty();
    });
```

The `onError` path calls `completeExceptionally` rather than completing with empty —
the `onFailure().recoverWithItem()` handles both timeout and error uniformly and the
fallback is explicit at the Mutiny level, not buried in the callback.

---

Before the reactive renderer could be rewritten, all the format assembly logic had to
move somewhere both the blocking and reactive renderers could reach it. That became
`EidosRenderPipeline` — a package-private `@ApplicationScoped` CDI bean holding the
shared constants (`PROMPT_TEMPLATE`, `RESPONSE_FORMAT`, both A2A equivalents, the timeout
value), Stage 1 payload building, Stage 3 format assembly, and cache management. The
blocking and reactive step classes both reference `EidosRenderPipeline.*` rather than
declaring local constants. That's roughly 400 lines that moved out of
`EidosSystemPromptRenderer`.

The rewritten `DefaultReactiveSystemPromptRenderer` has a clear three-stage structure:

```java
return Uni.createFrom()
    .item(() -> buildStageOne(descriptor, context))       // payload + cache check
    .runSubscriptionOn(Infrastructure.getDefaultWorkerPool())
    .chain(s1 -> {
        if (s1.cached() != null) return Uni.createFrom().item(s1.cached());
        return Uni.combine().all().unis(enrichUni, a2aUni).asTuple()
            .emitOn(Infrastructure.getDefaultWorkerPool())   // back to worker pool
            .map(t -> pipeline.assembleAndCache(...));        // Stage 3
    });
```

Stage 2 (the `enrichUni` and `a2aUni`) runs between the two thread hops — the streaming
calls return immediately, the `CompletableFuture`s resolve on the provider's callback
thread, and Mutiny combines the two `Uni`s without holding anything. The thread pool sees
Stage 1 briefly, then Stage 3 briefly. The LLM wait happens in the gap.

Falls back to `Uni.createFrom().item(() -> blockingDelegate.render(...)).runSubscriptionOn(workerPool)`
when no `StreamingChatModel` is present — preserving the existing behaviour for deployments
without a streaming LLM configured.

135 tests passing across `api` and `runtime`. The one test gap worth noting: the streaming
renderer tests all use `CLAUDE_MD` format. The `A2A_CARD` path through the reactive
renderer — which routes to `ReactiveA2ASemanticEnrichmentStep` instead of
`ReactiveSemanticEnrichmentStep` — is covered at the unit level but not at the integration
level. Filed as something to address when the next test pass happens.
