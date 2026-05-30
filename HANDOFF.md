# CaseHub Eidos — Session Handover
**Date:** 2026-05-30

## Current State

Three issues closed and delivered to upstream (`casehubio/eidos` main):

- **eidos#19** (architectural): `ReactiveRenderedPromptCache` is now the canonical cache SPI. Both renderers inject it. `EidosRenderPipeline` is stateless (only `VocabularyRegistry` + `ObjectMapper`). `BlockingToReactiveRenderedPromptCacheAdapter @DefaultBean` bridges blocking impls. Adapter has no `runSubscriptionOn` — adding it causes worker pool deadlock under saturation (garden entry GE-20260530-5e5c67).
- **eidos#15** (security): `AgentDescriptor` compact constructor validates 4 required fields at construction time via `AgentDescriptorValidator`. Rejects C0/C1, BiDi controls including U+061C (Arabic Letter Mark — not in C1 range, garden entry GE-20260530-e14065), zero-width chars, U+2028/U+2029.
- **eidos#16** (eval): `eval/` Maven module — `PromptJudge` + `EvalDataset` (5 fixtures) + `EvalReportWriter`. `PromptEvalTest @QuarkusTest @Tag("eval")` evaluates structural rendering against 4 LLM rubric dimensions. Completeness check is deterministic (substring match). 17 unit tests, CI excluded via Surefire `<excludedGroups>eval</excludedGroups>`.

Branch `issue-016-quality-eval-security` closed. Both repos on main. 22 commits squashed to 5 for upstream delivery.

## Immediate Next Step

Run `/work` for the next issue — Phase 4 knowledge graph is the natural next milestone. Or pick up eidos#20 (optional field validation) or eidos#21 (multi-format eval coverage).

## What's Left

- eidos#20 — Optional field + capability name validation in `AgentDescriptorValidator` (capability names and `jurisdiction`/`dataHandlingPolicy` are appended raw, injection surface pending) · S · Low
- eidos#21 — Multi-format eval coverage (A2A_CARD needs structural rubric; OPENAI_SYSTEM/GEMINI need format-specific criteria) · M · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Phase 4: knowledge graph (descriptor, task, outcome, attestation nodes) | L | High | Next major milestone |
| #20 | Optional + capability name field validation | S | Low | eidos#15 follow-on |
| #21 | Multi-format eval coverage | M | Med | eidos#16 follow-on |

## References

| What | Path |
|------|------|
| Latest blog | `blog/2026-05-30-mdp01-deadlock-in-the-adapter.md` |
| Spec (promoted) | `docs/superpowers/specs/2026-05-30-quality-eval-prompt-security-reactive-cache-design.md` (project repo) |
| Garden entries | GE-20260530-5e5c67 (runSubscriptionOn deadlock), GE-20260530-e14065 (U+061C BiDi) |
| Protocols | PP-20260530-b7b3be (ReactiveRenderedPromptCache canonical SPI), PP-20260530-2d6dbd (compact constructor validation) |
