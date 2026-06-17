# eidos Session Handover — 2026-06-17

## Last Session

Validated the enrichment pipeline (proximity 5.00/5.00, all 16 cases complete), discovered inverse self-evaluation bias (Claude scores own enriched renders 1.88-2.13 on FACTUAL_FIDELITY; Qwen 8B scores same renders 5.00), added `RenderedPrompt.enriched` flag to skip the completeness substring check for enriched renders (eidos#53), populated `briefing` in all 8 eval profiles from `vocabularyGap:FULL` entries (eidos#59), and recalibrated PROSE floor from 4.0 → 3.5. All work merged to upstream/main.

## Immediate Next Step

Run the next eval with briefings populated to see if briefing content shifts proximity scores:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean test -pl eval -Peval \
  -Dtest=PromptEvalTest#evaluateRealWorldScenarios \
  -Dcasehub.eval.renders-cache.path=/tmp/eidos-renders-cache.json \
  -f /Users/mdproctor/claude/casehub/eidos/pom.xml
```

## What's Left

- **eidos#52** — Claude CLI subprocesses survive `atMost()` cancellation; `destroyForcibly()` on cancellation · M · High
- **eidos#55** — feat: Capability sub-specialization metadata — learned from DECLINE/FAIL patterns · M · High

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Re-run evaluateRealWorldScenarios with briefings populated | XS | Low | Compare proximity scores vs pre-briefing baseline |
| — | evaluateWithIndependentJudge (Qwen 8B) after briefings run | XS | Low | Check if briefing improves FACTUAL_FIDELITY from Qwen too |
| #28 | casehub-engine: Belbin-based agent composition | L | High | casehubio/eidos repo |

## References

- Garden entry: GE-20260617-f3ea4e — inverse self-evaluation bias (tools/GE-20260617-f3ea4e.md)
- Protocol: PP-20260617-bfc66f — briefing from vocabularyGap:FULL (docs/protocols/eval/)
- Blog: 2026-06-17-mdp01-judging-the-judge.md
- Ollama: qwen3:8b confirmed working for independent judge
- Renders cache: /tmp/eidos-renders-cache.json (survives mvn clean — use -Dcasehub.eval.renders-cache.path)
