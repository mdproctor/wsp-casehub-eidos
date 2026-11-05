# HANDOFF — eidos

## Last Session

Completed #229 (OrganizationalUnit model enrichment). Three additive model changes to org-api: obligations/permissions on RoleSpec, dissolutionDrainTimeoutMinutes on GroupPolicy, TopologyFamily enum + TopologyFamilyClassifier SPI. Wired through annotations, YAML, and docs. Full build green, issue closed. Queue advanced to #230.

## Immediate Next Step

`work continue` — #230 is active (GroupInstance wither pattern, S/Low). Add withState(), withTopology(), withRoleAdoptions() etc. to GroupInstance. Mechanical record convenience methods.

## Decisions Made

- D1: Obligations/permissions go directly on RoleSpec (not RolePolicy — different concern)
- D2: `long dissolutionDrainTimeoutMinutes` (not Duration — avoids Jackson JSR310 friction)
- D3: TopologyFamily classification via SPI, not GroupPolicy field (classification, not policy)
- Design spec: `specs/issue-229-org-model-enrichment/2026-11-05-org-model-enrichment-design.md`
- Decisions: `specs/issue-229-org-model-enrichment/decisions.md`

## Linked Work

- Master unification issue: casehubio/neocortex#1105 (eidos step done; neocortex and agency steps remain)
- agency#490 — delete duplicate prompt types, simplify CognitionAvatarAdapter
- eidos#243 — model-resolving assembler variant (parked — wait for consumer demand)

## References

- Plan: `plans/2026-11-05-org-model-enrichment.md`
- Society design spec: `~/claude/public/casehub/engine/specs/2026-11-02-self-organising-agent-society-design.md`

## Known Issues

- Work-end orchestrator merge_failed error despite content landing — needed manual recovery in previous session
- Work-end promotion deleted workspace blog files — may recur
- close.py cannot handle standalone branches outside the queue — issue-240 landed manually
