# Behavioral Contracts Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Generalize `CapabilitySpecializationStore` into `BehavioralSignalStore` with COMPLIANT/VIOLATED signals, add behavioral compliance checking as probe Step 6, and provide attestation bridge utilities.

**Architecture:** Rename the existing specialization store to `BehavioralSignalStore`, extend the `BehavioralSignal` enum with COMPLIANT/VIOLATED, add `BehavioralViolation` to the `CapabilityStatus` sealed hierarchy, and add Step 6 to `DefaultCapabilityHealth.probe()` that queries VIOLATED signal counts against a configurable threshold. Convention constants and attestation utilities bridge to the ledger evidence layer.

**Tech Stack:** Java 21 (on Java 26 JVM), Quarkus 3.32.2, JPA/Hibernate, Flyway, AssertJ, Mockito

**Spec:** `specs/2026-07-02-behavioral-contracts-design.md`

## Global Constraints

- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- Test: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
- Use `mvn` not `./mvnw`
- All commits reference eidos#85
- Use IntelliJ MCP (`mcp__intellij-index__*`) for all renames, reference lookups, and navigation — never bash grep/find for classes
- After IntelliJ renames, sync files: `mcp__intellij-index__ide_sync_files`
- `domain` parameter renamed to `qualifier` in all SPI methods, entities, and schema
- No deployed instances — modify base migrations directly
- `CapabilityStatus` is a sealed interface — adding `BehavioralViolation` requires updating the `permits` clause

---

### Task 1: API — BehavioralSignal enum and BehavioralSignalStore SPI

Rename the foundational types in eidos-api. Everything else depends on these.

**Files:**
- Delete: `api/src/main/java/io/casehub/eidos/api/SpecializationSignal.java`
- Create: `api/src/main/java/io/casehub/eidos/api/BehavioralSignal.java`
- Delete: `api/src/main/java/io/casehub/eidos/api/CapabilitySpecializationStore.java`
- Create: `api/src/main/java/io/casehub/eidos/api/BehavioralSignalStore.java`
- Modify: `api/src/main/java/io/casehub/eidos/api/CapabilityResolver.java` (Javadoc only)

**Interfaces:**
- Produces: `BehavioralSignal { DECLINE, SUCCESS, COMPLIANT, VIOLATED }`
- Produces: `BehavioralSignalStore` SPI with `qualifier` parameter (replaces `domain`)

- [ ] **Step 1: Create `BehavioralSignal` enum**

```java
// api/src/main/java/io/casehub/eidos/api/BehavioralSignal.java
package io.casehub.eidos.api;

public enum BehavioralSignal {
    DECLINE,
    SUCCESS,
    COMPLIANT,
    VIOLATED
}
```

- [ ] **Step 2: Create `BehavioralSignalStore` SPI**

```java
// api/src/main/java/io/casehub/eidos/api/BehavioralSignalStore.java
package io.casehub.eidos.api;

import java.util.Map;

public interface BehavioralSignalStore {

    /**
     * Records one signal event for the given agent, capability, and qualifier.
     * TTL is owned by the store implementation — per-signal TTL is supported.
     *
     * <p>The {@code qualifier} parameter is a free-text key whose meaning depends
     * on signal type: task domain for DECLINE/SUCCESS signals, compliance dimension
     * key for COMPLIANT/VIOLATED signals.
     *
     * <p>{@code capabilityName} must be the agent's declared capability name
     * (as returned by {@link AgentCapability#name()}), not a query/lookup term.
     * When the caller has a query tag instead, use
     * {@link CapabilityResolver#resolve(java.util.List, String, VocabularyRegistry)}
     * to obtain the declared capability first.
     */
    void record(String agentId, String tenancyId, String capabilityName,
                String qualifier, BehavioralSignal signal);

    /**
     * Retracts all learned data of the given signal type for an
     * (agentId, tenancyId, capabilityName) triple.
     * Clears all qualifier entries regardless of TTL.
     *
     * <p>{@code capabilityName} must be the agent's declared capability name —
     * see {@link #record} for details.
     */
    void clear(String agentId, String tenancyId, String capabilityName,
               BehavioralSignal signal);

    /**
     * Returns qualifier to count of unexpired records for the given signal type,
     * for all qualifiers with at least one unexpired record.
     * Empty map when none. Never null.
     *
     * <p>{@code capabilityName} must be the agent's declared capability name —
     * see {@link #record} for details.
     */
    Map<String, Integer> learned(String agentId, String tenancyId,
                                 String capabilityName, BehavioralSignal signal);

    /**
     * Returns the count of unexpired records for the given signal type and qualifier.
     * 0 when no unexpired records exist. Never negative.
     *
     * <p>{@code capabilityName} must be the agent's declared capability name —
     * see {@link #record} for details.
     */
    int count(String agentId, String tenancyId, String capabilityName,
              String qualifier, BehavioralSignal signal);
}
```

- [ ] **Step 3: Delete old types**

Delete `api/src/main/java/io/casehub/eidos/api/SpecializationSignal.java`
Delete `api/src/main/java/io/casehub/eidos/api/CapabilitySpecializationStore.java`

- [ ] **Step 4: Update CapabilityResolver Javadoc**

In `api/src/main/java/io/casehub/eidos/api/CapabilityResolver.java`, update the Javadoc reference from `CapabilitySpecializationStore` to `BehavioralSignalStore`.

- [ ] **Step 5: Verify API module compiles**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api`
Expected: FAIL — runtime/memory/examples still reference old types. API module itself should compile.

- [ ] **Step 6: Commit**

```
feat(eidos#85): rename SpecializationSignal → BehavioralSignal, CapabilitySpecializationStore → BehavioralSignalStore

Generalizes the specialization signal store into a behavioral signal store.
Adds COMPLIANT and VIOLATED signal types for compliance checking.
Renames `domain` parameter to `qualifier` to reflect dual-purpose semantics.
```

---

### Task 2: API — BehavioralViolation status, ComplianceDimension, BehavioralExpectations

Add the new CapabilityStatus variant and utility types.

**Files:**
- Modify: `api/src/main/java/io/casehub/eidos/api/CapabilityHealth.java`
- Create: `api/src/main/java/io/casehub/eidos/api/ComplianceDimension.java`
- Create: `api/src/main/java/io/casehub/eidos/api/BehavioralExpectations.java`
- Test: `api/src/test/java/io/casehub/eidos/api/BehavioralExpectationsTest.java`

**Interfaces:**
- Consumes: `AgentCapability.latencyHintP50Ms()`, `AgentDisposition.delegation()`
- Produces: `CapabilityStatus.BehavioralViolation(Map<String, Integer>)`
- Produces: `ComplianceDimension.LATENCY`, `.ATTESTATION_RATE`, `.ATTESTOR_ID`, `.TRUST_DIMENSION_PREFIX`, `.LATENCY_VIOLATION_MULTIPLIER`
- Produces: `BehavioralExpectations.latencyBound(AgentCapability)`, `.delegationExpected(AgentDisposition)`

- [ ] **Step 1: Write failing test for BehavioralExpectations**

```java
// api/src/test/java/io/casehub/eidos/api/BehavioralExpectationsTest.java
package io.casehub.eidos.api;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class BehavioralExpectationsTest {

    @Test
    void latencyBound_returns_hint_when_present() {
        var cap = AgentCapability.builder().name("code-review")
                .latencyHintP50Ms(5000L).build();
        assertThat(BehavioralExpectations.latencyBound(cap)).hasValue(5000L);
    }

    @Test
    void latencyBound_empty_when_no_hint() {
        var cap = AgentCapability.builder().name("code-review").build();
        assertThat(BehavioralExpectations.latencyBound(cap)).isEmpty();
    }

    @Test
    void delegationExpected_true_when_delegation_flag_set() {
        var disp = AgentDisposition.builder().delegation(true).build();
        assertThat(BehavioralExpectations.delegationExpected(disp)).isTrue();
    }

    @Test
    void delegationExpected_false_when_delegation_not_set() {
        var disp = AgentDisposition.builder().build();
        assertThat(BehavioralExpectations.delegationExpected(disp)).isFalse();
    }

    @Test
    void delegationExpected_false_when_null_disposition() {
        assertThat(BehavioralExpectations.delegationExpected(null)).isFalse();
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=BehavioralExpectationsTest`
Expected: FAIL — `BehavioralExpectations` does not exist yet.

- [ ] **Step 3: Add BehavioralViolation to CapabilityStatus**

In `api/src/main/java/io/casehub/eidos/api/CapabilityHealth.java`, update the sealed interface:

```java
sealed interface CapabilityStatus permits
        CapabilityStatus.Degraded,
        CapabilityStatus.Unavailable,
        CapabilityStatus.Excluded,
        CapabilityStatus.EpistemicallyWeak,
        CapabilityStatus.BehavioralViolation,
        CapabilityStatus.Ready {

    record Ready() implements CapabilityStatus {}
    record Degraded(DegradationReason reason, String detail) implements CapabilityStatus {}
    record Unavailable(String reason) implements CapabilityStatus {}
    record EpistemicallyWeak(String domain, double confidence) implements CapabilityStatus {}
    record Excluded(String domain, ExclusionSource source, int declineCount) implements CapabilityStatus {}
    record BehavioralViolation(Map<String, Integer> violations) implements CapabilityStatus {}

    enum ExclusionSource { DECLARED, LEARNED }
}
```

Add `import java.util.Map;` to the imports if not already present.

- [ ] **Step 4: Create ComplianceDimension**

```java
// api/src/main/java/io/casehub/eidos/api/ComplianceDimension.java
package io.casehub.eidos.api;

public final class ComplianceDimension {

    public static final String LATENCY = "latency";
    public static final String ATTESTATION_RATE = "attestation-rate";

    public static final String ATTESTOR_ID = "eidos:compliance";
    public static final String TRUST_DIMENSION_PREFIX = "behavioral:";
    public static final double LATENCY_VIOLATION_MULTIPLIER = 2.0;

    private ComplianceDimension() {}
}
```

- [ ] **Step 5: Create BehavioralExpectations**

```java
// api/src/main/java/io/casehub/eidos/api/BehavioralExpectations.java
package io.casehub.eidos.api;

import java.util.OptionalLong;

public final class BehavioralExpectations {

    private BehavioralExpectations() {}

    public static OptionalLong latencyBound(final AgentCapability capability) {
        return capability.latencyHintP50Ms() != null
                ? OptionalLong.of(capability.latencyHintP50Ms())
                : OptionalLong.empty();
    }

    public static boolean delegationExpected(final AgentDisposition disposition) {
        return disposition != null && disposition.delegation();
    }
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=BehavioralExpectationsTest`
Expected: PASS

- [ ] **Step 7: Commit**

```
feat(eidos#85): add BehavioralViolation status, ComplianceDimension constants, BehavioralExpectations utility
```

---

### Task 3: Store implementations — rename, qualifier, TTL expansion

Rename all store implementations, entities, migration, and config properties. Expand `ttlDaysFor` to handle 4 signal types.

**Files:**
- Delete + Create (rename): all JPA entities, stores, NoOp store, InMemory store
- Modify: `runtime/src/main/resources/db/eidos/migration/V5__capability_specialization_bidirectional.sql` → `V5__behavioral_signal.sql`
- Modify: existing store tests

**Interfaces:**
- Consumes: `BehavioralSignal`, `BehavioralSignalStore` from Task 1
- Produces: `JpaBehavioralSignalStore`, `InMemoryBehavioralSignalStore`, `NoOpBehavioralSignalStore`

- [ ] **Step 1: Rename migration file and update schema**

Delete `runtime/src/main/resources/db/eidos/migration/V5__capability_specialization_bidirectional.sql`.
Create `runtime/src/main/resources/db/eidos/migration/V5__behavioral_signal.sql`:

```sql
DROP TABLE IF EXISTS behavioral_signal;

CREATE TABLE behavioral_signal (
    agent_id        VARCHAR(255) NOT NULL,
    tenancy_id      VARCHAR(255) NOT NULL,
    capability_name VARCHAR(100) NOT NULL,
    qualifier       VARCHAR(200) NOT NULL,
    signal_type     VARCHAR(20)  NOT NULL,
    signal_count    INT          NOT NULL DEFAULT 0,
    last_recorded   TIMESTAMP WITH TIME ZONE NOT NULL,
    expires_at      TIMESTAMP WITH TIME ZONE NOT NULL,
    PRIMARY KEY (agent_id, tenancy_id, capability_name, qualifier, signal_type)
);
```

- [ ] **Step 2: Create BehavioralSignalId**

```java
// runtime/src/main/java/io/casehub/eidos/runtime/health/jpa/BehavioralSignalId.java
package io.casehub.eidos.runtime.health.jpa;

import jakarta.persistence.Column;
import jakarta.persistence.Embeddable;
import java.io.Serializable;
import java.util.Objects;

@Embeddable
public class BehavioralSignalId implements Serializable {

    @Column(name = "agent_id")
    String agentId;

    @Column(name = "tenancy_id")
    String tenancyId;

    @Column(name = "capability_name")
    String capabilityName;

    @Column(name = "qualifier")
    String qualifier;

    @Column(name = "signal_type")
    String signalType;

    protected BehavioralSignalId() {}

    BehavioralSignalId(final String agentId, final String tenancyId,
                       final String capabilityName, final String qualifier,
                       final String signalType) {
        this.agentId = agentId;
        this.tenancyId = tenancyId;
        this.capabilityName = capabilityName;
        this.qualifier = qualifier;
        this.signalType = signalType;
    }

    @Override
    public boolean equals(final Object o) {
        if (this == o) return true;
        if (!(o instanceof BehavioralSignalId that)) return false;
        return Objects.equals(agentId, that.agentId)
            && Objects.equals(tenancyId, that.tenancyId)
            && Objects.equals(capabilityName, that.capabilityName)
            && Objects.equals(qualifier, that.qualifier)
            && Objects.equals(signalType, that.signalType);
    }

    @Override
    public int hashCode() {
        return Objects.hash(agentId, tenancyId, capabilityName, qualifier, signalType);
    }
}
```

- [ ] **Step 3: Create BehavioralSignalEntity**

```java
// runtime/src/main/java/io/casehub/eidos/runtime/health/jpa/BehavioralSignalEntity.java
package io.casehub.eidos.runtime.health.jpa;

import jakarta.persistence.*;
import java.time.Instant;

@Entity
@Table(name = "behavioral_signal")
public class BehavioralSignalEntity {

    @EmbeddedId
    BehavioralSignalId id;

    @Column(name = "signal_count", nullable = false)
    int signalCount;

    @Column(name = "last_recorded", nullable = false)
    Instant lastRecorded;

    @Column(name = "expires_at", nullable = false)
    Instant expiresAt;

    protected BehavioralSignalEntity() {}

    BehavioralSignalEntity(final String agentId, final String tenancyId,
                           final String capabilityName, final String qualifier,
                           final String signalType,
                           final int signalCount, final Instant lastRecorded,
                           final Instant expiresAt) {
        this.id = new BehavioralSignalId(agentId, tenancyId, capabilityName, qualifier, signalType);
        this.signalCount = signalCount;
        this.lastRecorded = lastRecorded;
        this.expiresAt = expiresAt;
    }
}
```

- [ ] **Step 4: Create JpaBehavioralSignalStore**

```java
// runtime/src/main/java/io/casehub/eidos/runtime/health/jpa/JpaBehavioralSignalStore.java
package io.casehub.eidos.runtime.health.jpa;

import io.casehub.eidos.api.BehavioralSignal;
import io.casehub.eidos.api.BehavioralSignalStore;
import io.quarkus.arc.properties.IfBuildProperty;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.transaction.Transactional;
import jakarta.transaction.Transactional.TxType;
import org.eclipse.microprofile.config.inject.ConfigProperty;

import java.time.Instant;
import java.util.HashMap;
import java.util.Map;

@IfBuildProperty(name = "casehub.eidos.reactive.enabled", stringValue = "false", enableIfMissing = true)
@ApplicationScoped
public class JpaBehavioralSignalStore implements BehavioralSignalStore {

    @Inject EntityManager em;

    @ConfigProperty(name = "casehub.eidos.behavioral-signal.decline-ttl-days", defaultValue = "30")
    int declineTtlDays;

    @ConfigProperty(name = "casehub.eidos.behavioral-signal.success-ttl-days", defaultValue = "30")
    int successTtlDays;

    @ConfigProperty(name = "casehub.eidos.behavioral-signal.compliant-ttl-days", defaultValue = "30")
    int compliantTtlDays;

    @ConfigProperty(name = "casehub.eidos.behavioral-signal.violated-ttl-days", defaultValue = "90")
    int violatedTtlDays;

    @Override
    @Transactional
    public void record(final String agentId, final String tenancyId,
                       final String capabilityName, final String qualifier,
                       final BehavioralSignal signal) {
        final var id = new BehavioralSignalId(agentId, tenancyId, capabilityName, qualifier, signal.name());
        final var existing = em.find(BehavioralSignalEntity.class, id);
        final var now = Instant.now();
        final var expiresAt = now.plusSeconds((long) ttlDaysFor(signal) * 86400);

        if (existing != null) {
            existing.signalCount++;
            existing.lastRecorded = now;
            existing.expiresAt = expiresAt;
        } else {
            em.persist(new BehavioralSignalEntity(
                agentId, tenancyId, capabilityName, qualifier, signal.name(),
                1, now, expiresAt));
        }
    }

    @Override
    @Transactional
    public void clear(final String agentId, final String tenancyId,
                      final String capabilityName, final BehavioralSignal signal) {
        em.createQuery("DELETE FROM BehavioralSignalEntity e"
                + " WHERE e.id.agentId = :agentId"
                + " AND e.id.tenancyId = :tenancyId"
                + " AND e.id.capabilityName = :capabilityName"
                + " AND e.id.signalType = :signalType")
            .setParameter("agentId", agentId)
            .setParameter("tenancyId", tenancyId)
            .setParameter("capabilityName", capabilityName)
            .setParameter("signalType", signal.name())
            .executeUpdate();
        em.flush();
        em.clear();
    }

    @Override
    @Transactional(TxType.SUPPORTS)
    public Map<String, Integer> learned(final String agentId, final String tenancyId,
                                         final String capabilityName,
                                         final BehavioralSignal signal) {
        final var results = em.createQuery(
                "SELECT e FROM BehavioralSignalEntity e"
                    + " WHERE e.id.agentId = :agentId"
                    + " AND e.id.tenancyId = :tenancyId"
                    + " AND e.id.capabilityName = :capabilityName"
                    + " AND e.id.signalType = :signalType"
                    + " AND e.expiresAt > :now",
                BehavioralSignalEntity.class)
            .setParameter("agentId", agentId)
            .setParameter("tenancyId", tenancyId)
            .setParameter("capabilityName", capabilityName)
            .setParameter("signalType", signal.name())
            .setParameter("now", Instant.now())
            .getResultList();

        final var map = new HashMap<String, Integer>();
        for (final var e : results) {
            map.put(e.id.qualifier, e.signalCount);
        }
        return Map.copyOf(map);
    }

    @Override
    @Transactional(TxType.SUPPORTS)
    public int count(final String agentId, final String tenancyId,
                     final String capabilityName, final String qualifier,
                     final BehavioralSignal signal) {
        final var id = new BehavioralSignalId(agentId, tenancyId, capabilityName, qualifier, signal.name());
        final var entity = em.find(BehavioralSignalEntity.class, id);
        if (entity == null || !Instant.now().isBefore(entity.expiresAt)) return 0;
        return entity.signalCount;
    }

    private int ttlDaysFor(final BehavioralSignal signal) {
        return switch (signal) {
            case DECLINE -> declineTtlDays;
            case SUCCESS -> successTtlDays;
            case COMPLIANT -> compliantTtlDays;
            case VIOLATED -> violatedTtlDays;
        };
    }
}
```

- [ ] **Step 5: Create NoOpBehavioralSignalStore**

```java
// runtime/src/main/java/io/casehub/eidos/runtime/health/NoOpBehavioralSignalStore.java
package io.casehub.eidos.runtime.health;

import io.casehub.eidos.api.BehavioralSignal;
import io.casehub.eidos.api.BehavioralSignalStore;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;

import java.util.Map;

@DefaultBean
@ApplicationScoped
public class NoOpBehavioralSignalStore implements BehavioralSignalStore {

    @Override
    public void record(final String agentId, final String tenancyId,
                       final String capabilityName, final String qualifier,
                       final BehavioralSignal signal) {}

    @Override
    public void clear(final String agentId, final String tenancyId,
                      final String capabilityName, final BehavioralSignal signal) {}

    @Override
    public Map<String, Integer> learned(final String agentId, final String tenancyId,
                                         final String capabilityName,
                                         final BehavioralSignal signal) {
        return Map.of();
    }

    @Override
    public int count(final String agentId, final String tenancyId,
                     final String capabilityName, final String qualifier,
                     final BehavioralSignal signal) {
        return 0;
    }
}
```

- [ ] **Step 6: Create InMemoryBehavioralSignalStore**

```java
// persistence-memory/src/main/java/io/casehub/eidos/memory/InMemoryBehavioralSignalStore.java
package io.casehub.eidos.memory;

import io.casehub.eidos.api.BehavioralSignal;
import io.casehub.eidos.api.BehavioralSignalStore;
import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import org.eclipse.microprofile.config.inject.ConfigProperty;

import java.time.Instant;
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentLinkedQueue;

@Alternative
@Priority(1)
@ApplicationScoped
public class InMemoryBehavioralSignalStore implements BehavioralSignalStore {

    @ConfigProperty(name = "casehub.eidos.behavioral-signal.decline-ttl-days", defaultValue = "30")
    int declineTtlDays;

    @ConfigProperty(name = "casehub.eidos.behavioral-signal.success-ttl-days", defaultValue = "30")
    int successTtlDays;

    @ConfigProperty(name = "casehub.eidos.behavioral-signal.compliant-ttl-days", defaultValue = "30")
    int compliantTtlDays;

    @ConfigProperty(name = "casehub.eidos.behavioral-signal.violated-ttl-days", defaultValue = "90")
    int violatedTtlDays;

    private final ConcurrentHashMap<StoreKey, ConcurrentHashMap<String, ConcurrentLinkedQueue<Instant>>>
        store = new ConcurrentHashMap<>();

    @Override
    public void record(final String agentId, final String tenancyId,
                       final String capabilityName, final String qualifier,
                       final BehavioralSignal signal) {
        final var key = new StoreKey(agentId, tenancyId, capabilityName, signal);
        final var qualifierQueues = store.computeIfAbsent(key, k -> new ConcurrentHashMap<>());
        final var queue = qualifierQueues.computeIfAbsent(qualifier, q -> new ConcurrentLinkedQueue<>());
        final Instant now = Instant.now();
        queue.removeIf(ts -> !now.isBefore(ts));
        queue.offer(now.plusSeconds((long) ttlDaysFor(signal) * 86400));
    }

    @Override
    public void clear(final String agentId, final String tenancyId,
                      final String capabilityName, final BehavioralSignal signal) {
        store.remove(new StoreKey(agentId, tenancyId, capabilityName, signal));
    }

    @Override
    public Map<String, Integer> learned(final String agentId, final String tenancyId,
                                         final String capabilityName,
                                         final BehavioralSignal signal) {
        final var qualifierQueues = store.get(new StoreKey(agentId, tenancyId, capabilityName, signal));
        if (qualifierQueues == null) return Map.of();
        final Instant now = Instant.now();
        final var result = new HashMap<String, Integer>();
        qualifierQueues.forEach((qualifier, queue) -> {
            final int cnt = (int) queue.stream().filter(ts -> now.isBefore(ts)).count();
            if (cnt > 0) result.put(qualifier, cnt);
        });
        return Map.copyOf(result);
    }

    @Override
    public int count(final String agentId, final String tenancyId,
                     final String capabilityName, final String qualifier,
                     final BehavioralSignal signal) {
        final var qualifierQueues = store.get(new StoreKey(agentId, tenancyId, capabilityName, signal));
        if (qualifierQueues == null) return 0;
        final var queue = qualifierQueues.get(qualifier);
        if (queue == null) return 0;
        final Instant now = Instant.now();
        return (int) queue.stream().filter(ts -> now.isBefore(ts)).count();
    }

    private int ttlDaysFor(final BehavioralSignal signal) {
        return switch (signal) {
            case DECLINE -> declineTtlDays;
            case SUCCESS -> successTtlDays;
            case COMPLIANT -> compliantTtlDays;
            case VIOLATED -> violatedTtlDays;
        };
    }

    private record StoreKey(String agentId, String tenancyId,
                             String capabilityName, BehavioralSignal signal) {}
}
```

- [ ] **Step 7: Delete old implementation files**

Delete (in order — entities before stores, to avoid dangling references):
- `runtime/src/main/java/io/casehub/eidos/runtime/health/jpa/CapabilitySpecializationId.java`
- `runtime/src/main/java/io/casehub/eidos/runtime/health/jpa/CapabilitySpecializationEntity.java`
- `runtime/src/main/java/io/casehub/eidos/runtime/health/jpa/JpaCapabilitySpecializationStore.java`
- `runtime/src/main/java/io/casehub/eidos/runtime/health/NoOpCapabilitySpecializationStore.java`
- `persistence-memory/src/main/java/io/casehub/eidos/memory/InMemoryCapabilitySpecializationStore.java`

- [ ] **Step 8: Update InMemory store test**

Rename and update `persistence-memory/src/test/java/io/casehub/eidos/memory/InMemoryCapabilitySpecializationStoreTest.java` → `InMemoryBehavioralSignalStoreTest.java`. Change all `SpecializationSignal` → `BehavioralSignal`, `InMemoryCapabilitySpecializationStore` → `InMemoryBehavioralSignalStore`, field names `declineTtlDays`/`successTtlDays` stay the same. Add tests for COMPLIANT and VIOLATED signal TTLs.

Add these tests to the existing test body:

```java
// Add imports
import static io.casehub.eidos.api.BehavioralSignal.COMPLIANT;
import static io.casehub.eidos.api.BehavioralSignal.VIOLATED;

// Add to setUp():
setTtl(store, "compliantTtlDays", 30);
setTtl(store, "violatedTtlDays", 90);

// Add test methods:
@Test
void compliant_signal_recorded_and_counted() {
    store.record("a1", "t1", "code-review", "latency", COMPLIANT);
    assertThat(store.count("a1", "t1", "code-review", "latency", COMPLIANT)).isEqualTo(1);
}

@Test
void violated_signal_recorded_and_counted() {
    store.record("a1", "t1", "code-review", "latency", VIOLATED);
    assertThat(store.count("a1", "t1", "code-review", "latency", VIOLATED)).isEqualTo(1);
}

@Test
void violated_expires_after_ttl() throws Exception {
    setTtl(store, "violatedTtlDays", -1);
    store.record("a1", "t1", "code-review", "latency", VIOLATED);
    assertThat(store.count("a1", "t1", "code-review", "latency", VIOLATED)).isEqualTo(0);
}

@Test
void compliant_and_violated_coexist_independently() {
    store.record("a1", "t1", "code-review", "latency", COMPLIANT);
    store.record("a1", "t1", "code-review", "latency", COMPLIANT);
    store.record("a1", "t1", "code-review", "latency", VIOLATED);

    assertThat(store.count("a1", "t1", "code-review", "latency", COMPLIANT)).isEqualTo(2);
    assertThat(store.count("a1", "t1", "code-review", "latency", VIOLATED)).isEqualTo(1);
}

@Test
void learned_returns_violated_qualifiers() {
    store.record("a1", "t1", "code-review", "latency", VIOLATED);
    store.record("a1", "t1", "code-review", "latency", VIOLATED);
    store.record("a1", "t1", "code-review", "attestation-rate", VIOLATED);

    var violations = store.learned("a1", "t1", "code-review", VIOLATED);
    assertThat(violations).containsEntry("latency", 2).containsEntry("attestation-rate", 1);
}
```

- [ ] **Step 9: Run InMemory store tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory`
Expected: PASS

- [ ] **Step 10: Commit**

```
feat(eidos#85): rename store implementations — JPA, InMemory, NoOp — with qualifier column and 4-signal TTL
```

---

### Task 4: Probe Step 6 — behavioral compliance checking (TDD)

The core new behavior. Write tests first, then implement.

**Files:**
- Create: `runtime/src/test/java/io/casehub/eidos/runtime/health/DefaultCapabilityHealthBehavioralViolationTest.java`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/health/DefaultCapabilityHealth.java`
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/preferences/ComplianceViolationThresholdPreference.java`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/preferences/EidosPreferenceKeys.java`

**Interfaces:**
- Consumes: `BehavioralSignalStore.learned()`, `BehavioralSignal.VIOLATED`
- Consumes: `CapabilityResolver.resolve()`, `CapabilityStatus.BehavioralViolation`
- Produces: Step 6 in `DefaultCapabilityHealth.probe()`

- [ ] **Step 1: Create ComplianceViolationThresholdPreference**

```java
// runtime/src/main/java/io/casehub/eidos/runtime/preferences/ComplianceViolationThresholdPreference.java
package io.casehub.eidos.runtime.preferences;

import io.casehub.platform.api.preferences.SingleValuePreference;

public record ComplianceViolationThresholdPreference(int value) implements SingleValuePreference {
    public ComplianceViolationThresholdPreference {
        if (value < 1) throw new IllegalArgumentException(
            "behavioral.compliance-violation-threshold must be >= 1, got: " + value);
    }
}
```

- [ ] **Step 2: Add COMPLIANCE_VIOLATION_THRESHOLD to EidosPreferenceKeys**

In `runtime/src/main/java/io/casehub/eidos/runtime/preferences/EidosPreferenceKeys.java`, add:

```java
public static final PreferenceKey<ComplianceViolationThresholdPreference> COMPLIANCE_VIOLATION_THRESHOLD =
    new PreferenceKey<>("casehub.eidos", "behavioral.compliance-violation-threshold",
                        new ComplianceViolationThresholdPreference(3),
                        s -> new ComplianceViolationThresholdPreference(Integer.parseInt(s)));
```

- [ ] **Step 3: Write failing test — violations below threshold returns Ready**

```java
// runtime/src/test/java/io/casehub/eidos/runtime/health/DefaultCapabilityHealthBehavioralViolationTest.java
package io.casehub.eidos.runtime.health;

import io.casehub.eidos.api.*;
import io.casehub.eidos.api.CapabilityHealth.CapabilityStatus;
import io.casehub.eidos.api.CapabilityHealth.ProbeContext;
import io.casehub.platform.api.preferences.PreferenceProvider;
import jakarta.enterprise.inject.Instance;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

class DefaultCapabilityHealthBehavioralViolationTest {

    StubBehavioralSignalStore signalStore;
    DefaultCapabilityHealth health;

    @BeforeEach
    void setUp() {
        signalStore = new StubBehavioralSignalStore();
        @SuppressWarnings("unchecked")
        Instance<PreferenceProvider> emptyProvider = mock(Instance.class);
        when(emptyProvider.isUnsatisfied()).thenReturn(true);
        health = new DefaultCapabilityHealth(0.3, mock(AgentStateStore.class),
                signalStore, emptyProvider, new StubVocabularyRegistry());
    }

    private AgentDescriptor agent(String id, String capabilityName) {
        return AgentDescriptor.builder()
                .agentId(id).name("Test").tenancyId("default")
                .capabilities(List.of(
                        AgentCapability.builder().name(capabilityName).build()))
                .build();
    }

    @Test
    void violations_below_threshold_returns_ready() {
        signalStore.setViolations("a1", "default", "code-review",
                Map.of("latency", 2));
        var status = health.probe(agent("a1", "code-review"), "code-review",
                ProbeContext.of("java"));
        assertThat(status).isInstanceOf(CapabilityStatus.Ready.class);
    }

    @Test
    void violations_at_threshold_returns_behavioral_violation() {
        signalStore.setViolations("a1", "default", "code-review",
                Map.of("latency", 3));
        var status = health.probe(agent("a1", "code-review"), "code-review",
                ProbeContext.of("java"));
        assertThat(status).isInstanceOf(CapabilityStatus.BehavioralViolation.class);
        var violation = (CapabilityStatus.BehavioralViolation) status;
        assertThat(violation.violations()).containsEntry("latency", 3);
    }

    @Test
    void multiple_dimensions_above_threshold_all_returned() {
        signalStore.setViolations("a1", "default", "code-review",
                Map.of("latency", 5, "attestation-rate", 4));
        var status = health.probe(agent("a1", "code-review"), "code-review",
                ProbeContext.of("java"));
        assertThat(status).isInstanceOf(CapabilityStatus.BehavioralViolation.class);
        var violation = (CapabilityStatus.BehavioralViolation) status;
        assertThat(violation.violations()).hasSize(2)
                .containsEntry("latency", 5)
                .containsEntry("attestation-rate", 4);
    }

    @Test
    void mixed_dimensions_only_above_threshold_returned() {
        signalStore.setViolations("a1", "default", "code-review",
                Map.of("latency", 5, "attestation-rate", 1));
        var status = health.probe(agent("a1", "code-review"), "code-review",
                ProbeContext.of("java"));
        assertThat(status).isInstanceOf(CapabilityStatus.BehavioralViolation.class);
        var violation = (CapabilityStatus.BehavioralViolation) status;
        assertThat(violation.violations()).hasSize(1).containsEntry("latency", 5);
    }

    @Test
    void null_task_domain_still_checks_compliance() {
        signalStore.setViolations("a1", "default", "code-review",
                Map.of("latency", 3));
        var status = health.probe(agent("a1", "code-review"), "code-review",
                ProbeContext.of(null));
        assertThat(status).isInstanceOf(CapabilityStatus.BehavioralViolation.class);
    }

    @Test
    void no_violations_recorded_returns_ready() {
        var status = health.probe(agent("a1", "code-review"), "code-review",
                ProbeContext.of("java"));
        assertThat(status).isInstanceOf(CapabilityStatus.Ready.class);
    }

    @Test
    void uses_resolved_capability_name_not_query_tag() {
        signalStore.setViolations("a1", "default", "code-review",
                Map.of("latency", 3));
        var status = health.probe(agent("a1", "code-review"), "review",
                ProbeContext.of("java"));
        // "review" doesn't match "code-review" exactly — agent has no "review" capability
        // so probe returns Unavailable before reaching Step 6
        assertThat(status).isInstanceOf(CapabilityStatus.Unavailable.class);
    }

    // --- Stubs ---

    static class StubBehavioralSignalStore implements BehavioralSignalStore {
        private final Map<String, Map<String, Integer>> violationData = new java.util.HashMap<>();

        void setViolations(String agentId, String tenancyId, String capabilityName,
                           Map<String, Integer> violations) {
            violationData.put(agentId + "|" + tenancyId + "|" + capabilityName, violations);
        }

        @Override public void record(String a, String t, String c, String q, BehavioralSignal s) {}
        @Override public void clear(String a, String t, String c, BehavioralSignal s) {}
        @Override public Map<String, Integer> learned(String a, String t, String c, BehavioralSignal s) {
            if (s != BehavioralSignal.VIOLATED) return Map.of();
            return violationData.getOrDefault(a + "|" + t + "|" + c, Map.of());
        }
        @Override public int count(String a, String t, String c, String q, BehavioralSignal s) { return 0; }
    }

    static class StubVocabularyRegistry implements VocabularyRegistry {
        @Override public <T extends Enum<T> & VocabularyTerm> void register(Class<T> c) {}
        @Override public boolean isRegistered(String uri) { return false; }
        @Override public java.util.Optional<VocabularyTerm> resolve(String uri, String v) { return java.util.Optional.empty(); }
        @Override public java.util.List<VocabularyTerm> allTerms(String uri) { return List.of(); }
        @Override public java.util.List<String> equivalentValues(String v) { return List.of(v); }
        @Override public <T extends Enum<T> & VocabularyTerm> java.util.List<String> equivalentValues(Class<T> c, T t) { return List.of(); }
        @Override public java.util.List<String> equivalentValues(DispositionAxis a, String v) { return List.of(v); }
        @Override public boolean subsumes(String uri, String ancestor, String descendant) { return false; }
        @Override public MatchDegree match(String uri, String registered, String query) { return MatchDegree.NONE; }
        @Override public java.util.Set<String> ancestors(String uri, String term) { return java.util.Set.of(); }
        @Override public java.util.Set<String> descendants(String uri, String term) { return java.util.Set.of(); }
        @Override public Map<String, java.util.Set<String>> expandForMatchingByVocabulary(String termName) { return Map.of(); }
    }
}
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=DefaultCapabilityHealthBehavioralViolationTest`
Expected: FAIL — Step 6 not implemented yet, constructor signature may not match.

- [ ] **Step 5: Update DefaultCapabilityHealth — rename field, add Step 6**

Update `runtime/src/main/java/io/casehub/eidos/runtime/health/DefaultCapabilityHealth.java`:

1. Change field `CapabilitySpecializationStore specializationStore` → `BehavioralSignalStore signalStore`
2. Update constructor parameter to match
3. Update imports
4. Add Step 6 after Step 5 (epistemic weakness), before returning Ready
5. Add `complianceViolationThreshold` method

The complete updated probe method body after Step 5:

```java
// Step 6: behavioral compliance
final var violations = signalStore.learned(
    descriptor.agentId(), descriptor.tenancyId(), capability.name(),
    BehavioralSignal.VIOLATED);
if (!violations.isEmpty()) {
    final int threshold = complianceViolationThreshold(descriptor.tenancyId());
    final var exceeding = new java.util.LinkedHashMap<String, Integer>();
    violations.forEach((dimension, count) -> {
        if (count >= threshold) exceeding.put(dimension, count);
    });
    if (!exceeding.isEmpty()) {
        return new CapabilityStatus.BehavioralViolation(Map.copyOf(exceeding));
    }
}

return new CapabilityStatus.Ready();
```

Add the threshold resolution method (same pattern as `excludeThreshold`):

```java
private int complianceViolationThreshold(final String tenancyId) {
    if (preferenceProviderInstance.isUnsatisfied()) {
        return EidosPreferenceKeys.COMPLIANCE_VIOLATION_THRESHOLD.defaultValue().value();
    }
    return preferenceProviderInstance.get()
        .resolve(SettingsScope.of(tenancyId))
        .getOrDefault(EidosPreferenceKeys.COMPLIANCE_VIOLATION_THRESHOLD).value();
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=DefaultCapabilityHealthBehavioralViolationTest`
Expected: PASS

- [ ] **Step 7: Commit**

```
feat(eidos#85): add probe Step 6 — behavioral compliance checking with configurable violation threshold
```

---

### Task 5: Update existing tests and add ComplianceAttestations

Update all remaining test files for the rename, add ComplianceAttestations utility, and add integration example.

**Files:**
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/health/DefaultCapabilityHealthExclusionTest.java`
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/health/DefaultCapabilityHealthDegradedTest.java`
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/health/DefaultCapabilityHealthTest.java` (if exists)
- Delete + Create: `runtime/src/test/java/io/casehub/eidos/runtime/health/jpa/JpaCapabilitySpecializationStoreTest.java` → `JpaBehavioralSignalStoreTest.java`
- Modify: `examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/LearnedExclusionSubsumptionTest.java`
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/health/ComplianceAttestations.java`
- Create: `runtime/src/test/java/io/casehub/eidos/runtime/health/ComplianceAttestationsTest.java`

**Interfaces:**
- Consumes: `BehavioralSignal`, `BehavioralSignalStore`, `ComplianceDimension`
- Consumes: `LedgerAttestation`, `AttestationVerdict`, `ActorType` (from ledger-api)
- Produces: `ComplianceAttestations.violation()`, `.compliance()`

- [ ] **Step 1: Update DefaultCapabilityHealthExclusionTest**

In `runtime/src/test/java/io/casehub/eidos/runtime/health/DefaultCapabilityHealthExclusionTest.java`:
- Replace all `SpecializationSignal` imports with `BehavioralSignal`
- Replace all `CapabilitySpecializationStore` with `BehavioralSignalStore`
- Rename stub class `StubSpecializationStore` → `StubBehavioralSignalStore`
- Rename parameter `domain` → `qualifier` in stub method signatures
- Replace `SpecializationSignal.DECLINE` → `BehavioralSignal.DECLINE`, `SpecializationSignal.SUCCESS` → `BehavioralSignal.SUCCESS`

- [ ] **Step 2: Update DefaultCapabilityHealthDegradedTest**

In `runtime/src/test/java/io/casehub/eidos/runtime/health/DefaultCapabilityHealthDegradedTest.java`:
- Same rename pattern as Step 1 for the `NoOpSpecializationStore` inner class

- [ ] **Step 3: Rename JPA store test**

Delete `runtime/src/test/java/io/casehub/eidos/runtime/health/jpa/JpaCapabilitySpecializationStoreTest.java`.
Create `runtime/src/test/java/io/casehub/eidos/runtime/health/jpa/JpaBehavioralSignalStoreTest.java` with updated type references (`BehavioralSignalStore`, `BehavioralSignal.DECLINE`, `BehavioralSignal.SUCCESS`). Add COMPLIANT and VIOLATED signal tests matching the InMemory test additions from Task 3 Step 8.

- [ ] **Step 4: Update LearnedExclusionSubsumptionTest**

In `examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/LearnedExclusionSubsumptionTest.java`:
- Replace `CapabilitySpecializationStore` → `BehavioralSignalStore`
- Replace `SpecializationSignal.DECLINE` → `BehavioralSignal.DECLINE`
- Update `qualifier` parameter name in `store.record()` calls (the value `"rust"` stays the same)

- [ ] **Step 5: Write failing test for ComplianceAttestations**

```java
// runtime/src/test/java/io/casehub/eidos/runtime/health/ComplianceAttestationsTest.java
package io.casehub.eidos.runtime.health;

import io.casehub.eidos.api.ComplianceDimension;
import io.casehub.ledger.api.model.AttestationVerdict;
import io.casehub.platform.api.identity.ActorType;
import org.junit.jupiter.api.Test;

import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

class ComplianceAttestationsTest {

    @Test
    void violation_creates_flagged_attestation() {
        var entryId = UUID.randomUUID();
        var subjectId = UUID.randomUUID();
        var att = ComplianceAttestations.violation(entryId, subjectId,
                "code-review", "latency", "28500ms exceeded 5000ms p50", 0.0);

        assertThat(att.attestorId).isEqualTo(ComplianceDimension.ATTESTOR_ID);
        assertThat(att.attestorType).isEqualTo(ActorType.SYSTEM);
        assertThat(att.verdict).isEqualTo(AttestationVerdict.FLAGGED);
        assertThat(att.confidence).isEqualTo(1.0);
        assertThat(att.capabilityTag).isEqualTo("code-review");
        assertThat(att.trustDimension).isEqualTo("behavioral:latency");
        assertThat(att.dimensionScore).isEqualTo(0.0);
        assertThat(att.evidence).isEqualTo("28500ms exceeded 5000ms p50");
        assertThat(att.ledgerEntryId).isEqualTo(entryId);
        assertThat(att.subjectId).isEqualTo(subjectId);
    }

    @Test
    void compliance_creates_sound_attestation() {
        var entryId = UUID.randomUUID();
        var subjectId = UUID.randomUUID();
        var att = ComplianceAttestations.compliance(entryId, subjectId,
                "code-review", "latency", 1.0);

        assertThat(att.verdict).isEqualTo(AttestationVerdict.SOUND);
        assertThat(att.trustDimension).isEqualTo("behavioral:latency");
        assertThat(att.dimensionScore).isEqualTo(1.0);
    }
}
```

- [ ] **Step 6: Implement ComplianceAttestations**

```java
// runtime/src/main/java/io/casehub/eidos/runtime/health/ComplianceAttestations.java
package io.casehub.eidos.runtime.health;

import io.casehub.eidos.api.ComplianceDimension;
import io.casehub.ledger.api.model.AttestationVerdict;
import io.casehub.ledger.api.model.LedgerAttestation;
import io.casehub.platform.api.identity.ActorType;

import java.time.Instant;
import java.util.UUID;

public final class ComplianceAttestations {

    private ComplianceAttestations() {}

    public static LedgerAttestation violation(
            final UUID ledgerEntryId, final UUID subjectId,
            final String capabilityTag, final String dimension,
            final String evidence, final double dimensionScore) {
        return build(ledgerEntryId, subjectId, capabilityTag, dimension,
                AttestationVerdict.FLAGGED, evidence, dimensionScore);
    }

    public static LedgerAttestation compliance(
            final UUID ledgerEntryId, final UUID subjectId,
            final String capabilityTag, final String dimension,
            final double dimensionScore) {
        return build(ledgerEntryId, subjectId, capabilityTag, dimension,
                AttestationVerdict.SOUND, null, dimensionScore);
    }

    private static LedgerAttestation build(
            final UUID ledgerEntryId, final UUID subjectId,
            final String capabilityTag, final String dimension,
            final AttestationVerdict verdict, final String evidence,
            final double dimensionScore) {
        final var att = new LedgerAttestation();
        att.id = UUID.randomUUID();
        att.ledgerEntryId = ledgerEntryId;
        att.subjectId = subjectId;
        att.attestorId = ComplianceDimension.ATTESTOR_ID;
        att.attestorType = ActorType.SYSTEM;
        att.verdict = verdict;
        att.evidence = evidence;
        att.confidence = 1.0;
        att.capabilityTag = capabilityTag;
        att.trustDimension = ComplianceDimension.TRUST_DIMENSION_PREFIX + dimension;
        att.dimensionScore = dimensionScore;
        att.occurredAt = Instant.now();
        return att;
    }
}
```

- [ ] **Step 7: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
Expected: PASS across all modules

- [ ] **Step 8: Commit**

```
feat(eidos#85): update tests for BehavioralSignalStore rename, add ComplianceAttestations utility
```

---

### Task 6: Final verification and CLAUDE.md update

Full build, update CLAUDE.md documentation, verify coherence.

**Files:**
- Modify: `CLAUDE.md` (project repo)

- [ ] **Step 1: Full clean build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS, all tests pass

- [ ] **Step 2: Update CLAUDE.md**

Update the following sections in `CLAUDE.md`:

In the **CapabilityHealth** bullet:
- Update probe step list to include Step 6: BehavioralViolation
- Update the variant count (6 variants now)

In the **CapabilitySpecializationStore** references:
- Replace `CapabilitySpecializationStore` with `BehavioralSignalStore`
- Replace `SpecializationSignal { DECLINE, SUCCESS }` with `BehavioralSignal { DECLINE, SUCCESS, COMPLIANT, VIOLATED }`
- Update config property names
- Add `ComplianceDimension` and `ComplianceAttestations` to the relevant module listing

In the **Key Design Decisions** section:
- Add: behavioral contracts / compliance checking description

- [ ] **Step 3: Commit**

```
docs(eidos#85): update CLAUDE.md for BehavioralSignalStore, probe Step 6, and compliance framework
```

- [ ] **Step 4: Run tests one final time after CLAUDE.md commit**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
Expected: PASS — documentation-only change, no code affected
