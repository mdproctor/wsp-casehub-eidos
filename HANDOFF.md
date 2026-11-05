# HANDOFF — eidos

## Last Session

Housekeeping session. Cleared stale pause stack (issue-213, already closed), advanced queue past completed #227 and closed #228. Verified build green on main. Scaffolded queue for #229 and #230.

## Immediate Next Step

`work start` — queue has #229 active, #230 queued.

### #229 — OrganizationalUnit model enrichment (M)

Four model additions to org-api (no runtime behaviour):

1. **Role obligations/permissions on RoleSpec** — `List<String>` norm identifiers for agency-core's NormDecision framework. Additive to existing RolePolicy (capability-based matching stays).
2. **Dissolution policy on GroupPolicy** — `Duration dissolutionDrainTimeout` (default 30min) + optional `DissolutionPolicy` record for stuck-work strategy.
3. **Topology transition governance** — needs design decision: TopologyFamily enum vs `Set<String> allowedTopologyTransitions` on GroupPolicy. Enum commits to a taxonomy (distributed/centralised/independent). Set is more conservative. Recommend brainstorming this before committing.
4. **Items 1 and 4 in the issue overlap** — both describe obligations/permissions on roles. The answer is "add to RoleSpec, leave RolePolicy alone" but the issue isn't crisp about it.

Downstream unblocks: OrgGoalCompiler enrichment, OrgPlanApprovalPolicy, OrgDriftPolicy refinement.

### #230 — GroupInstance wither pattern (S/Low)

Add withState(), withTopology(), withRoleAdoptions() etc. to GroupInstance. Reduces 15+ line record reconstructions to 1-2 lines. Mechanical.

## References

- Design spec: `wksp/specs/issue-216-self-organising-society/2026-11-02-self-organising-society-design.md`
- Decisions: `wksp/specs/issue-216-self-organising-society/decisions.md` (D1-D28)
- org-api source: `org-api/src/main/java/io/casehub/eidos/org/api/`

## Known Issues

- Work-end orchestrator merge_failed error despite content landing — needed manual recovery in previous session
- Work-end promotion deleted workspace blog files — may recur
