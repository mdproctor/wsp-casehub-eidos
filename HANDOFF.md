# CaseHub Eidos — Session Handover
**Date:** 2026-05-25

## Current State

Phases 1–3 complete and merged to `casehubio/eidos` main. 121 tests across 7 modules, all green.

Phase 3 delivered: `SystemPromptRenderer` SPI redesigned (`AgentPromptContext` replaces `String goal + RenderContext`); `ClaudeMarkdownRenderer @DefaultBean` (structural YAML + optional LangChain4j `ChatModel` semantic pass); `AgentStateStore` SPI + `NoOpAgentStateStore @DefaultBean` + `InMemoryAgentStateStore @Alternative @Priority(1)`; `DefaultCapabilityHealth` checks store first; `DegradationReason` promoted to top-level.

CI fixed: `casehub-eidos-deployment` was missing 8 transitive `-deployment` deps — added.

`docs/repos/casehub-eidos.md` deep-dive created in parent. `PLATFORM.md` updated with SystemPromptRenderer + AgentStateStore capability rows, eidos/engine boundary rule, cross-repo dep row. ADR 0001 (LangChain4j core choice) written.

## Immediate Next Step

Pick up eidos#2 (InMemoryAgentRegistry NPE on null slot — XS) or eidos#3 (minor Phase 1 findings — S), or start Phase 4 with `work-start`.

## Cross-Module

**We're blocking:**
- `casehub-engine` — engine#341 design agreed; eidos#8 (EpistemicallyWeak as soft preference, not hard filter) should be resolved before engine implements dispatch filtering · S · Low

## What's Left

- casehubio/eidos#2 — InMemoryAgentRegistry NPE on null slot · XS · Low
- casehubio/eidos#3 — minor findings from Phase 1 review · S · Low
- casehubio/eidos#8 — EpistemicallyWeak soft preference (engine-side) · S · Low
- casehubio/eidos#9 — minor renderer findings (hash doc, test helpers, StubStateStore comment) · XS · Low
- casehubio/parent#67 — PLATFORM.md + eidos deep-dive sync for Phase 3 · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #2 | InMemoryAgentRegistry null slot NPE | XS | Low | |
| #3 | Phase 1 minor findings batch | S | Low | |
| engine#341 | Wire CapabilityHealth.probe() into engine dispatch | S | Low | Cross-repo; design agreed; eidos#8 first |
| — | Phase 4: knowledge graph (descriptor, task, outcome, attestation) | L | High | |
| #6 | Semantic rendering pipeline research (LLM prompt templates, YAML schema) | M | High | Research issue |
| #7 | Reactive parity: AgentStateStore + SystemPromptRenderer | S | Med | After JPA persistence |

## References

| What | Path |
|------|------|
| Phase 3 spec | `specs/2026-05-24-phase3-system-prompt-renderer-design.md` |
| Blog entries | `blog/2026-05-25-mdp01-rendering-with-and-without-brain.md` |
| ADR 0001 | `adr/0001-langchain4j-core-for-llm-semantic-pass.md` |
| Design journal | `design/JOURNAL.md` |
| Eidos deep-dive | `casehub/parent/docs/repos/casehub-eidos.md` |
