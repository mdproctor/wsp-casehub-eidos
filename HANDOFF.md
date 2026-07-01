# eidos Session Handover — 2026-07-01

**Previous handover:** `git show HEAD~1:HANDOFF.md` | diff: `git diff HEAD~1 HEAD -- HANDOFF.md`

## What Changed This Session

- Closed eidos#75 — updated ARC42STORIES.MD with subsumption hierarchy and cross-vocabulary specialization documentation. Updated §1 description, §4 layer taxonomy, §5 building block view, L1 Zero-Dep API layer (MatchDegree, VocabularyTerm.specializes(), VocabularyRegistry subsumption API), L2 CDI Discovery layer (two-pass registration, global DAG, per-vocabulary index injection, collision detection, late register atomicity), and §13 Glossary (MatchDegree, VocabularyTerm, VocabularyRegistry entries).

## Immediate Next Step

Pick up #76 (learned exclusion tag semantics) — the most impactful remaining issue. Pre-existing design amplified by cross-vocab matching.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #76 | Learned exclusion tag semantics — probe() uses requested tag not declared capability | M | Med | Pre-existing design, amplified by cross-vocab matching |
| engine#609 | AgentCandidateFactory subsumption matching | S | Low | Engine dispatch path — parallel to eidos registry path |
| — | Behavioral contracts / runtime validation | L | High | No issue filed yet; bridges eidos and ledger |

## References

*Unchanged — `git show HEAD~1:HANDOFF.md`*
