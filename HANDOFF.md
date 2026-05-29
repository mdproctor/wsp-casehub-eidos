# CaseHub Eidos — Session Handover
**Date:** 2026-05-29

## Current State

Both issues closed and delivered to upstream (`casehubio/eidos` main):

- **eidos#18** (one-liner): `DefaultReactiveCapabilityHealth.probe()` missing `runSubscriptionOn` — fixed.
- **eidos#17** (architectural): `DefaultReactiveSystemPromptRenderer` rewritten with true non-blocking streaming. `EidosRenderPipeline` extracted as shared `@ApplicationScoped` CDI bean. `ReactiveSemanticEnrichmentStep` + `ReactiveA2ASemanticEnrichmentStep` bridge `StreamingChatModel` callbacks via `CompletableFuture` + `Uni.createFrom().completionStage()`. Three-stage threading: Stage 1 on workerPool, Stage 2 async streaming, Stage 3 `emitOn(workerPool)`. 135 tests, BUILD SUCCESS.

Branch `issue-018-17-reactive-streaming` closed. Both repos on main.

## Immediate Next Step

Run `work-start` for the next issue — Phase 4 knowledge graph is the natural next milestone. Or pick up eidos#19 (ReactiveRenderedPromptCache SPI).

## What's Left

- eidos#19 — ReactiveRenderedPromptCache SPI (filed this session; deferred correctness risk — safe to leave until a real cache backend appears) · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Phase 4: knowledge graph (descriptor, task, outcome, attestation nodes) | L | High | Next major milestone |
| #19 | ReactiveRenderedPromptCache SPI | S | Low | Deferred — no external cache impl yet |

## References

| What | Path |
|------|------|
| Latest blog | `blog/2026-05-29-mdp03-hold-no-threads.md` |
| Spec (promoted) | `docs/superpowers/specs/2026-05-29-reactive-streaming-renderer-design.md` (project repo) |
| Design journal | `design/JOURNAL.md` (empty — no journal entries this branch) |
