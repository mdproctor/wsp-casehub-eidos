# eidos Session Handover — 2026-07-01

**Previous handover:** `git show HEAD~1:HANDOFF.md` | diff: `git diff HEAD~1 HEAD -- HANDOFF.md`

## What Changed This Session

- Closed eidos#73 — cross-vocabulary subsumption. `VocabularyTerm.specializes()` now crosses vocabulary boundaries. `CdiVocabularyRegistry` refactored to two-pass registration (registerTerms → buildAllHierarchyIndexes), global DAG construction, per-vocabulary index injection for bidirectional `match()`, inline collision detection (native-vs-injected, injected-vs-injected), `expandForMatchingByVocabulary()` cross-vocab grouping by declaring vocabulary URI. Late `register()` atomicity with snapshot-rollback. No API signature changes. 4 files, +1261 -102.
- Filed eidos#75 (ARC42STORIES.MD subsumption docs) and eidos#76 (learned exclusion tag semantics) — out-of-scope items from design review.

## Immediate Next Step

Run eval baseline to confirm quality after subsumption additions:

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean test -pl eval -Peval \
  -Dtest=PromptEvalTest#evaluateRealWorldScenarios \
  -Dcasehub.eval.renders-cache.path=/tmp/eidos-renders-cache.json \
  -f /Users/mdproctor/claude/casehub/eidos/pom.xml
```

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #75 | Update ARC42STORIES.MD with subsumption hierarchy and cross-vocabulary specialization | S | Low | Documentation only |
| #76 | Learned exclusion tag semantics — probe() uses requested tag not declared capability | M | Med | Pre-existing design, amplified by cross-vocab matching |
| engine#609 | AgentCandidateFactory subsumption matching | S | Low | Engine dispatch path — parallel to eidos registry path |
| — | Behavioral contracts / runtime validation | L | High | No issue filed yet; bridges eidos and ledger |

## References

*Unchanged — `git show HEAD~1:HANDOFF.md`*
