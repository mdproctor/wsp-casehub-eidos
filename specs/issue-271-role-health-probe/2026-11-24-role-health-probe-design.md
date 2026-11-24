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

### Evidence source: knowledge graph

Role fitness evidence comes from the existing knowledge graph (AgentTask / AgentGraphQuery). A new `role` field on AgentTask records which role the agent was serving when dispatched. The knowledge graph is the authoritative evidence source because:

- Task outcomes are **objective** (agent succeeded or failed) rather than subjective (someone recorded a signal)
- Outcomes carry **attestation chains** via AgentGraphStore.linkAttestation()
- The existing **Wilson lower-bound** scoring infrastructure is reused directly
- Recording happens through the **existing engine dispatch flow** — no new recording protocol

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

        record DeclaredFit() implements RoleStatus {}

        record Fit(double confidence) implements RoleStatus {}

        record Marginal(double confidence, int totalOutcomes) implements RoleStatus {}

        record Unfit(double confidence, int failureCount, int totalOutcomes)
                implements RoleStatus {}

        record Undeclared(String roleName) implements RoleStatus {}
    }
}
```

**Sealed result variants:**

| Variant | When | Confidence range |
|---------|------|-----------------|
| `DeclaredFit` | Role declared in RoleProfile but zero task outcomes with this role in the temporal window | n/a — no evidence |
| `Fit` | Wilson lower-bound confidence ≥ RoleChangePolicy.minConfidence | ≥ minConfidence |
| `Marginal` | Confidence between demotionConfidence and minConfidence (hysteresis zone) | [demotionConfidence, minConfidence) |
| `Unfit` | Wilson lower-bound confidence < RoleChangePolicy.demotionConfidence | < demotionConfidence |
| `Undeclared` | Role not in descriptor's RoleProfile.declaredRoles | n/a |

**Probe steps:**

1. Check descriptor's RoleProfile for roleName → `Undeclared` if absent
2. Query knowledge graph for task outcomes where role = roleName within temporal window
3. If zero outcomes → `DeclaredFit` (like CapabilityHealth.Ready for declared capabilities)
4. Compute Wilson lower-bound from outcome distribution (SUCCEEDED=1.0, PARTIALLY=0.5, FAILED=0.0 quality multipliers, z=1.645 at 95%)
5. Compare confidence against RoleChangePolicy thresholds → `Fit`, `Marginal`, or `Unfit`

**Reuses `ProbeContext`** from CapabilityHealth for consistency. `ProbeContext.taskDomain` is available for future domain-scoped filtering but not used in the initial implementation — role fitness is aggregate across domains.

### RoleTransitionAdvisor SPI (api/)

```java
public interface RoleTransitionAdvisor {

    TransitionAdvice evaluate(AgentDescriptor descriptor, String roleName,
                              RoleStatus currentStatus);

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

1. Read RoleChangePolicy from descriptor → `PolicyDisabled` if DISABLED
2. If `RoleStatus.DeclaredFit` or `Fit` → reset hold-cycle counter → `Hold` or `PromotionWarranted`
3. If `Marginal` → `Hold` (in hysteresis zone, no action)
4. If `Unfit` → increment hold-cycle counter:
   - holdCyclesElapsed < minHoldCycles → `Hold` (waiting)
   - holdCyclesElapsed ≥ minHoldCycles → `DemotionAdvised`

**Hold-cycle state:** A `RoleHoldCycleStore` SPI tracks the per-(agentId, roleName) counter. CDI ladder: NoOp `@DefaultBean`, InMemory `@Alternative`, JPA `@IfBuildProperty`.

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

### AgentGraphQuery extension (api/)

Add role-scoped query methods:

```java
// In AgentGraphQuery
AgentTaskHistory historyByRole(String agentId, String roleName,
                               String tenancyId, Instant since);
```

The `since` parameter implements the temporal window (D3). Default window: 90 days, configurable via `casehub.eidos.role-health.window-days`.

## CDI Wiring

| Type | Module | Annotation |
|------|--------|------------|
| `RoleHealth` | api/ | SPI interface |
| `DefaultRoleHealth` | runtime/ | `@ApplicationScoped` |
| `NoOpRoleHealth` | runtime/ (eidos-core) | `@DefaultBean` — returns `DeclaredFit` for all |
| `RoleTransitionAdvisor` | api/ | SPI interface |
| `DefaultRoleTransitionAdvisor` | runtime/ | `@ApplicationScoped` |
| `NoOpRoleTransitionAdvisor` | runtime/ (eidos-core) | `@DefaultBean` — returns `PolicyDisabled` |
| `RoleHoldCycleStore` | api/ | SPI interface |
| `NoOpRoleHoldCycleStore` | runtime/ (eidos-core) | `@DefaultBean` |
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

This reuses the existing `wilsonScore` static method in `JpaAgentGraphQuery` (or extracts it to a shared utility if currently private).

**Zero evidence:** Returns `DeclaredFit` rather than a numeric score. This parallels CapabilityHealth.Ready — a declared capability with no evidence is assumed fit until proven otherwise. The first evidence pushes the Wilson score conservatively low (by design), creating natural evidence pressure.

## Temporal Window

Task outcomes are filtered at query time via `Instant since = Instant.now().minus(windowDays, ChronoUnit.DAYS)`. Knowledge graph records are permanent (audit trail); only the probe's view is temporal.

**Configuration:** `casehub.eidos.role-health.window-days` (default: 90).

**Trade-off:** A hard window (day 89 counts fully, day 91 drops off entirely) vs. gradual decay. The hard window is simpler and sufficient for the initial design. Weighted recency can be introduced later if needed.

## Schema Changes

### V-next migration

```sql
ALTER TABLE agent_task ADD COLUMN role VARCHAR(100);
CREATE INDEX idx_agent_task_role ON agent_task (agent_id, tenancy_id, role);

CREATE TABLE role_hold_cycle (
    agent_id VARCHAR(255) NOT NULL,
    tenancy_id VARCHAR(255) NOT NULL,
    role_name VARCHAR(100) NOT NULL,
    consecutive_cycles INT NOT NULL DEFAULT 0,
    last_evaluated_at TIMESTAMP,
    PRIMARY KEY (agent_id, tenancy_id, role_name)
);
```

## Testing Strategy

| Test | Type | What it verifies |
|------|------|-----------------|
| `RoleHealthTest` | Unit | Probe step logic: DeclaredFit for zero evidence, Fit/Marginal/Unfit thresholds, Undeclared for missing role |
| `WilsonRoleConfidenceTest` | Unit | Wilson computation with known inputs; edge cases (1 outcome, all failures, mixed) |
| `RoleTransitionAdvisorTest` | Unit | Hold-cycle accumulation, reset on recovery, PolicyDisabled for DISABLED policy, hysteresis zone |
| `InMemoryRoleHoldCycleStoreTest` | Unit | Store contract: increment, reset, concurrent access |
| `RoleHealthIntegrationTest` | `@QuarkusTest` | End-to-end: record tasks with roles, probe, verify confidence matches Wilson expectation |

## Known Limitations

- **Task-level granularity:** Outcomes measure task success, not role-specific performance within a task. An agent can succeed at a task while performing poorly in coordination. Future work may add behavioral observation as supplementary evidence.
- **No domain-scoping:** The probe computes aggregate confidence across all domains. Domain-specific role fitness (e.g., "good coordinator for Java projects") is available through the knowledge graph's taskDomain field but not surfaced in the probe API initially.
- **Hard temporal window:** No gradual decay — outcomes contribute equally within the window and drop off entirely outside it.

## References

- `api/CapabilityHealth.java` — primary probe pattern
- `api/DispositionHealth.java` / `api/DispositionEvolution.java` — probe/evolution SPI split
- `api/RoleProfile.java` / `api/RoleChangePolicy.java` — role model, threshold governance
- `api/AgentTask.java` / `api/AgentGraphQuery.java` — knowledge graph, Wilson scoring
- `api/AgentOutcome.java` / `api/TaskResult.java` — outcome model (SUCCEEDED/PARTIALLY/FAILED)
- ARC42STORIES.MD §1 — quality goals, Wilson lower-bound convention
- casehubio/engine#1295 — role model redesign context
