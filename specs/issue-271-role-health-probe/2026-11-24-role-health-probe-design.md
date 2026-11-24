# RoleHealth Probe — Declared-vs-Operable Role Assessment

**Issue:** casehubio/eidos#271
**Date:** 2026-11-24
**Status:** Design

## Overview

A health probe layer for agent roles, paralleling CapabilityHealth for capabilities and DispositionHealth for disposition. Two SPIs: **RoleHealth** (stateless fitness probe) computes role confidence from knowledge graph task outcomes; **RoleTransitionAdvisor** (stateful policy evaluator) applies RoleChangePolicy thresholds and hold-cycle governance to recommend role transitions.

## Architecture

### Two-SPI design

The design separates fitness probing (read-only) from transition advisory (stateful), following the DispositionHealth / DispositionEvolution pattern.

| SPI | Responsibility | State | Parallel |
|-----|---------------|-------|----------|
| `RoleHealth` | Probe: "how fit is this agent for role X?" | Stateless — reads knowledge graph | CapabilityHealth, DispositionHealth |
| `RoleTransitionAdvisor` | Advise: "should this agent transition?" | Stateful — tracks hold-cycles | DispositionEvolution |

**Why separate:** `RoleChangePolicy.minHoldCycles` requires a persistent counter per (agentId, roleName) pair tracking consecutive below-threshold probe cycles. This makes transition advisory inherently stateful. Bundling it with the probe would make a read-only operation have side effects.

**Scope difference from DispositionEvolution:** `DispositionEvolution.evaluate()` takes only a specific status variant (`EvolutionPending`) because it only acts when evolution is pending — the caller pre-filters. `RoleTransitionAdvisor.evaluate()` takes any `RoleStatus` because the hold-cycle counter requires seeing all probe results: it must reset on `Fit`/`DeclaredFit` and increment on `Unfit`. Restricting to a single variant would leave counter management to the caller, breaking encapsulation.

### Evidence source: knowledge graph

Role fitness evidence comes from the existing knowledge graph (AgentTask / AgentGraphQuery). A new `role` field on AgentTask records which role the agent was serving when dispatched. The knowledge graph is the authoritative evidence source because:

- Task outcomes are **objective** (agent succeeded or failed) rather than subjective (someone recorded a signal)
- Outcomes carry **attestation chains** via AgentGraphStore.linkAttestation()
- The existing **Wilson lower-bound** scoring infrastructure is reused directly
- Recording happens through the **existing engine dispatch flow** — no new recording protocol

**Divergence from #271:** The issue identifies "Behavioral signals (from DetectedRole / stigmergy)" as an evidence source. These types do not exist in the codebase — they are part of the future role model redesign (casehubio/engine#1295). The probe cannot depend on infrastructure that hasn't been built. Task outcomes are used as the initial evidence source because they are available today through the knowledge graph. When DetectedRole and the stigmergy observation model are implemented, the probe's `DefaultRoleHealth` implementation can integrate them as supplementary evidence without changing the `RoleHealth` SPI. Filed as #277 for explicit tracking.

### Consumer

The primary consumer is the agent dispatch flow. `SimpleAgentSelector.filterHealthy()` already calls `CapabilityHealth.probe()` to gate dispatch candidates. `DefaultOrgAwareDispatch.escalate()` does the same for cross-unit dispatch. The role health probe slots into the same flow:

- **When:** At dispatch time, after capability health filtering, before scoring
- **Who calls:** `SimpleAgentSelector` (or its equivalent role-dispatch path in engine/) invokes `RoleHealth.probe()` for the target role, then optionally `RoleTransitionAdvisor.evaluate()` to determine transition recommendations
- **Lifecycle:** Each dispatch-time probe constitutes one evaluation cycle for hold-cycle governance

This follows the established pattern: SPIs defined in api/, consumed by dispatch logic in runtime/ and engine/. The consumer integration is in engine/ (casehubio/engine#1295) — this spec defines the eidos-side SPI contract.

## API Design

### RoleHealth SPI (api/)

```java
public interface RoleHealth {

    RoleStatus probe(AgentDescriptor descriptor, String roleName, ProbeContext context);

    sealed interface RoleStatus permits
            RoleStatus.DeclaredFit,
            RoleStatus.Fit,
            RoleStatus.Marginal,
            RoleStatus.Unfit,
            RoleStatus.Undeclared {

        String roleName();

        record DeclaredFit(String roleName) implements RoleStatus {}

        record Fit(String roleName, double confidence) implements RoleStatus {}

        record Marginal(String roleName, double confidence, int totalOutcomes)
                implements RoleStatus {}

        record Unfit(String roleName, double confidence, int failureCount,
                     int totalOutcomes) implements RoleStatus {}

        record Undeclared(String roleName) implements RoleStatus {}
    }
}
```

**Sealed result variants:**

| Variant | When | Confidence range |
|---------|------|-----------------|
| `DeclaredFit` | Role declared in RoleProfile but zero task outcomes with this role in the temporal window, OR data sufficiency is INSUFFICIENT (fewer than 5 outcomes) | n/a — no evidence or insufficient evidence |
| `Fit` | Wilson lower-bound confidence ≥ RoleChangePolicy.minConfidence | ≥ minConfidence |
| `Marginal` | Confidence between demotionConfidence and minConfidence (hysteresis zone) | [demotionConfidence, minConfidence) |
| `Unfit` | Wilson lower-bound confidence < RoleChangePolicy.demotionConfidence | < demotionConfidence |
| `Undeclared` | Role not in descriptor's RoleProfile.declaredRoles, or roleProfile is null | n/a |

**`roleName` on RoleStatus:** Every variant carries `roleName`, making the status object self-describing. This parallels `EvolutionPending` carrying `candidateFunction` and allows `RoleTransitionAdvisor.evaluate()` to take `(descriptor, roleStatus)` without a separate `roleName` parameter.

**Probe steps:**

1. If `descriptor.roleProfile()` is null → `Undeclared(roleName)` — an agent without a role profile has no declared roles
2. If `roleName` not in `roleProfile.declaredRoles()` → `Undeclared(roleName)`
3. Query knowledge graph for task outcomes where role = roleName within temporal window
4. Check data sufficiency via `GraphDataSufficiency`:
   - If `SufficiencyLevel.INSUFFICIENT` (fewer than 5 outcomes) → `DeclaredFit(roleName)` — treat as insufficient evidence rather than producing a misleadingly low Wilson score
5. Compute Wilson lower-bound from outcome distribution (SUCCEEDED=1.0, PARTIALLY=0.5, FAILED=0.0 quality multipliers, z=1.645 at 95%)
6. Compare confidence against RoleChangePolicy thresholds → `Fit`, `Marginal`, or `Unfit`

**Data sufficiency rationale:** Wilson lower-bound at z=1.645 is intentionally conservative — 1 success yields ~0.09, 2 successes yield ~0.21. These scores fall below any reasonable `minConfidence` threshold, so an agent with 1-2 successful tasks would be marked `Unfit`, which is counterintuitive and harmful. `GraphDataSufficiency.forCount()` returns `INSUFFICIENT` below 5 outcomes; the probe returns `DeclaredFit` in this case, paralleling how `CapabilityHealth` returns `Ready` for declared capabilities with no contrary evidence.

**Reuses `ProbeContext`** from CapabilityHealth for consistency — both `CapabilityHealth.probe()` and `DispositionHealth.probe()` accept `ProbeContext`. Following the established probe pattern.

### RoleTransitionAdvisor SPI (api/)

```java
public interface RoleTransitionAdvisor {

    TransitionAdvice evaluate(AgentDescriptor descriptor, RoleStatus currentStatus);

    sealed interface TransitionAdvice permits
            TransitionAdvice.Hold,
            TransitionAdvice.PromotionWarranted,
            TransitionAdvice.DemotionAdvised,
            TransitionAdvice.PolicyDisabled {

        record Hold(String roleName, double confidence,
                    RoleChangePolicy.Mode policyMode) implements TransitionAdvice {}

        record PromotionWarranted(String roleName, double confidence,
                                  RoleChangePolicy.Mode policyMode) implements TransitionAdvice {}

        record DemotionAdvised(String roleName, double confidence, int holdCyclesElapsed,
                               int holdCyclesRequired,
                               RoleChangePolicy.Mode policyMode) implements TransitionAdvice {}

        record PolicyDisabled() implements TransitionAdvice {}
    }
}
```

**Evaluation logic:**

1. Read RoleChangePolicy from descriptor's roleProfile → `PolicyDisabled` if `policy.mode() == Mode.DISABLED`
2. If `RoleStatus.Undeclared` → `Hold` (undeclared role, no transition recommendation)
3. If `RoleStatus.DeclaredFit` → reset hold-cycle counter → `Hold` (no evidence, no basis for transition)
4. If `RoleStatus.Fit`:
   - Reset hold-cycle counter
   - If policy mode is `AUTOMATIC` or `AUTO_ABOVE_THRESHOLD` → `PromotionWarranted` (evidence supports the role, policy permits automated transitions)
   - If policy mode is `ADVISORY` or `DISABLED` → `Hold` (advisory-only, no automated transition)
5. If `RoleStatus.Marginal` → `Hold` (in hysteresis zone, no action)
6. If `RoleStatus.Unfit` → increment hold-cycle counter:
   - holdCyclesElapsed < minHoldCycles → `Hold` (waiting)
   - holdCyclesElapsed ≥ minHoldCycles → `DemotionAdvised`

**PromotionWarranted semantics:** `PromotionWarranted` signals that the agent has demonstrated fitness for the role (above `minConfidence`) AND the governance policy permits automated transitions. `ADVISORY` mode never produces `PromotionWarranted` — in advisory mode, fitness is observed and reported but transitions require human action.

**Hold-cycle semantics:** A "cycle" is each invocation of `evaluate()`. The consumer determines the cadence — at dispatch time, the probe is called per-dispatch, so hold cycles accumulate with dispatch frequency. The counter tracks *consecutive* below-threshold evaluations: any `Fit` or `DeclaredFit` result resets the counter to zero. `minHoldCycles` governs how many consecutive `Unfit` evaluations must occur before demotion is advised, providing stability against transient poor performance.

**Prerequisite change — `RoleChangePolicy.Mode.DISABLED`:** The current `Mode` enum has only `ADVISORY`, `AUTO_ABOVE_THRESHOLD`, `AUTOMATIC`. The `DISABLED` constant uses `Mode.ADVISORY` with an unreachable `minConfidence` of 1.1 — a workaround. This spec requires adding `Mode.DISABLED` to the enum as a proper variant, following the platform convention of fixing the design rather than working around it. The `DISABLED` convenience constant then becomes `new RoleChangePolicy(Mode.DISABLED, 0.0, 0.0, 0)`.

**Hold-cycle state:** A `RoleHoldCycleStore` SPI tracks the per-(agentId, roleName) counter. CDI ladder: NoOp in eidos-core/ (plain class, no annotations), `@Produces @DefaultBean` producer in `EidosCoreProducer`; InMemory `@Alternative @Priority(1)` in persistence-memory/; JPA `@IfBuildProperty` in runtime/.

### AgentTask extension (api/)

Add an optional `role` field to the existing AgentTask record:

```java
public record AgentTask(
    String taskId,
    String agentId,
    String tenancyId,
    String capabilityTag,
    String taskDomain,
    String role,              // NEW — nullable, the role the agent served
    String externalRef,
    Instant startedAt,
    Instant endedAt
) { ... }
```

**Cross-repo coordination:** Adding `role` changes AgentTask's canonical constructor. This breaks `AgentTaskEntity.from()` and `toRecord()` in graph/, and any engine code (casehubio/engine) that constructs `AgentTask` for `AgentGraphStore.recordTask()`. The engine determines the role at dispatch time and passes it when recording the task. Existing tasks with `role = null` are handled correctly: the probe's query filters by `role = roleName`, so null-role tasks are excluded from role-specific history. This is correct behavior — tasks recorded before role tracking have no role attribution.

### AgentGraphQuery extension (api/)

Add role-scoped query methods:

```java
// In AgentGraphQuery
AgentTaskHistory historyByRole(String agentId, String roleName,
                               String tenancyId, Instant since);
```

The `since` parameter implements the temporal window (D3). Default window: 90 days, configurable via `casehub.eidos.role-health.window-days`.

**Temporal filtering asymmetry:** Existing methods `agentHistory()` and `historyByCapability()` return all history without temporal scoping. The `since` parameter is needed here because role health requires windowed evidence. This establishes a pattern that should be backported — filed as #276.

## CDI Wiring

NoOp classes are plain Java in eidos-core/ with no CDI annotations. The `@Produces @DefaultBean` wiring happens through producer methods in `EidosCoreProducer` (runtime/), following the established pattern used by `NoOpDispositionHealth`, `NoOpDispositionEvolution`, `NoOpAgentGraphQuery`, and all other NoOp fallbacks.

| Type | Module | Annotation |
|------|--------|------------|
| `RoleHealth` | api/ | SPI interface |
| `DefaultRoleHealth` | runtime/ | `@ApplicationScoped` |
| `NoOpRoleHealth` | eidos-core/ | Plain class — returns `DeclaredFit` for all |
| `NoOpRoleHealth` producer | runtime/ `EidosCoreProducer` | `@Produces @DefaultBean` |
| `RoleTransitionAdvisor` | api/ | SPI interface |
| `DefaultRoleTransitionAdvisor` | runtime/ | `@ApplicationScoped` |
| `NoOpRoleTransitionAdvisor` | eidos-core/ | Plain class — returns `PolicyDisabled` |
| `NoOpRoleTransitionAdvisor` producer | runtime/ `EidosCoreProducer` | `@Produces @DefaultBean` |
| `RoleHoldCycleStore` | api/ | SPI interface |
| `NoOpRoleHoldCycleStore` | eidos-core/ | Plain class |
| `NoOpRoleHoldCycleStore` producer | runtime/ `EidosCoreProducer` | `@Produces @DefaultBean` |
| `InMemoryRoleHoldCycleStore` | persistence-memory/ | `@Alternative @Priority(1)` |
| `JpaRoleHoldCycleStore` | runtime/ | `@IfBuildProperty` blocking-mode |

## Confidence Computation

Wilson lower-bound at 95% confidence (z = 1.645), matching `AgentGraphQuery.topAgentsByOutcome()`:

```
quality = Σ (outcome.confidence × multiplier(outcome.result))
  where multiplier: SUCCEEDED=1.0, PARTIALLY=0.5, FAILED=0.0

n = count of outcomes
p̂ = quality / n

wilson = (p̂ + z²/2n - z√(p̂(1-p̂)/n + z²/4n²)) / (1 + z²/n)
```

**Wilson utility extraction:** The `wilsonScore` method is currently package-private in `JpaAgentGraphQuery`. Extract to a public `WilsonConfidence` utility class in `api/` — the computation is pure arithmetic with no JPA dependency. This enables the probe's unit tests to verify against the same algorithm used by `topAgentsByOutcome`, and makes the Wilson computation available to any module without depending on graph/.

**Data sufficiency gate:** Before computing Wilson, check `GraphDataSufficiency.forCount(n, ...)`:
- `SufficiencyLevel.INSUFFICIENT` (n < 5): return `DeclaredFit` — insufficient data for meaningful confidence scoring
- `SufficiencyLevel.INDICATIVE` (5 ≤ n < 10) or `SUFFICIENT` (n ≥ 10): compute Wilson score normally

**Zero evidence:** Returns `DeclaredFit` rather than a numeric score. This parallels CapabilityHealth.Ready — a declared capability with no evidence is assumed fit until proven otherwise.

## Temporal Window

Task outcomes are filtered at query time via `Instant since = Instant.now().minus(windowDays, ChronoUnit.DAYS)`. Knowledge graph records are permanent (audit trail); only the probe's view is temporal.

**Configuration:** `casehub.eidos.role-health.window-days` (default: 90).

**Trade-off:** A hard window (day 89 counts fully, day 91 drops off entirely) vs. gradual decay. The hard window is simpler and sufficient for the initial design. Weighted recency can be introduced later — filed as #275.

## Schema Changes

### V4__role_health.sql (graph/)

```sql
-- Column width matches AgentDescriptorValidator.MAX_ROLE_NAME (100)
ALTER TABLE agent_task ADD COLUMN role VARCHAR(100);
CREATE INDEX idx_agent_task_role ON agent_task (agent_id, tenancy_id, role);

CREATE TABLE role_hold_cycle (
    agent_id VARCHAR(255) NOT NULL,
    tenancy_id VARCHAR(255) NOT NULL,
    role_name VARCHAR(100) NOT NULL,  -- matches AgentDescriptorValidator.MAX_ROLE_NAME
    consecutive_cycles INT NOT NULL DEFAULT 0,
    last_evaluated_at TIMESTAMP,
    PRIMARY KEY (agent_id, tenancy_id, role_name)
);
```

**Module placement:** graph/ — the `agent_task` table is owned by graph/ (V3__agent_graph.sql), and the `role_hold_cycle` table stores knowledge-graph-adjacent state. Both tables in one migration file for atomic deployment.

## Testing Strategy

| Test | Type | What it verifies |
|------|------|-----------------|
| `RoleHealthTest` | Unit | Probe step logic: DeclaredFit for zero evidence, DeclaredFit for insufficient data (< 5 outcomes), Fit/Marginal/Unfit thresholds, Undeclared for missing role, Undeclared for null roleProfile |
| `WilsonConfidenceTest` | Unit | Wilson computation with known inputs; edge cases (1 outcome, all failures, mixed). Shared utility in api/ |
| `RoleTransitionAdvisorTest` | Unit | Hold-cycle accumulation, reset on recovery, PolicyDisabled for DISABLED mode, PromotionWarranted for AUTOMATIC/AUTO_ABOVE_THRESHOLD modes, Hold for ADVISORY mode, hysteresis zone |
| `InMemoryRoleHoldCycleStoreTest` | Unit | Store contract: increment, reset, concurrent access |
| `RoleHealthIntegrationTest` | `@QuarkusTest` | End-to-end: record tasks with roles, probe, verify confidence matches Wilson expectation |

## Known Limitations

- **Task-level granularity:** Outcomes measure task success, not role-specific performance within a task. Filed as #273.
- **No domain-scoping:** The probe computes aggregate confidence across all domains. Filed as #274.
- **Hard temporal window:** No gradual decay — outcomes contribute equally within the window and drop off entirely outside it. Filed as #275.

## References

- `api/CapabilityHealth.java` — primary probe pattern
- `api/DispositionHealth.java` / `api/DispositionEvolution.java` — probe/evolution SPI split
- `api/RoleProfile.java` / `api/RoleChangePolicy.java` — role model, threshold governance
- `api/AgentTask.java` / `api/AgentGraphQuery.java` — knowledge graph, Wilson scoring
- `api/AgentOutcome.java` / `api/TaskResult.java` — outcome model (SUCCEEDED/PARTIALLY/FAILED)
- `api/GraphDataSufficiency.java` — data sufficiency gating
- `runtime/EidosCoreProducer.java` — CDI @Produces @DefaultBean pattern
- ARC42STORIES.MD §1 — quality goals, Wilson lower-bound convention
- casehubio/engine#1295 — role model redesign context
