# eidos Session Handover — 2026-06-18

## Last Session

Fixed eidos#52: Claude CLI subprocesses accumulate when parallel `AgentProviderChatModel` enrichment calls all timeout simultaneously. Root cause was `buildEventStream()` using the SDK's session mode (bidirectional subprocess, 5-second graceful close) for single-turn `invoke()` calls. Built `ClaudeOneShotProcess` in casehub-platform: one-shot subprocess (`-p "prompt" --output-format stream-json`), direct `Process` ownership, immediate `destroyForcibly()` on cancellation. 22 unit tests, 4 garden entries. Branch closed; platform commit cc08daa on `issue-52-subprocess-destroy-on-cancel`.

## Immediate Next Step

Re-run `evaluateRealWorldScenarios` with briefings — the zombie accumulation that was hanging ProximityJudge is now fixed. Compare proximity scores against pre-briefing baseline:

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean test -pl eval -Peval \
  -Dtest=PromptEvalTest#evaluateRealWorldScenarios \
  -Dcasehub.eval.renders-cache.path=/tmp/eidos-renders-cache.json \
  -f /Users/mdproctor/claude/casehub/eidos/pom.xml
```

## What's Left

- **eidos#55** — feat: Capability sub-specialization metadata — learned from DECLINE/FAIL patterns · M · High
- **casehubio/parent#273** — sync PLATFORM.md and casehub-platform.md deep-dive for invoke() one-shot split

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Re-run `evaluateRealWorldScenarios` with briefings | XS | Low | Compare vs pre-briefing baseline; zombie fix now in place |
| — | `evaluateWithIndependentJudge` (Qwen 8B) after | XS | Low | Check if briefing lifts FACTUAL_FIDELITY |
| #28 | casehub-engine: Belbin-based agent composition | L | High | casehubio/eidos repo |

## References

- Platform commit: casehub-platform cc08daa — `ClaudeOneShotProcess` + `ClaudeAgentClient` changes
- Platform branch: `issue-52-subprocess-destroy-on-cancel` (open — needs PR to casehubio/platform)
- Spec: `docs/superpowers/specs/2026-06-17-subprocess-destroy-on-cancel-design.md`
- Garden entries: GE-20260618-03a482 (parse null), GE-20260618-a0f1e0 (--verbose), GE-20260618-c4f95a (close() blocks elastic), GE-20260618-268aab (cat subprocess test)
- parent#273: PLATFORM.md sync needed
- Renders cache: `/tmp/eidos-renders-cache.json` (survives `mvn clean`)
