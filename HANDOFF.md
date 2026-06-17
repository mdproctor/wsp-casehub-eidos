# eidos Session Handover — 2026-06-17

## Last Session

Discovered ARC42STORIES.MD was workspace-only (routing table in CLAUDE.md never mentioned it). Fixed the routing gap, rescued 2 specs that existed only on closed workspace branches, reconstituted ARC42STORIES.MD for all post-C6 changes (L2 vocab enum redesign, L5 capability signal discrimination/enrichment mechanics/briefing/enriched flag, L6 eval improvements, 6 new glossary entries), and promoted the updated document to the project repo. Confirmed all 27 eidos blogs are already published.

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
| — | Re-run `evaluateRealWorldScenarios` with briefings | XS | Low | Compare vs pre-briefing baseline |
| — | `evaluateWithIndependentJudge` (Qwen 8B) after | XS | Low | Check if briefing lifts FACTUAL_FIDELITY |
| #28 | casehub-engine: Belbin-based agent composition | L | High | casehubio/eidos repo |

## References

- Garden entry: GE-20260617-f3ea4e — inverse self-evaluation bias (tools/GE-20260617-f3ea4e.md)
- Protocol: PP-20260617-bfc66f — briefing from vocabularyGap:FULL (docs/protocols/eval/)
- Blog: 2026-06-17-mdp01-judging-the-judge.md
- Ollama: qwen3:8b confirmed working for independent judge
- Renders cache: /tmp/eidos-renders-cache.json (survives mvn clean — use -Dcasehub.eval.renders-cache.path)
- ARC42STORIES.MD: now in project repo root (promoted this session); covers all 6 chapters through Knowledge Graph
