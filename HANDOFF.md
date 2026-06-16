# eidos Session Handover — 2026-06-17

## Last Session

Delivered three interdependent issues in one session: #56 (enrichment-mechanics), #57 (briefing-field), #58 (proximity-eval-redesign). All merged to `upstream/main` as 4 squashed commits. Evaluation infrastructure also stabilised (#54 eval judge resilience, already on main before this session). The enrichment pipeline now actually runs (was silently falling back to structural on every call), disposition prose is role-specific and JSON-resilient, and the proximity eval measures the right thing.

## Immediate Next Step

Run `evaluateRealWorldScenarios` with Claude to confirm enrichment success rate >80% (was ~0% before). Use the configurable cache path:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean test -pl eval -Peval \
  -Dtest=PromptEvalTest#evaluateRealWorldScenarios \
  -Dcasehub.eval.renders-cache.path=/tmp/eidos-renders-cache.json \
  -f /Users/mdproctor/claude/casehub/eidos/pom.xml
```

## What's Left

- **eidos#52** — Claude CLI subprocesses survive `atMost()` cancellation; `destroyForcibly()` on cancellation · M · High
- **eidos#53** — `allCasesComplete()` fails for LLM-enriched renders (capability names paraphrased) · S · Med
- **parent#245** — casehub-eidos.md Current State: add eidos#54–#58 complete · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Validate enrichment success rate with evaluateRealWorldScenarios | XS | Low | See Immediate Next Step |
| — | evaluateWithIndependentJudge (Qwen 8B) to check for self-eval bias on enriched renders | XS | Low | Cache survives at /tmp/eidos-renders-cache.json |
| #28 | casehub-engine: Belbin-based agent composition | L | High | casehubio/eidos repo |
| — | Populate `briefing` in eval profiles from vocabularyGap: FULL entries | XS | Low | Then re-run proximity eval with redesigned ProximityJudge |

## References

- Spec: `docs/superpowers/specs/2026-06-16-enrichment-rethink-design.md` (workspace)
- Plan: `docs/superpowers/plans/2026-06-16-enrichment-rethink.md` (workspace)
- Squash plan: `docs/superpowers/specs/squash-plan-2026-06-17.md` (project)
- Commits: d06cb98 (#56), 6946396 (#57), b5080ee (#58), 525ddd4 (#54)
- Ollama: qwen3:8b confirmed working for independent judge (Qwen 35B OOMs this machine)
- Renders cache: /tmp/eidos-renders-cache.json (survives mvn clean — use -Dcasehub.eval.renders-cache.path)
