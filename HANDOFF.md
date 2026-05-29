# CaseHub Eidos — Session Handover
**Date:** 2026-05-29

## Current State

Phases 1–3 + issues #6, #7, #13, #14 complete and shipped to upstream. Branch
`issue-007-reactive-parity-rendering` closed. Both repos on `main`.

This session: reactive parity SPIs (`ReactiveAgentStateStore`, `ReactiveSystemPromptRenderer`),
bridge impls with `runSubscriptionOn(workerPool)`, JPA blocking + Hibernate Reactive impls,
`DefaultReactiveCapabilityHealth` fixed (removed incorrect `@IfBuildProperty` gate),
`ClaudeMarkdownRenderer` → `EidosSystemPromptRenderer` rename, proper GEMINI format
assembly, `A2ASemanticEnrichmentStep` (separate schema + descriptor-only payload — caught
during code review, not implementation). 7→3 commits squashed, delivered to upstream.
Garden: 3 entries. Protocols: PP-20260529-5745c1, PP-20260529-368527. ADR-0002.

## Immediate Next Step

Run `work-start` and pick the next issue — Phase 4 knowledge graph is the natural
next milestone.

## What's Left

- eidos#17 — truly async `ReactiveSystemPromptRenderer` via `StreamingChatModel` (deferred; needs LangChain4j streaming + ResponseFormat compatibility investigation) · S · Med
- eidos#18 — `DefaultReactiveCapabilityHealth` missing `runSubscriptionOn` (pre-existing, filed this session; one-liner fix) · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Phase 4: knowledge graph (descriptor, task, outcome, attestation nodes) | L | High | Next major milestone |
| #17 | Truly async ReactiveSystemPromptRenderer via StreamingChatModel | S | Med | Deferred — LangChain4j streaming investigation needed |
| #18 | DefaultReactiveCapabilityHealth — add runSubscriptionOn | XS | Low | Pre-existing; filed this session |

## References

| What | Path |
|------|------|
| Latest blog | `blog/2026-05-29-mdp02-reactive-parity-a2a-enrichment.md` |
| ADR 0002 | `adr/0002-separate-a2a-enrichment-from-narrative-enrichment.md` |
| Design journal | `design/JOURNAL.md` |
