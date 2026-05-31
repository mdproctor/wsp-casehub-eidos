# CaseHub Eidos — Session Handover
**Date:** 2026-05-31

## Current State

Three issues closed and delivered to `casehubio/eidos` main (3 squashed commits):

- **eidos#21** (foundational): `RenderFormat` renamed to structure-named (`MARKDOWN`, `PROSE`, `A2A_CARD`). `OPENAI_SYSTEM` + `GEMINI` collapsed into `PROSE` — structurally identical, one space in resource formatting is not a format. Multi-format eval harness: `EvalDimension.COMPLETENESS` + `applicableFor(RenderFormat)`, `EvalReport` format-grouped, `PromptJudge` format-aware with JSON-aware A2A completeness check, 9 `EvalDataset` cases.
- **eidos#22** (security): `AgentCapability` compact constructor validates all string fields. `AgentDescriptorValidationException` → `AgentValidationException`. `AgentDescriptorValidator` extended with `validateOptional`/`validateItems`/`validateMapKeys`.
- **eidos#20** (security): `AgentDescriptor` optional field validation (10 fields). `AgentDisposition` compact constructor (4 axes).

Branch `issue-021-020-eval-coverage-validation` closed. Both repos on main. 12 commits squashed to 3 for upstream delivery.

## Immediate Next Step

Run `/work` for the next issue — Phase 4 knowledge graph is the natural next milestone. Or pick up eidos#24 (minor code review follow-ups) or eidos#23 (real-world agent profile library — now unblocked).

## What's Left

- eidos#24 — Minor code review follow-ups: `validateRequired` wrapper comment, `MAX_MODEL_IDENTIFIER` constant rename, `dataHandlingPolicy`/`MAX_JURISDICTION` comment, multi-format `EvalReportWriterTest`, `PromptJudge` re-throw comment · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Phase 4: knowledge graph (descriptor, task, outcome, attestation nodes) | L | High | Next major milestone; eidos#23 should precede this |
| #23 | Real-world agent profile library (internet + Google Scholar research) | M | Med | Now unblocked — #21, #20, #22 closed; should complete before knowledge graph |
| #24 | Minor code review follow-ups from this branch | XS | Low | Cosmetic; batch into one commit |

## References

| What | Path |
|------|------|
| Latest blog | `blog/2026-05-31-mdp01-format-names-matter.md` |
| Design spec | `docs/superpowers/specs/2026-05-31-multi-format-eval-validation-design.md` (project repo) |
| Garden entries | GE-20260531-4354e3 (A2A substring completeness), GE-20260531-686150 (enum dimension set regression), GE-20260531-5e6553 (applicableFor technique), GE-20260531-afc422 (structure-named format enum) |
| Protocol | PP-20260531-60dc12 (render-format-structure-naming) |
| Minor follow-ups | eidos#24 |
