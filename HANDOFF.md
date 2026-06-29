# eidos Session Handover — 2026-06-29

**Previous handover:** `git show HEAD~1:HANDOFF.md` | diff: `git diff HEAD~1 HEAD -- HANDOFF.md`

## What Changed This Session

- Closed eidos#28 (Belbin composition) — eidos prerequisites shipped; filed engine#577 for engine-side work
- Fixed eidos#69 — exhaustive switch in eval `AgentProviderChatModel` (sealed `AgentEvent` gained subtypes; `-pl` hid the error)
- Eidos backlog is now empty — all remaining work lives in other repos (engine#505, engine#577)
- Discussed roadmap: positive specialization learning, semantic capability matching, behavioral contracts

## Immediate Next Step

Re-run eval baseline — the command from the prior session is still valid:

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean test -pl eval -Peval \
  -Dtest=PromptEvalTest#evaluateRealWorldScenarios \
  -Dcasehub.eval.renders-cache.path=/tmp/eidos-renders-cache.json \
  -f /Users/mdproctor/claude/casehub/eidos/pom.xml
```

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Positive specialization learning (symmetric to DECLINE exclusions) | M | Med | No issue filed yet; builds on CapabilitySpecializationStore |
| — | Semantic capability matching (subsumption in VocabularyRegistry) | L | High | No issue filed yet; taxonomy design required |
| — | Behavioral contracts / runtime validation | L | High | No issue filed yet; bridges eidos and ledger |

## References

- Blog entry: `blog/2026-06-29-mdp01-backlog-zero.md`
- Garden: `GE-20260629-5d23ca` (Maven `-pl` skips test-compile on sealed interface subtypes)
