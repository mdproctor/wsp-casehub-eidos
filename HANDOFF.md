# CaseHub Eidos — Session Handover
**Date:** 2026-05-26

## Current State

Phases 1–3 complete and merged to `casehubio/eidos` main. 125 tests, all green.

This session: eidos#10 closed (null guard on `findById(tenancyId)` in all four registry
impls — InMemoryAgentRegistry, JpaAgentRegistry, JpaReactiveAgentRegistry; InMemoryReactive
inherits via delegation). Discovery: `@WithSession` CDI interceptor swallows synchronous
throws — reactive test requires `@RunOnVertxContext + UniAsserter.assertFailedWith`
(garden: GE-20260526-3c8553). Follow-on filed: eidos#11 (agentId null guard), eidos#12
(SPI Javadoc). parent#67 closed. engine#341 closed.

## Immediate Next Step

Before starting eidos#8: read `WorkOrchestrator.java` in
`/Users/mdproctor/claude/casehub/engine/runtime/src/main/java/io/casehub/engine/internal/orchestration/`
to verify whether `EpistemicallyWeak` is treated as soft preference (fallback) or still hard
filter. If correct: close eidos#8. If wrong: file engine issue, close eidos#8 as delegated.

## Cross-Module

*No active blockers.* engine#341 closed. eidos#8 is engine-side investigation only —
no eidos code change expected.

## What's Left

- casehubio/eidos#8 — EpistemicallyWeak soft preference (engine-side — see Immediate Next Step) · S · Low
- casehubio/eidos#11 — agentId null guard in all four findById impls · XS · Low
- casehubio/eidos#12 — Javadoc @throws on AgentRegistry / ReactiveAgentRegistry · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #8 | Verify EpistemicallyWeak in engine; close or delegate | XS | Low | Read WorkOrchestrator first |
| #11 | agentId null guard in all four findById impls | XS | Low | |
| #12 | @throws Javadoc on SPI interfaces | XS | Low | |
| — | Phase 4: knowledge graph (descriptor, task, outcome, attestation) | L | High | |
| #6 | Semantic rendering pipeline research | M | High | Research issue |
| #7 | Reactive parity: AgentStateStore + SystemPromptRenderer | S | Med | After JPA persistence |

## References

| What | Path |
|------|------|
| Blog entries | `blog/2026-05-26-mdp01-interceptors-eat-exceptions.md` |
| ADR 0001 | `adr/0001-langchain4j-core-for-llm-semantic-pass.md` |
| Design journal | `design/JOURNAL.md` |
| Eidos deep-dive | `casehub/parent/docs/repos/casehub-eidos.md` |
