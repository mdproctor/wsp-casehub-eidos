# RoleHealth Probe Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #271 — feat: RoleHealth probe — declared-vs-operable role assessment
**Issue group:** #271

**Goal:** Add a health probe layer for agent roles — RoleHealth (stateless fitness) and RoleTransitionAdvisor (stateful policy evaluation) — using knowledge graph task outcomes and Wilson lower-bound scoring.

**Architecture:** Two SPIs in api/ following the DispositionHealth/DispositionEvolution pattern. RoleHealth reads task outcomes from AgentGraphQuery filtered by a new `role` field on AgentTask, computes Wilson lower-bound confidence, and returns a sealed RoleStatus. RoleTransitionAdvisor evaluates RoleChangePolicy thresholds with hold-cycle governance via RoleHoldCycleStore.

**Tech Stack:** Java 21, Quarkus 3.32.2, CDI, JPA/Hibernate, Flyway

## Global Constraints

- Wilson z=1.645 (95% confidence) — matches `JpaAgentGraphQuery.topAgentsByOutcome()`
- Quality multipliers: SUCCEEDED=1.0, PARTIALLY=0.5, FAILED=0.0
- Data sufficiency threshold: 5 outcomes (via `GraphDataSufficiency`)
- Temporal window default: 90 days (`casehub.eidos.role-health.window-days`)
- Role name max length: 100 (`AgentDescriptorValidator.MAX_ROLE_NAME`)
- Max declared roles: 10 (`AgentDescriptorValidator.MAX_ROLE_DECLARATIONS`)
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
- NoOp pattern: plain class in `eidos-core/`, `@Produces @DefaultBean` in `EidosCoreProducer`

---

## Batch 1: API Foundation

### Task 1: Extract WilsonConfidence utility + add Mode.DISABLED

**Files:**
- Create: `api/src/main/java/io/casehub/eidos/api/WilsonConfidence.java`
- Create: `api/src/test/java/io/casehub/eidos/api/WilsonConfidenceTest.java`
- Modify: `graph/src/main/java/io/casehub/eidos/graph/JpaAgentGraphQuery.java` — delegate to WilsonConfidence
- Modify: `api/src/main/java/io/casehub/eidos/api/RoleChangePolicy.java` — add Mode.DISABLED
- Modify: `api/src/test/java/io/casehub/eidos/api/RoleChangePolicyTest.java` — update disabled test

**Interfaces:**
- Produces: `WilsonConfidence.score(double sumQuality, int n) → double` (static utility, z=1.645)
- Produces: `RoleChangePolicy.Mode.DISABLED` enum value

- [ ] **Step 1: Write WilsonConfidence failing test**

```java
package io.casehub.eidos.api;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class WilsonConfidenceTest {

    @Test
    void zero_observations_returns_zero() {
        assertThat(WilsonConfidence.score(0.0, 0)).isEqualTo(0.0);
    }

    @Test
    void single_success_conservative_score() {
        double score = WilsonConfidence.score(1.0, 1);
        assertThat(score).isBetween(0.05, 0.15);
    }

    @Test
    void ten_successes_higher_confidence() {
        double score = WilsonConfidence.score(10.0, 10);
        assertThat(score).isGreaterThan(0.70);
    }

    @Test
    void all_failures_near_zero() {
        double score = WilsonConfidence.score(0.0, 10);
        assertThat(score).isLessThan(0.05);
    }

    @Test
    void mixed_outcomes_intermediate() {
        // 7 succeeded (7.0), 1 partial (0.5), 2 failed (0.0) = quality 7.5 / 10
        double score = WilsonConfidence.score(7.5, 10);
        assertThat(score).isBetween(0.45, 0.80);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=WilsonConfidenceTest`
Expected: FAIL — WilsonConfidence class not found

- [ ] **Step 3: Implement WilsonConfidence**

Use `ide_create_file`:
```java
package io.casehub.eidos.api;

public final class WilsonConfidence {

    private static final double Z = 1.645;

    private WilsonConfidence() {}

    public static double score(final double sumQuality, final int n) {
        if (n == 0) return 0.0;
        final double p = sumQuality / n;
        final double z2 = Z * Z;
        final double denominator = 1.0 + z2 / n;
        final double centre = p + z2 / (2.0 * n);
        final double spread = Z * Math.sqrt((p * (1.0 - p) + z2 / (4.0 * n)) / n);
        return Math.max(0.0, (centre - spread) / denominator);
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=WilsonConfidenceTest`
Expected: PASS

- [ ] **Step 5: Delegate JpaAgentGraphQuery.wilsonScore to WilsonConfidence**

In `JpaAgentGraphQuery.java`, use `ide_replace_member` on `wilsonScore`:
```java
return WilsonConfidence.score(sumQuality, n);
```

- [ ] **Step 6: Add Mode.DISABLED to RoleChangePolicy**

Use `ide_edit_member` on the `Mode` enum in `RoleChangePolicy.java`:
```java
public enum Mode { ADVISORY, AUTO_ABOVE_THRESHOLD, AUTOMATIC, DISABLED }
```

Update the compact constructor validation — use `ide_replace_member` on the compact constructor to change the hysteresis guard from `mode != Mode.ADVISORY` to `mode != Mode.ADVISORY && mode != Mode.DISABLED`:

Update the `DISABLED` constant — use `ide_edit_member` on `DISABLED` field:
```java
public static final RoleChangePolicy DISABLED = new RoleChangePolicy(Mode.DISABLED, 0.0, 0.0, 0);
```

- [ ] **Step 7: Update RoleChangePolicyTest**

Use `ide_replace_member` on `disabled_constant_has_unreachable_threshold` (or the equivalent disabled test method) to assert mode instead of threshold:
```java
assertThat(RoleChangePolicy.DISABLED.mode()).isEqualTo(RoleChangePolicy.Mode.DISABLED);
assertThat(RoleChangePolicy.DISABLED.minConfidence()).isEqualTo(0.0);
assertThat(RoleChangePolicy.DISABLED.demotionConfidence()).isEqualTo(0.0);
assertThat(RoleChangePolicy.DISABLED.minHoldCycles()).isEqualTo(0);
```

- [ ] **Step 8: Run all api + graph tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api,graph`
Expected: PASS — all existing tests plus new WilsonConfidenceTest

- [ ] **Step 9: Commit**

```bash
git add api/ graph/
git commit -m "feat(#271): extract WilsonConfidence utility, add Mode.DISABLED to RoleChangePolicy

Refs #271"
```

---

### Task 2: Add `role` field to AgentTask + historyByRole query + schema

**Files:**
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentTask.java` — add `role` field
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentGraphQuery.java` — add `historyByRole`
- Modify: `graph/src/main/java/io/casehub/eidos/graph/entity/AgentTaskEntity.java` — add `role` column + update from()/toRecord()
- Modify: `graph/src/main/java/io/casehub/eidos/graph/JpaAgentGraphQuery.java` — implement `historyByRole`
- Create: `graph/src/main/resources/db/migration/V4__role_health.sql`
- Create: `api/src/test/java/io/casehub/eidos/api/AgentTaskRoleTest.java`

**Interfaces:**
- Consumes: `AgentTask` existing record (api/)
- Produces: `AgentTask.role()` — nullable String field
- Produces: `AgentGraphQuery.historyByRole(agentId, roleName, tenancyId, since) → AgentTaskHistory`

- [ ] **Step 1: Write failing test for AgentTask role field**

```java
package io.casehub.eidos.api;

import org.junit.jupiter.api.Test;
import java.time.Instant;
import static org.assertj.core.api.Assertions.assertThat;

class AgentTaskRoleTest {

    @Test
    void agent_task_accepts_role() {
        var task = new AgentTask("t1", "a1", "tenant1", "code-review",
                "java", "coordinator", "ext1", Instant.now(), null);
        assertThat(task.role()).isEqualTo("coordinator");
    }

    @Test
    void agent_task_accepts_null_role() {
        var task = new AgentTask("t1", "a1", "tenant1", "code-review",
                "java", null, "ext1", Instant.now(), null);
        assertThat(task.role()).isNull();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=AgentTaskRoleTest`
Expected: FAIL — constructor does not accept `role` parameter

- [ ] **Step 3: Add `role` field to AgentTask**

Use `ide_edit_member` on the `AgentTask` record declaration to add `role` after `taskDomain`:
```java
public record AgentTask(
    String taskId,
    String agentId,
    String tenancyId,
    String capabilityTag,
    String taskDomain,
    String role,
    String externalRef,
    Instant startedAt,
    Instant endedAt
) { ... }
```

- [ ] **Step 4: Fix all compilation errors from constructor change**

Use `ide_find_references` on `AgentTask` to find all callers. Update each constructor call to include the `role` parameter (null for existing calls). Key files:
- `AgentTaskEntity.toRecord()` — add `null` or `this.role` for the role parameter
- `AgentTaskEntity.from()` — read role from AgentTask
- Any test files constructing AgentTask

- [ ] **Step 5: Add `role` to AgentTaskEntity**

Use `ide_insert_member` to add the field:
```java
@Column(name = "role", length = 100)
private String role;
```

Update `from(AgentTask t)` — use `ide_replace_member`:
```java
entity.role = t.role();
```

Update `toRecord()` — use `ide_replace_member` to include `role` in the constructor call.

- [ ] **Step 6: Add `historyByRole` to AgentGraphQuery**

Use `ide_insert_member`:
```java
AgentTaskHistory historyByRole(String agentId, String roleName,
                               String tenancyId, Instant since);
```

- [ ] **Step 7: Implement `historyByRole` in JpaAgentGraphQuery**

Use `ide_insert_member`, following the `historyByCapability` pattern but filtering by `role` and `startedAt >= since`:
```java
@Override
public AgentTaskHistory historyByRole(final String agentId, final String roleName,
                                       final String tenancyId, final Instant since) {
    Objects.requireNonNull(agentId, "agentId");
    Objects.requireNonNull(roleName, "roleName");
    Objects.requireNonNull(tenancyId, "tenancyId");
    Objects.requireNonNull(since, "since");
    final var tasks = em.createQuery(
            "SELECT t FROM AgentTaskEntity t WHERE t.agentId = :agentId " +
            "AND t.tenancyId = :tenancyId AND t.role = :role " +
            "AND t.startedAt >= :since ORDER BY t.startedAt DESC",
            AgentTaskEntity.class)
        .setParameter("agentId", agentId)
        .setParameter("tenancyId", tenancyId)
        .setParameter("role", roleName)
        .setParameter("since", since)
        .getResultList();
    return toHistory(tasks);
}
```

- [ ] **Step 8: Create schema migration**

Write `graph/src/main/resources/db/migration/V4__role_health.sql`:
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

- [ ] **Step 9: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api,graph`
Expected: PASS

- [ ] **Step 10: Commit**

```bash
git add api/ graph/
git commit -m "feat(#271): add role field to AgentTask, historyByRole query, schema migration

Refs #271"
```

---

### Task 3: RoleHealth + RoleHoldCycleStore + RoleTransitionAdvisor SPIs

**Files:**
- Create: `api/src/main/java/io/casehub/eidos/api/RoleHealth.java`
- Create: `api/src/main/java/io/casehub/eidos/api/RoleHoldCycleStore.java`
- Create: `api/src/main/java/io/casehub/eidos/api/RoleTransitionAdvisor.java`

**Interfaces:**
- Consumes: `AgentDescriptor`, `ProbeContext` (from CapabilityHealth), `RoleChangePolicy`, `RoleChangePolicy.Mode`
- Produces: `RoleHealth.probe(AgentDescriptor, String, ProbeContext) → RoleStatus`
- Produces: `RoleStatus` sealed interface (DeclaredFit, Fit, Marginal, Unfit, Undeclared)
- Produces: `RoleHoldCycleStore.get/increment/reset` + `HoldCycleState` record
- Produces: `RoleTransitionAdvisor.evaluate(AgentDescriptor, RoleStatus) → TransitionAdvice`
- Produces: `TransitionAdvice` sealed interface (Hold, PromotionWarranted, DemotionAdvised, PolicyDisabled)

- [ ] **Step 1: Create RoleHealth SPI with sealed RoleStatus**

Use `ide_create_file` for `api/src/main/java/io/casehub/eidos/api/RoleHealth.java`:
```java
package io.casehub.eidos.api;

import io.casehub.eidos.api.CapabilityHealth.ProbeContext;

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

- [ ] **Step 2: Create RoleHoldCycleStore SPI**

Use `ide_create_file` for `api/src/main/java/io/casehub/eidos/api/RoleHoldCycleStore.java`:
```java
package io.casehub.eidos.api;

import java.time.Instant;

public interface RoleHoldCycleStore {

    HoldCycleState get(String agentId, String roleName, String tenancyId);

    void increment(String agentId, String roleName, String tenancyId);

    void reset(String agentId, String roleName, String tenancyId);

    record HoldCycleState(int consecutiveCycles, Instant lastEvaluatedAt) {
        public static final HoldCycleState EMPTY = new HoldCycleState(0, null);
    }
}
```

- [ ] **Step 3: Create RoleTransitionAdvisor SPI**

Use `ide_create_file` for `api/src/main/java/io/casehub/eidos/api/RoleTransitionAdvisor.java`:
```java
package io.casehub.eidos.api;

public interface RoleTransitionAdvisor {

    TransitionAdvice evaluate(AgentDescriptor descriptor,
                              RoleHealth.RoleStatus currentStatus);

    sealed interface TransitionAdvice permits
            TransitionAdvice.Hold,
            TransitionAdvice.PromotionWarranted,
            TransitionAdvice.DemotionAdvised,
            TransitionAdvice.PolicyDisabled {

        record Hold(String roleName, double confidence,
                    RoleChangePolicy.Mode policyMode) implements TransitionAdvice {}

        record PromotionWarranted(String roleName, double confidence,
                                  RoleChangePolicy.Mode policyMode) implements TransitionAdvice {}

        record DemotionAdvised(String roleName, double confidence,
                               int holdCyclesElapsed, int holdCyclesRequired,
                               RoleChangePolicy.Mode policyMode) implements TransitionAdvice {}

        record PolicyDisabled() implements TransitionAdvice {}
    }
}
```

- [ ] **Step 4: Verify compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api`
Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```bash
git add api/
git commit -m "feat(#271): add RoleHealth, RoleHoldCycleStore, RoleTransitionAdvisor SPIs

Refs #271"
```

---

## Batch 2: Implementation + Wiring

### Task 4: DefaultRoleHealth + NoOps + CDI wiring

**Files:**
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/health/DefaultRoleHealth.java`
- Create: `runtime/src/test/java/io/casehub/eidos/runtime/health/DefaultRoleHealthTest.java`
- Create: `eidos-core/src/main/java/io/casehub/eidos/core/health/NoOpRoleHealth.java`
- Create: `eidos-core/src/main/java/io/casehub/eidos/core/health/NoOpRoleTransitionAdvisor.java`
- Create: `eidos-core/src/main/java/io/casehub/eidos/core/health/NoOpRoleHoldCycleStore.java`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/EidosCoreProducer.java` — add 3 producer methods

**Interfaces:**
- Consumes: `RoleHealth` SPI, `AgentGraphQuery.historyByRole()`, `WilsonConfidence.score()`, `GraphDataSufficiency.forCount()`, `RoleChangePolicy`, `ProbeContext`
- Produces: `DefaultRoleHealth` — `@ApplicationScoped` implementation
- Produces: `NoOpRoleHealth`, `NoOpRoleTransitionAdvisor`, `NoOpRoleHoldCycleStore` — fallback beans

- [ ] **Step 1: Write DefaultRoleHealth failing test**

```java
package io.casehub.eidos.runtime.health;

import io.casehub.eidos.api.*;
import io.casehub.eidos.api.CapabilityHealth.ProbeContext;
import io.casehub.eidos.api.RoleHealth.RoleStatus;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class DefaultRoleHealthTest {

    private StubGraphQuery graphQuery;
    private DefaultRoleHealth roleHealth;

    @BeforeEach
    void setUp() {
        graphQuery = new StubGraphQuery();
        roleHealth = new DefaultRoleHealth(graphQuery, 90);
    }

    @Test
    void undeclared_when_no_role_profile() {
        var descriptor = descriptorWithoutRoleProfile();
        var status = roleHealth.probe(descriptor, "coordinator", ProbeContext.EMPTY);
        assertThat(status).isInstanceOf(RoleStatus.Undeclared.class);
        assertThat(status.roleName()).isEqualTo("coordinator");
    }

    @Test
    void undeclared_when_role_not_in_profile() {
        var descriptor = descriptorWithRoles("analyst");
        var status = roleHealth.probe(descriptor, "coordinator", ProbeContext.EMPTY);
        assertThat(status).isInstanceOf(RoleStatus.Undeclared.class);
    }

    @Test
    void declared_fit_when_zero_outcomes() {
        var descriptor = descriptorWithRoles("coordinator");
        graphQuery.setHistory("coordinator", AgentTaskHistory.EMPTY);
        var status = roleHealth.probe(descriptor, "coordinator", ProbeContext.EMPTY);
        assertThat(status).isInstanceOf(RoleStatus.DeclaredFit.class);
    }

    @Test
    void declared_fit_when_insufficient_outcomes() {
        var descriptor = descriptorWithRoles("coordinator");
        graphQuery.setHistory("coordinator", historyWithSuccesses(3));
        var status = roleHealth.probe(descriptor, "coordinator", ProbeContext.EMPTY);
        assertThat(status).isInstanceOf(RoleStatus.DeclaredFit.class);
    }

    @Test
    void fit_when_above_min_confidence() {
        var descriptor = descriptorWithRoles("coordinator",
                new RoleChangePolicy(RoleChangePolicy.Mode.AUTO_ABOVE_THRESHOLD,
                        0.5, 0.3, 2));
        graphQuery.setHistory("coordinator", historyWithSuccesses(20));
        var status = roleHealth.probe(descriptor, "coordinator", ProbeContext.EMPTY);
        assertThat(status).isInstanceOf(RoleStatus.Fit.class);
        assertThat(((RoleStatus.Fit) status).confidence()).isGreaterThan(0.5);
    }

    @Test
    void unfit_when_below_demotion_confidence() {
        var descriptor = descriptorWithRoles("coordinator",
                new RoleChangePolicy(RoleChangePolicy.Mode.AUTO_ABOVE_THRESHOLD,
                        0.7, 0.4, 2));
        graphQuery.setHistory("coordinator", historyWithFailures(15));
        var status = roleHealth.probe(descriptor, "coordinator", ProbeContext.EMPTY);
        assertThat(status).isInstanceOf(RoleStatus.Unfit.class);
    }

    // Helper methods: descriptorWithoutRoleProfile(), descriptorWithRoles(),
    // historyWithSuccesses(), historyWithFailures(), StubGraphQuery
    // — implement as private inner classes/methods
}
```

Note: the test helpers (StubGraphQuery, descriptor builders, history builders) will need to be written as part of this step. StubGraphQuery implements `AgentGraphQuery` returning canned `AgentTaskHistory` per role.

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=DefaultRoleHealthTest`
Expected: FAIL — DefaultRoleHealth class not found

- [ ] **Step 3: Implement DefaultRoleHealth**

Use `ide_create_file`:
```java
package io.casehub.eidos.runtime.health;

import io.casehub.eidos.api.*;
import io.casehub.eidos.api.CapabilityHealth.ProbeContext;
import io.casehub.eidos.api.RoleHealth.RoleStatus;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.eclipse.microprofile.config.inject.ConfigProperty;

import java.time.Instant;
import java.time.temporal.ChronoUnit;

@ApplicationScoped
public class DefaultRoleHealth implements RoleHealth {

    private final AgentGraphQuery graphQuery;
    private final int windowDays;

    @Inject
    public DefaultRoleHealth(final AgentGraphQuery graphQuery,
                             @ConfigProperty(name = "casehub.eidos.role-health.window-days",
                                     defaultValue = "90") final int windowDays) {
        this.graphQuery = graphQuery;
        this.windowDays = windowDays;
    }

    @Override
    public RoleStatus probe(final AgentDescriptor descriptor, final String roleName,
                            final ProbeContext context) {
        if (descriptor.roleProfile() == null) {
            return new RoleStatus.Undeclared(roleName);
        }
        if (!descriptor.roleProfile().hasRole(roleName)) {
            return new RoleStatus.Undeclared(roleName);
        }

        final Instant since = Instant.now().minus(windowDays, ChronoUnit.DAYS);
        final var history = graphQuery.historyByRole(
                descriptor.agentId(), roleName, descriptor.tenancyId(), since);

        final int n = history.outcomes().size();
        final var sufficiency = GraphDataSufficiency.forCount(n, since, Instant.now());
        if (sufficiency.level() == GraphDataSufficiency.SufficiencyLevel.INSUFFICIENT) {
            return new RoleStatus.DeclaredFit(roleName);
        }

        double sumQuality = 0.0;
        int failureCount = 0;
        for (final var outcome : history.outcomes()) {
            sumQuality += outcome.confidence() * multiplier(outcome.result());
            if (outcome.result() == TaskResult.FAILED) failureCount++;
        }

        final double confidence = WilsonConfidence.score(sumQuality, n);
        final var policy = descriptor.roleProfile().changePolicy();

        if (confidence >= policy.minConfidence()) {
            return new RoleStatus.Fit(roleName, confidence);
        } else if (confidence >= policy.demotionConfidence()) {
            return new RoleStatus.Marginal(roleName, confidence, n);
        } else {
            return new RoleStatus.Unfit(roleName, confidence, failureCount, n);
        }
    }

    private static double multiplier(final TaskResult result) {
        return switch (result) {
            case SUCCEEDED -> 1.0;
            case PARTIALLY -> 0.5;
            case FAILED -> 0.0;
        };
    }
}
```

- [ ] **Step 4: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=DefaultRoleHealthTest`
Expected: PASS

- [ ] **Step 5: Create NoOps in eidos-core**

Use `ide_create_file` for each:

`NoOpRoleHealth.java`:
```java
package io.casehub.eidos.core.health;

import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.eidos.api.CapabilityHealth.ProbeContext;
import io.casehub.eidos.api.RoleHealth;

public class NoOpRoleHealth implements RoleHealth {
    @Override
    public RoleStatus probe(final AgentDescriptor descriptor, final String roleName,
                            final ProbeContext context) {
        return new RoleStatus.DeclaredFit(roleName);
    }
}
```

`NoOpRoleTransitionAdvisor.java`:
```java
package io.casehub.eidos.core.health;

import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.eidos.api.RoleHealth.RoleStatus;
import io.casehub.eidos.api.RoleTransitionAdvisor;

public class NoOpRoleTransitionAdvisor implements RoleTransitionAdvisor {
    @Override
    public TransitionAdvice evaluate(final AgentDescriptor descriptor,
                                     final RoleStatus currentStatus) {
        return new TransitionAdvice.PolicyDisabled();
    }
}
```

`NoOpRoleHoldCycleStore.java`:
```java
package io.casehub.eidos.core.health;

import io.casehub.eidos.api.RoleHoldCycleStore;

public class NoOpRoleHoldCycleStore implements RoleHoldCycleStore {
    @Override
    public HoldCycleState get(final String agentId, final String roleName,
                              final String tenancyId) {
        return HoldCycleState.EMPTY;
    }

    @Override
    public void increment(final String agentId, final String roleName,
                          final String tenancyId) {}

    @Override
    public void reset(final String agentId, final String roleName,
                      final String tenancyId) {}
}
```

- [ ] **Step 6: Add producer methods to EidosCoreProducer**

Use `ide_insert_member` to add 3 methods to `EidosCoreProducer`:
```java
@Produces
@DefaultBean
public RoleHealth roleHealth() {
    return new NoOpRoleHealth();
}

@Produces
@DefaultBean
public RoleTransitionAdvisor roleTransitionAdvisor() {
    return new NoOpRoleTransitionAdvisor();
}

@Produces
@DefaultBean
public RoleHoldCycleStore roleHoldCycleStore() {
    return new NoOpRoleHoldCycleStore();
}
```

- [ ] **Step 7: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api,eidos-core,runtime`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add api/ eidos-core/ runtime/
git commit -m "feat(#271): implement DefaultRoleHealth, NoOp fallbacks, CDI wiring

Refs #271"
```

---

### Task 5: InMemoryRoleHoldCycleStore + DefaultRoleTransitionAdvisor

**Files:**
- Create: `persistence-memory/src/main/java/io/casehub/eidos/memory/InMemoryRoleHoldCycleStore.java`
- Create: `persistence-memory/src/test/java/io/casehub/eidos/memory/InMemoryRoleHoldCycleStoreTest.java`
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/health/DefaultRoleTransitionAdvisor.java`
- Create: `runtime/src/test/java/io/casehub/eidos/runtime/health/DefaultRoleTransitionAdvisorTest.java`

**Interfaces:**
- Consumes: `RoleHoldCycleStore` SPI, `RoleHealth.RoleStatus`, `RoleChangePolicy`, `RoleTransitionAdvisor` SPI
- Produces: `InMemoryRoleHoldCycleStore` — `@Alternative @Priority(1)`
- Produces: `DefaultRoleTransitionAdvisor` — `@ApplicationScoped`

- [ ] **Step 1: Write InMemoryRoleHoldCycleStore failing test**

```java
package io.casehub.eidos.memory;

import io.casehub.eidos.api.RoleHoldCycleStore;
import io.casehub.eidos.api.RoleHoldCycleStore.HoldCycleState;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class InMemoryRoleHoldCycleStoreTest {

    private InMemoryRoleHoldCycleStore store;

    @BeforeEach
    void setUp() {
        store = new InMemoryRoleHoldCycleStore();
    }

    @Test
    void get_returns_empty_for_unknown_key() {
        var state = store.get("a1", "coordinator", "t1");
        assertThat(state).isEqualTo(HoldCycleState.EMPTY);
    }

    @Test
    void increment_creates_and_increments() {
        store.increment("a1", "coordinator", "t1");
        var state = store.get("a1", "coordinator", "t1");
        assertThat(state.consecutiveCycles()).isEqualTo(1);
        assertThat(state.lastEvaluatedAt()).isNotNull();
    }

    @Test
    void increment_accumulates() {
        store.increment("a1", "coordinator", "t1");
        store.increment("a1", "coordinator", "t1");
        store.increment("a1", "coordinator", "t1");
        assertThat(store.get("a1", "coordinator", "t1").consecutiveCycles()).isEqualTo(3);
    }

    @Test
    void reset_clears_cycles_and_updates_timestamp() {
        store.increment("a1", "coordinator", "t1");
        store.increment("a1", "coordinator", "t1");
        store.reset("a1", "coordinator", "t1");
        var state = store.get("a1", "coordinator", "t1");
        assertThat(state.consecutiveCycles()).isEqualTo(0);
        assertThat(state.lastEvaluatedAt()).isNotNull();
    }

    @Test
    void different_roles_independent() {
        store.increment("a1", "coordinator", "t1");
        store.increment("a1", "analyst", "t1");
        store.increment("a1", "analyst", "t1");
        assertThat(store.get("a1", "coordinator", "t1").consecutiveCycles()).isEqualTo(1);
        assertThat(store.get("a1", "analyst", "t1").consecutiveCycles()).isEqualTo(2);
    }

    @Test
    void concurrent_increment_is_thread_safe() throws Exception {
        final int threads = 20;
        final Thread[] ts = new Thread[threads];
        for (int i = 0; i < threads; i++) {
            ts[i] = new Thread(() -> store.increment("a1", "coordinator", "t1"));
        }
        for (final Thread t : ts) t.start();
        for (final Thread t : ts) t.join();
        assertThat(store.get("a1", "coordinator", "t1").consecutiveCycles()).isEqualTo(threads);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory -Dtest=InMemoryRoleHoldCycleStoreTest`
Expected: FAIL — class not found

- [ ] **Step 3: Implement InMemoryRoleHoldCycleStore**

Use `ide_create_file`:
```java
package io.casehub.eidos.memory;

import io.casehub.eidos.api.RoleHoldCycleStore;
import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;

import java.time.Instant;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicReference;

@Alternative
@Priority(1)
@ApplicationScoped
public class InMemoryRoleHoldCycleStore implements RoleHoldCycleStore {

    private final ConcurrentHashMap<String, AtomicReference<HoldCycleState>> store =
            new ConcurrentHashMap<>();

    @Override
    public HoldCycleState get(final String agentId, final String roleName,
                              final String tenancyId) {
        final var ref = store.get(key(agentId, roleName, tenancyId));
        return ref == null ? HoldCycleState.EMPTY : ref.get();
    }

    @Override
    public void increment(final String agentId, final String roleName,
                          final String tenancyId) {
        store.computeIfAbsent(key(agentId, roleName, tenancyId),
                k -> new AtomicReference<>(HoldCycleState.EMPTY))
            .updateAndGet(s -> new HoldCycleState(s.consecutiveCycles() + 1, Instant.now()));
    }

    @Override
    public void reset(final String agentId, final String roleName,
                      final String tenancyId) {
        store.computeIfAbsent(key(agentId, roleName, tenancyId),
                k -> new AtomicReference<>(HoldCycleState.EMPTY))
            .set(new HoldCycleState(0, Instant.now()));
    }

    private static String key(final String agentId, final String roleName,
                              final String tenancyId) {
        return agentId + "|" + roleName + "|" + tenancyId;
    }
}
```

- [ ] **Step 4: Run store tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory -Dtest=InMemoryRoleHoldCycleStoreTest`
Expected: PASS

- [ ] **Step 5: Write DefaultRoleTransitionAdvisor failing test**

```java
package io.casehub.eidos.runtime.health;

import io.casehub.eidos.api.*;
import io.casehub.eidos.api.RoleHealth.RoleStatus;
import io.casehub.eidos.api.RoleTransitionAdvisor.TransitionAdvice;
import io.casehub.eidos.memory.InMemoryRoleHoldCycleStore;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class DefaultRoleTransitionAdvisorTest {

    private InMemoryRoleHoldCycleStore holdCycleStore;
    private DefaultRoleTransitionAdvisor advisor;

    @BeforeEach
    void setUp() {
        holdCycleStore = new InMemoryRoleHoldCycleStore();
        advisor = new DefaultRoleTransitionAdvisor(holdCycleStore, 90);
    }

    @Test
    void policy_disabled_returns_policy_disabled() {
        var descriptor = descriptorWithPolicy(RoleChangePolicy.DISABLED);
        var advice = advisor.evaluate(descriptor, new RoleStatus.Fit("coordinator", 0.9));
        assertThat(advice).isInstanceOf(TransitionAdvice.PolicyDisabled.class);
    }

    @Test
    void undeclared_throws_illegal_argument() {
        var descriptor = descriptorWithPolicy(RoleChangePolicy.ADVISORY);
        assertThatThrownBy(() -> advisor.evaluate(descriptor,
                new RoleStatus.Undeclared("coordinator")))
            .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void fit_resets_hold_cycles_and_returns_promotion_for_auto() {
        var policy = new RoleChangePolicy(RoleChangePolicy.Mode.AUTO_ABOVE_THRESHOLD,
                0.7, 0.4, 2);
        var descriptor = descriptorWithPolicy(policy);
        holdCycleStore.increment("a1", "coordinator", "t1");
        holdCycleStore.increment("a1", "coordinator", "t1");

        var advice = advisor.evaluate(descriptor, new RoleStatus.Fit("coordinator", 0.85));
        assertThat(advice).isInstanceOf(TransitionAdvice.PromotionWarranted.class);
        assertThat(holdCycleStore.get("a1", "coordinator", "t1").consecutiveCycles()).isEqualTo(0);
    }

    @Test
    void fit_returns_hold_for_advisory_mode() {
        var descriptor = descriptorWithPolicy(RoleChangePolicy.ADVISORY);
        var advice = advisor.evaluate(descriptor, new RoleStatus.Fit("coordinator", 0.85));
        assertThat(advice).isInstanceOf(TransitionAdvice.Hold.class);
    }

    @Test
    void marginal_returns_hold() {
        var policy = new RoleChangePolicy(RoleChangePolicy.Mode.AUTO_ABOVE_THRESHOLD,
                0.7, 0.4, 2);
        var descriptor = descriptorWithPolicy(policy);
        var advice = advisor.evaluate(descriptor,
                new RoleStatus.Marginal("coordinator", 0.55, 15));
        assertThat(advice).isInstanceOf(TransitionAdvice.Hold.class);
    }

    @Test
    void unfit_accumulates_hold_cycles_before_demotion() {
        var policy = new RoleChangePolicy(RoleChangePolicy.Mode.AUTO_ABOVE_THRESHOLD,
                0.7, 0.4, 3);
        var descriptor = descriptorWithPolicy(policy);
        var unfit = new RoleStatus.Unfit("coordinator", 0.25, 10, 15);

        // Cycle 1 — hold
        var advice1 = advisor.evaluate(descriptor, unfit);
        assertThat(advice1).isInstanceOf(TransitionAdvice.Hold.class);

        // Cycle 2 — still hold
        var advice2 = advisor.evaluate(descriptor, unfit);
        assertThat(advice2).isInstanceOf(TransitionAdvice.Hold.class);

        // Cycle 3 — demotion advised
        var advice3 = advisor.evaluate(descriptor, unfit);
        assertThat(advice3).isInstanceOf(TransitionAdvice.DemotionAdvised.class);
        var demotion = (TransitionAdvice.DemotionAdvised) advice3;
        assertThat(demotion.holdCyclesElapsed()).isEqualTo(3);
        assertThat(demotion.holdCyclesRequired()).isEqualTo(3);
    }

    @Test
    void declared_fit_resets_hold_cycles() {
        var policy = new RoleChangePolicy(RoleChangePolicy.Mode.AUTO_ABOVE_THRESHOLD,
                0.7, 0.4, 2);
        var descriptor = descriptorWithPolicy(policy);
        holdCycleStore.increment("a1", "coordinator", "t1");

        advisor.evaluate(descriptor, new RoleStatus.DeclaredFit("coordinator"));
        assertThat(holdCycleStore.get("a1", "coordinator", "t1").consecutiveCycles()).isEqualTo(0);
    }

    // Helper: descriptorWithPolicy() — creates minimal AgentDescriptor with the given policy
}
```

- [ ] **Step 6: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=DefaultRoleTransitionAdvisorTest`
Expected: FAIL — class not found

- [ ] **Step 7: Implement DefaultRoleTransitionAdvisor**

Use `ide_create_file`:
```java
package io.casehub.eidos.runtime.health;

import io.casehub.eidos.api.*;
import io.casehub.eidos.api.RoleHealth.RoleStatus;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.eclipse.microprofile.config.inject.ConfigProperty;

import java.time.Instant;
import java.time.temporal.ChronoUnit;

@ApplicationScoped
public class DefaultRoleTransitionAdvisor implements RoleTransitionAdvisor {

    private final RoleHoldCycleStore holdCycleStore;
    private final int windowDays;

    @Inject
    public DefaultRoleTransitionAdvisor(
            final RoleHoldCycleStore holdCycleStore,
            @ConfigProperty(name = "casehub.eidos.role-health.window-days",
                    defaultValue = "90") final int windowDays) {
        this.holdCycleStore = holdCycleStore;
        this.windowDays = windowDays;
    }

    @Override
    public TransitionAdvice evaluate(final AgentDescriptor descriptor,
                                     final RoleStatus currentStatus) {
        if (currentStatus instanceof RoleStatus.Undeclared) {
            throw new IllegalArgumentException(
                    "Cannot evaluate transition for undeclared role: " +
                    currentStatus.roleName());
        }

        final var policy = descriptor.roleProfile().changePolicy();
        if (policy.mode() == RoleChangePolicy.Mode.DISABLED) {
            return new TransitionAdvice.PolicyDisabled();
        }

        final String roleName = currentStatus.roleName();
        final String agentId = descriptor.agentId();
        final String tenancyId = descriptor.tenancyId();

        // Check staleness — reset if last evaluation is outside the temporal window
        final var cycleState = holdCycleStore.get(agentId, roleName, tenancyId);
        if (cycleState.lastEvaluatedAt() != null) {
            final Instant staleThreshold = Instant.now().minus(windowDays, ChronoUnit.DAYS);
            if (cycleState.lastEvaluatedAt().isBefore(staleThreshold)) {
                holdCycleStore.reset(agentId, roleName, tenancyId);
            }
        }

        return switch (currentStatus) {
            case RoleStatus.DeclaredFit df -> {
                holdCycleStore.reset(agentId, roleName, tenancyId);
                yield new TransitionAdvice.Hold(roleName, 0.0, policy.mode());
            }
            case RoleStatus.Fit fit -> {
                holdCycleStore.reset(agentId, roleName, tenancyId);
                if (policy.mode() == RoleChangePolicy.Mode.ADVISORY) {
                    yield new TransitionAdvice.Hold(roleName, fit.confidence(), policy.mode());
                }
                yield new TransitionAdvice.PromotionWarranted(
                        roleName, fit.confidence(), policy.mode());
            }
            case RoleStatus.Marginal marginal ->
                new TransitionAdvice.Hold(roleName, marginal.confidence(), policy.mode());
            case RoleStatus.Unfit unfit -> {
                holdCycleStore.increment(agentId, roleName, tenancyId);
                final var updated = holdCycleStore.get(agentId, roleName, tenancyId);
                if (updated.consecutiveCycles() >= policy.minHoldCycles()) {
                    yield new TransitionAdvice.DemotionAdvised(
                            roleName, unfit.confidence(),
                            updated.consecutiveCycles(), policy.minHoldCycles(),
                            policy.mode());
                }
                yield new TransitionAdvice.Hold(roleName, unfit.confidence(), policy.mode());
            }
            case RoleStatus.Undeclared u ->
                throw new IllegalArgumentException("unreachable");
        };
    }
}
```

- [ ] **Step 8: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api,eidos-core,runtime,persistence-memory`
Expected: PASS

- [ ] **Step 9: Commit**

```bash
git add persistence-memory/ runtime/
git commit -m "feat(#271): implement DefaultRoleTransitionAdvisor, InMemoryRoleHoldCycleStore

Refs #271"
```

---

### Task 6: Integration test

**Files:**
- Create: `examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/RoleHealthScenarioTest.java`

**Interfaces:**
- Consumes: `RoleHealth`, `RoleTransitionAdvisor`, `AgentGraphStore`, `InMemoryRoleHoldCycleStore`

- [ ] **Step 1: Write integration test**

```java
package io.casehub.eidos.examples;

import io.casehub.eidos.api.*;
import io.casehub.eidos.api.CapabilityHealth.ProbeContext;
import io.casehub.eidos.api.RoleHealth.RoleStatus;
import io.casehub.eidos.api.RoleTransitionAdvisor.TransitionAdvice;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.time.Instant;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class RoleHealthScenarioTest {

    @Inject RoleHealth roleHealth;
    @Inject RoleTransitionAdvisor transitionAdvisor;
    @Inject AgentGraphStore graphStore;

    @Test
    void agent_with_declared_role_and_no_history_is_declared_fit() {
        var descriptor = buildDescriptorWithRole("coordinator",
                new RoleChangePolicy(RoleChangePolicy.Mode.AUTO_ABOVE_THRESHOLD,
                        0.7, 0.4, 2));
        var status = roleHealth.probe(descriptor, "coordinator", ProbeContext.EMPTY);
        assertThat(status).isInstanceOf(RoleStatus.DeclaredFit.class);
    }

    @Test
    void agent_with_strong_role_history_is_fit() {
        var descriptor = buildDescriptorWithRole("coordinator",
                new RoleChangePolicy(RoleChangePolicy.Mode.AUTO_ABOVE_THRESHOLD,
                        0.5, 0.3, 2));
        recordSuccessfulTasks(descriptor, "coordinator", 15);
        var status = roleHealth.probe(descriptor, "coordinator", ProbeContext.EMPTY);
        assertThat(status).isInstanceOf(RoleStatus.Fit.class);
    }

    @Test
    void transition_advisor_recommends_demotion_after_hold_cycles() {
        var policy = new RoleChangePolicy(RoleChangePolicy.Mode.AUTO_ABOVE_THRESHOLD,
                0.7, 0.4, 2);
        var descriptor = buildDescriptorWithRole("coordinator", policy);
        recordFailedTasks(descriptor, "coordinator", 15);

        var status = roleHealth.probe(descriptor, "coordinator", ProbeContext.EMPTY);
        assertThat(status).isInstanceOf(RoleStatus.Unfit.class);

        // Cycle 1
        var advice1 = transitionAdvisor.evaluate(descriptor, status);
        assertThat(advice1).isInstanceOf(TransitionAdvice.Hold.class);

        // Cycle 2 — demotion advised
        var advice2 = transitionAdvisor.evaluate(descriptor, status);
        assertThat(advice2).isInstanceOf(TransitionAdvice.DemotionAdvised.class);
    }

    // Helpers: buildDescriptorWithRole(), recordSuccessfulTasks(), recordFailedTasks()
}
```

- [ ] **Step 2: Run integration test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl examples/agent-scenarios -Dtest=RoleHealthScenarioTest`
Expected: PASS

- [ ] **Step 3: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS — all modules compile and all tests pass

- [ ] **Step 4: Commit**

```bash
git add examples/
git commit -m "test(#271): add RoleHealth integration scenario test

Closes #271"
```

---

## References

- [2026-11-24-role-health-probe-design.md] — design spec this plan implements
- `api/src/main/java/io/casehub/eidos/api/CapabilityHealth.java` — probe pattern
- `api/src/main/java/io/casehub/eidos/api/DispositionHealth.java` / `DispositionEvolution.java` — SPI split pattern
- `api/src/main/java/io/casehub/eidos/api/RoleProfile.java` / `RoleChangePolicy.java` — role model
- `api/src/main/java/io/casehub/eidos/api/AgentTask.java` / `AgentGraphQuery.java` — knowledge graph
- `api/src/main/java/io/casehub/eidos/api/GraphDataSufficiency.java` — data sufficiency gating
- `graph/src/main/java/io/casehub/eidos/graph/JpaAgentGraphQuery.java:wilsonScore` — Wilson computation to extract
- `runtime/src/main/java/io/casehub/eidos/runtime/EidosCoreProducer.java` — NoOp producer pattern
- `eidos-core/src/main/java/io/casehub/eidos/core/health/NoOpDispositionHealth.java` — NoOp class pattern
- GitHub #271 — focal issue
