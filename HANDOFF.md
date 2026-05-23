# CaseHub Eidos — Session Handover
**Date:** 2026-05-23

## Current State

Phases 1 and 2 complete and merged to main. 96 tests across 7 modules, all green.

Engine#341 design agreed: `Worker.agentDescriptor()` (optional, guarded by `hasDescriptor()`),
`NoOpCapabilityHealth @DefaultBean` in engine, `WorkOrchestrator` calls `probe()` at dispatch
time. `casehub-engine-api → casehub-eidos-api` optional dep registered in PLATFORM.md
cross-repo dependency table. `docs/repos/casehub-eidos.md` deep-dive created.

**ProbeContext.taskDomain ≠ capabilityTag** — documented in PLATFORM.md and garden
(GE-20260523-fa7407). Engine lacks task subject domain concept; EpistemicallyWeak stays
dormant until calling layer gets richer.

**Open design question:** `EpistemicallyWeak` is currently a hard filter. Should be a
preference signal (deprioritise, not gate) — zeroing out all candidates is worse than
dispatching a weak match.

## Immediate Next Step

Start Phase 3 — `work-start` for casehubio/eidos#5:
- `SystemPromptRenderer` + `ClaudeMarkdownRenderer` implementation
- Runtime state probing for `CapabilityHealth` (Degraded status)

## Cross-Module

**We're blocking:**
- `casehub-engine` — needs `casehub-eidos-api` as optional dep; design agreed (engine#341) · S · Low

## What's Left

- casehubio/eidos#2 — InMemoryAgentRegistry NPE on null slot · XS · Low
- casehubio/eidos#3 — minor findings batch from Phase 1 review · S · Low
- engine#341 EpistemicallyWeak filter semantics — pin down before implementing (soft preference vs hard gate)

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #5 | Phase 3: SystemPromptRenderer + runtime state probing | M | Med | |
| engine#341 | Wire CapabilityHealth.probe() into engine dispatch | S | Low | Cross-repo: casehub-engine; design agreed |
| — | Phase 4: Knowledge graph (descriptor, task, outcome, attestation) | L | High | |

## References

| What | Path |
|------|------|
| Phase 2 spec | `specs/2026-05-23-phase2-capability-health-design.md` |
| Blog entries | `blog/2026-05-23-mdp01-agent-identity-from-scratch.md`, `blog/2026-05-23-mdp02-closing-the-loop-on-dispatch.md` |
| Eidos deep-dive | `casehub/parent/docs/repos/casehub-eidos.md` |
| Eidos research | `research/eidos.md` |
