# eidos Session Handover — 2026-06-30

**Previous handover:** `git show HEAD~1:HANDOFF.md` | diff: `git diff HEAD~1 HEAD -- HANDOFF.md`

## What Changed This Session

- Shipped eidos#70 — bidirectional `CapabilitySpecializationStore` with `SpecializationSignal { DECLINE, SUCCESS }`. Signal-parameterized API, per-signal TTL, V5 schema migration. All 774 tests pass.
- Design decision: store captures evidence, consumption is strategy — `DefaultCapabilityHealth.probe()` unchanged (DECLINE-only); positive evidence consumption left to domain-specific implementations
- Filed parent#329 for PLATFORM.md + casehub-eidos.md doc sync

## Immediate Next Step

Re-run eval baseline to confirm quality after the specialization store refactoring:

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean test -pl eval -Peval \
  -Dtest=PromptEvalTest#evaluateRealWorldScenarios \
  -Dcasehub.eval.renders-cache.path=/tmp/eidos-renders-cache.json \
  -f /Users/mdproctor/claude/casehub/eidos/pom.xml
```

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Semantic capability matching (subsumption in VocabularyRegistry) | L | High | No issue filed yet; taxonomy design required |
| — | Behavioral contracts / runtime validation | L | High | No issue filed yet; bridges eidos and ledger |

## References

- Spec: `specs/2026-06-30-positive-specialization-learning-design.md`
- Blog: `blog/2026-06-29-mdp01-backlog-zero.md`
- Garden: `GE-20260629-5d23ca` (Maven `-pl` skips test-compile on sealed interface subtypes)
