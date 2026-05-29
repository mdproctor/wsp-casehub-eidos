# CaseHub Eidos — Session Handover
**Date:** 2026-05-29

## Current State

Phases 1–3 complete plus issue #6 (semantic rendering pipeline) — full three-stage
pipeline implemented, reviewed (3 rounds), and shipped. 362 tests, all green.

This session: implemented three-stage `ClaudeMarkdownRenderer` — Jackson payload
builder (Stage 1), `SemanticEnrichmentStep` with `ResponseFormat`+`JsonSchema`
LangChain4j (Stage 2), format-specific assembly for CLAUDE_MD/OPENAI_SYSTEM/A2A_CARD/
GEMINI (Stage 3). `RenderedPromptCache` SPI with no-op and in-memory impls. 6 clean
commits squashed and delivered to upstream. Garden: 3 entries (GE-20260528-e9564b,
GE-20260528-e9ed9f, GE-20260528-1e92db). Protocols: PP-20260529-35f3bd (LLM-pass
structural fallback), PP-20260529-5c883f (renderer cache key includes format).

## Immediate Next Step

Pick the next issue — Phase 4 knowledge graph or reactive parity (#7). Run `work-start`.

## What's Left

Nothing trailing from closed issues.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Phase 4: knowledge graph (descriptor, task, outcome, attestation) | L | High | |
| #7 | Reactive parity: AgentStateStore + SystemPromptRenderer | S | Med | After JPA persistence |
| #13 | A2A_CARD per-capability prose narratives | S | Med | Deferred from #6 |
| #14 | GEMINI enriched rendering + ClaudeMarkdownRenderer rename | S | Low | Deferred from #6 |

## References

| What | Path |
|------|------|
| Blog entries | `blog/2026-05-29-mdp01-two-architectural-errors.md` |
| ADR 0001 | `adr/0001-langchain4j-core-for-llm-semantic-pass.md` |
| Spec | `specs/2026-05-28-semantic-rendering-pipeline-design.md` |
| Design journal | `design/JOURNAL.md` |
| DESIGN.md | `design/DESIGN.md` |
