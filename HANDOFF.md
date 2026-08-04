# eidos Session Handover - 2026-08-04

**Previous handover:** git show HEAD~1:HANDOFF.md

## What Changed This Session

- Closed eidos#135 — added `List<String> capabilities` to `AgentGoal` record for goal-capability mapping. Landed as 3d47268 on main.
- `AgentGoal` now has 5 fields: name, description, priority, visibility, capabilities. Null→empty, null-element filtering, duplicate rejection.
- Cross-validation in `AgentDescriptor` compact constructor: every capability name in a goal's list must match a declared `AgentCapability.name()` on the same descriptor.
- YAML parsing (`GoalConfig.capabilities`), JPA persistence (JSON column in `agent_goal`), `DefaultGoalEvolution` carries capabilities through promotion/demotion, `AgentDescriptorComparator` updated.
- V8 migration updated with `capabilities TEXT` column (no deployed instances — direct base file edit).
- CLAUDE.md and consumer-guide.md updated with new field documentation.
- Design spec: docs/specs/issue-135-goal-capability-mapping/2026-08-04-goal-capability-mapping-design.md

## Immediate Next Step

engine#860 depends on this — `GoalFailureRecorder` needs updating to filter goals by `goal.capabilities().contains(failedCapabilityName)` before recording DECLINE signals. Engine construction sites (`GoalAbandonmentEvaluator`, `GoalFailureRecorderTest`, `GoalAbandonmentEvaluatorTest`, `GoalSignalProviderTest`, `AgentGoalCompletionMarkerTest`) all need the 5th arg added.

## What is Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #126 | FunctionActivationJudge Ni/Ne calibration | M | Med | Judge accuracy |
| #78 | Example: learned specialization lifecycle | M | Med | Epic #82 |
| #80 | Example: full probe pipeline | M | Med | Epic #82 |
| #81 | Example: cost-aware multi-agent routing | M | Med | Epic #82 |
| #118 | Refactor: move API coverage tests out of examples | S | Low | |

## References

- Design spec: docs/specs/issue-135-goal-capability-mapping/2026-08-04-goal-capability-mapping-design.md
- Downstream: casehubio/engine#860 (GoalFailureRecorder per-goal discrimination)
