# eidos Session Handover — 2026-06-10

*Updated: parent#216, parent#220 closed — removed from backlog.*

## Last Session

Closed #46 (eval: first-principles validation). Delivered three squashed commits to
`casehubio/eidos` main: DispositionAxis type safety (`jsonKey()`, `description()`,
typed AXES throughout eval), Phase 3 behavioral eval (`BehavioralJudge`,
`AgentProviderChatModel` bridge, `evaluateBehavioralScenarios()`), and docs/protocols
(PP-20260610-de090d behavioral-judge-blind, PP-20260610-70478e disposition-axis-string-boundary,
ADR-0005 pair-contrast methodology, CLAUDE.md, spec). The eval harness is now wired and
ready to run — the structural baseline and behavioral pair-contrast loop are implemented;
neither has been executed yet.

## Immediate Next Step

Wire credentials and run Phase 1:
```
claude config set model claude-sonnet-4-6
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -Peval -Dgroups=eval \
  -Dtest=PromptEvalTest#evaluateAllScenarios
```
Inspect `target/eval-report.json` → calibrate `SCORE_FLOORS` in `PromptEvalTest.java` →
commit the floor update + baseline artifact.

## Cross-Module

**We're blocking:** `casehub-engine` (#28) — Belbin-based agent composition. All eidos
deps satisfied; unblocked this session.

## What's Left

- `eidos#47` — eval test polish: loadIndex null-path test, inline ObjectMapper, blind-invariant assertion · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Phase 1: run evaluateAllScenarios(), calibrate SCORE_FLOORS | S | Low | Needs Claude CLI configured |
| — | Phase 2: run evaluateRealWorldScenarios(), inspect 3 reports | M | Low | Depends on Phase 1 floors |
| — | Phase 3: run evaluateBehavioralScenarios(), calibrate ACCURACY_FLOOR | M | Med | Depends on Phase 2; 6 data points |
| — | Phase 3 (Jlama): run with -Peval-jlama, compare reports | M | Med | Needs GGUF model file; may hit GE-20260423-878486 |
| #28 | casehub-engine: Belbin-based agent composition | L | High | Cross-repo; all eidos deps done |

## References

- Blog: `blog/2026-06-10-mdp01-the-evaluation-that-tests-itself.md`
- Spec: `specs/2026-06-09-eval-baseline-behavioral-design.md`
- ADR-0005: `adr/0005-pair-contrast-behavioral-evaluation.md`
- Protocols: `docs/protocols/eval/`
- Eval run command: see Immediate Next Step above
