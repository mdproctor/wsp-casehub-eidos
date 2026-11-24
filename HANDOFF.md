# HANDOFF — eidos

## Last Session

Two issues worked: #272 landed (XS), #271 in progress (M/High, 1/6 tasks done).

### #272 — COMPLIANT TTL expiry test (landed)
- Added `compliant_expires_after_ttl` test to `InMemoryBehavioralSignalStoreTest`
- Fixed pre-existing gap: `violated_expires_after_ttl` was missing `learned()` assertion
- Fixed SVG text overlap in "When Confidence Lies" blog image (anti-gaming pipeline diagram)
- Squashed to 1 commit, landed on main as `125f799`, published

### #271 — RoleHealth probe (in progress, Task 1/6 done)

**Design phase complete:**
- 8 design decisions captured and reviewed (3 rounds of spec review)
- Spec: `specs/issue-271-role-health-probe/2026-11-24-role-health-probe-design.md`
- Decisions: `specs/issue-271-role-health-probe/decisions.md`
- Plan: `plans/2026-11-24-role-health-probe.md` (6 tasks, 2 batches)

**Key design decisions:**
- Two SPIs: `RoleHealth` (stateless fitness probe) + `RoleTransitionAdvisor` (stateful policy evaluation) — follows DispositionHealth/DispositionEvolution pattern
- Evidence from knowledge graph task outcomes (new `role` field on `AgentTask`), not a dedicated signal store
- Wilson lower-bound confidence (z=1.645), consistent with platform convention
- Data sufficiency gating: `DeclaredFit` for <5 outcomes (Wilson misleadingly low with sparse data)
- Hold-cycle staleness decay: reset counter when older than temporal window
- Mode.DISABLED added to RoleChangePolicy enum (replaces unreachable-threshold workaround)

**Task 1 complete:**
- `WilsonConfidence` utility extracted to api/ from JpaAgentGraphQuery (commit `990aa0b`)
- `Mode.DISABLED` added to `RoleChangePolicy.Mode` enum
- `DISABLED` constant updated from `Mode.ADVISORY + minConfidence=1.1` to `Mode.DISABLED + 0.0`

**Remaining tasks (Batch 1 continues, then Batch 2):**

| Task | Batch | Status |
|------|-------|--------|
| WilsonConfidence + Mode.DISABLED | API Foundation | Done |
| AgentTask role + historyByRole + schema | API Foundation | Next |
| RoleHealth + RoleHoldCycleStore + RoleTransitionAdvisor SPIs | API Foundation | Pending |
| DefaultRoleHealth + NoOps + CDI | Implementation + Wiring | Pending |
| InMemoryRoleHoldCycleStore + DefaultRoleTransitionAdvisor | Implementation + Wiring | Pending |
| Integration test | Implementation + Wiring | Pending |

### Issues closed
- **#257** — independence-aware attestation relevance factor (stale open, was already landed)
- **#258** — ARC42STORIES attestation relevance update (stale open, was already landed)
- **#272** — COMPLIANT TTL expiry test (landed this session)

### Blog
- "The Stateful Line" — where fitness probing ends and policy evaluation begins in RoleHealth design

### Follow-up issues filed (by spec review)
- **#273** — behavioral observation as supplementary evidence source
- **#274** — domain-scoped role fitness
- **#275** — weighted recency decay for temporal window
- **#276** — backport temporal filtering to existing AgentGraphQuery methods
- **#277** — DetectedRole/stigmergy tracking

## Active Work

| # | Title | Scale | Complexity | Status |
|---|-------|-------|------------|--------|
| #271 | RoleHealth probe | M | High | Task 1/6 done, on branch `issue-271-role-health-probe` |

## Notes for Next Session

- Start at Task 2: add `role` field to `AgentTask`, implement `historyByRole` in `JpaAgentGraphQuery`, create V4 migration
- The plan is at `plans/2026-11-24-role-health-probe.md` — read it, resume at Task 2
- `AgentTask` constructor change will break `AgentTaskEntity.from()/toRecord()` and any engine callers — fix all compilation errors
- `GraphDataSufficiency.forCount()` exists and returns INSUFFICIENT for <5 outcomes — use it in the probe
- `EidosCoreProducer` has 18 existing `@Produces @DefaultBean` methods — add 3 more for NoOps
- Quarkus augmentation cache can go stale across branches — `mvn clean install -DskipTests` fixes startup failures
