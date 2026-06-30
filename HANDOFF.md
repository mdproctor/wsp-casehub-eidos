*Updated: parent#329 closed — removed from backlog.*

# eidos Session Handover — 2026-06-30

**Previous handover:** `git show HEAD~1:HANDOFF.md` | diff: `git diff HEAD~1 HEAD -- HANDOFF.md`

## What Changed This Session

- Shipped eidos#71 — semantic capability matching with XKOS-style subsumption in VocabularyRegistry. VocabularyTerm.specializes() for DAG hierarchy, MatchDegree (Exact/Plugin/Specialization/None) with depth, AgentCapability.capabilityVocabulary for optional grounding, subsumption-aware find() and probe(). CasehubCapabilityTerm starter vocabulary. 32 files, 2377 insertions.
- Design-reviewed via adversarial 5-round process (14 issues raised, 13 verified, 1 accepted). $15.59 review cost.
- Deep research on capability matching: OWLS-MX, XKOS, SkillNet, A2A — 105 agents, 23 sources, 21 confirmed claims.
- Filed eidos#72 (probe integration — done in-scope), eidos#73 (cross-vocab subsumption), eidos#74 (minor hardening), engine#609 (AgentCandidateFactory subsumption)

## Immediate Next Step

Re-run eval baseline to confirm quality after the subsumption additions:

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean test -pl eval -Peval \
  -Dtest=PromptEvalTest#evaluateRealWorldScenarios \
  -Dcasehub.eval.renders-cache.path=/tmp/eidos-renders-cache.json \
  -f /Users/mdproctor/claude/casehub/eidos/pom.xml
```

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #73 | Cross-vocabulary subsumption (app terms extending foundation terms) | M | Med | Enables domain-specific capability hierarchies |
| #74 | Minor hardening from code review (defensive copies, cycle error messages) | XS | Low | |
| engine#609 | AgentCandidateFactory subsumption matching | S | Low | Engine dispatch path — parallel to eidos registry path |
| — | Semantic capability matching (subsumption in VocabularyRegistry) | L | High | No issue filed yet; taxonomy design required |
| — | Behavioral contracts / runtime validation | L | High | No issue filed yet; bridges eidos and ledger |

## References

- Spec: `specs/2026-06-30-semantic-capability-matching-design.md` (promoted to project `docs/specs/`)
- Research: deep-research workflow output (105 agents, OWLS-MX/XKOS/SkillNet/A2A)
- Blog: `blog/2026-06-29-mdp01-backlog-zero.md` (previous session)
- Design review: `~/adr/eidos/semantic-capability-matching-*/tracker.md`
