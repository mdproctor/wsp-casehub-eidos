# eidos Session Handover — 2026-06-23

## Last Session

No eidos implementation work. Session was CI support: casehub-ops CI was failing with "wrong SNAPSHOT" symptom — root cause was 401 on a secondary `github-casehubio` Maven repo entry (auth misconfiguration, not stale publish). eidos SNAPSHOT with `excludedDomains` was confirmed published at `casehub-eidos-api-0.2-20260620.024844-153`. engine-ai CI unblocked once eidos SNAPSHOT timing resolved (engine#541 one-line test fix, `Set.of()` as 9th arg).

## Immediate Next Step

*Unchanged — git show HEAD~1:HANDOFF.md*

*Updated: parent#281 closed — removed from backlog.*

## What's Left

- **eidos#61** — `AgentQuery.taskDomain` pre-filter at registry level · M · Med
- **eidos#62** — JPA-backed `CapabilitySpecializationStore` for production persistence · S · Low
- **eidos#63** — `AgentDescriptorMapper.toCapability()` must revert to positional constructor (protocol violation found in sweep) · XS · Low

## What's Next

*Unchanged — git show HEAD~1:HANDOFF.md*

## References

*Unchanged — git show HEAD~1:HANDOFF.md*
- Garden: GE-20260623-02d4f4 — CI failure blamed on stale SNAPSHOT, actual cause 401 on secondary Maven repo
