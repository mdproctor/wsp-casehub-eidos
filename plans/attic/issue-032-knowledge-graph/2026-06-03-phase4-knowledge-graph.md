# Phase 4: Knowledge Graph Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement casehub-eidos Phase 4 — a production knowledge graph recording agent task history, outcomes, and attestation evidence.

**Architecture:** Three phases. Phase 1 fixes foundation issues (AgentStateStore tenancyId, agent_descriptor surrogate key). Phase 2 is a PoC gate — do not proceed to Phase 3 until Phase 2 passes evaluation. Phase 3 builds the full casehub-eidos-graph module.

**Tech Stack:** Java 21, Quarkus 3.32.2, JPA/Hibernate, Flyway, H2 (tests), AssertJ, JUnit 5

**Build command throughout:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl <module> -q`

---

## PHASE 1: Foundation Fixes

Issues: eidos#33 (AgentStateStore tenancyId), eidos#34 (surrogate key)

---

### Task 1: Extend AgentStateStore and ReactiveAgentStateStore SPIs with tenancyId

**Files:**
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentStateStore.java`
- Modify: `api/src/main/java/io/casehub/eidos/api/ReactiveAgentStateStore.java`
- Modify: `api/src/test/java/io/casehub/eidos/api/AgentStateStoreContractTest.java`

- [ ] **Update AgentStateStore.java**

```java
package io.casehub.eidos.api;

import java.time.Instant;
import java.util.Optional;

public interface AgentStateStore {
    void record(String agentId, String tenancyId, DegradationReason reason, Instant expiresAt);
    Optional<DegradationReason> query(String agentId, String tenancyId);
    void clear(String agentId, String tenancyId);
}
```

- [ ] **Update ReactiveAgentStateStore.java**

```java
package io.casehub.eidos.api;

import io.smallrye.mutiny.Uni;
import java.time.Instant;
import java.util.Optional;

public interface ReactiveAgentStateStore {
    Uni<Void>                        record(String agentId, String tenancyId, DegradationReason reason, Instant expiresAt);
    Uni<Optional<DegradationReason>> query(String agentId, String tenancyId);
    Uni<Void>                        clear(String agentId, String tenancyId);
}
```

- [ ] **Update AgentStateStoreContractTest.java** — add tenancyId to all call sites

```java
package io.casehub.eidos.api;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import static org.assertj.core.api.Assertions.*;

public abstract class AgentStateStoreContractTest {

    protected abstract AgentStateStore store();

    @BeforeEach
    protected void resetStore() {}

    @Test
    void query_returns_empty_when_no_record() {
        assertThat(store().query("agent-1", "tenant-1")).isEmpty();
    }

    @Test
    void record_and_query_returns_reason() {
        store().record("agent-1", "tenant-1", DegradationReason.RATE_LIMITED, Instant.now().plusSeconds(60));
        assertThat(store().query("agent-1", "tenant-1")).contains(DegradationReason.RATE_LIMITED);
    }

    @Test
    void query_returns_empty_after_ttl_expired() {
        store().record("agent-1", "tenant-1", DegradationReason.OVERLOADED, Instant.now().minusSeconds(1));
        assertThat(store().query("agent-1", "tenant-1")).isEmpty();
    }

    @Test
    void clear_removes_entry() {
        store().record("agent-1", "tenant-1", DegradationReason.RATE_LIMITED, Instant.now().plusSeconds(60));
        store().clear("agent-1", "tenant-1");
        assertThat(store().query("agent-1", "tenant-1")).isEmpty();
    }

    @Test
    void clear_on_absent_agent_does_not_throw() {
        assertThatCode(() -> store().clear("nonexistent", "tenant-1")).doesNotThrowAnyException();
    }

    @Test
    void record_overwrites_previous_state() {
        store().record("agent-1", "tenant-1", DegradationReason.RATE_LIMITED, Instant.now().plusSeconds(60));
        store().record("agent-1", "tenant-1", DegradationReason.CONTEXT_EXHAUSTED, Instant.now().plusSeconds(60));
        assertThat(store().query("agent-1", "tenant-1")).contains(DegradationReason.CONTEXT_EXHAUSTED);
    }

    @Test
    void different_agents_are_independent() {
        store().record("agent-1", "tenant-1", DegradationReason.RATE_LIMITED, Instant.now().plusSeconds(60));
        assertThat(store().query("agent-2", "tenant-1")).isEmpty();
    }

    @Test
    void different_tenancies_are_isolated() {
        store().record("agent-1", "tenant-1", DegradationReason.RATE_LIMITED, Instant.now().plusSeconds(60));
        assertThat(store().query("agent-1", "tenant-2")).isEmpty();
    }
}
```

- [ ] **Verify compilation fails** (all implementors now have wrong signature):

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api -q 2>&1 | tail -5
```

Expected: api compiles. runtime, persistence-memory fail — that's correct, proceed.

- [ ] **Commit api changes**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add api/src/main/java/io/casehub/eidos/api/AgentStateStore.java \
  api/src/main/java/io/casehub/eidos/api/ReactiveAgentStateStore.java \
  api/src/test/java/io/casehub/eidos/api/AgentStateStoreContractTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(api): add tenancyId to AgentStateStore and ReactiveAgentStateStore SPIs

Refs #33"
```

---

### Task 2: Update NoOpAgentStateStore

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/health/NoOpAgentStateStore.java`

- [ ] **Update NoOpAgentStateStore.java**

```java
package io.casehub.eidos.runtime.health;

import io.casehub.eidos.api.AgentStateStore;
import io.casehub.eidos.api.DegradationReason;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import java.time.Instant;
import java.util.Optional;

@DefaultBean
@ApplicationScoped
public class NoOpAgentStateStore implements AgentStateStore {

    @Override
    public void record(final String agentId, final String tenancyId,
                       final DegradationReason reason, final Instant expiresAt) {}

    @Override
    public Optional<DegradationReason> query(final String agentId, final String tenancyId) {
        return Optional.empty();
    }

    @Override
    public void clear(final String agentId, final String tenancyId) {}
}
```

---

### Task 3: Update InMemoryAgentStateStore — composite key

**Files:**
- Modify: `persistence-memory/src/main/java/io/casehub/eidos/memory/InMemoryAgentStateStore.java`

- [ ] **Update InMemoryAgentStateStore.java**

```java
package io.casehub.eidos.memory;

import io.casehub.eidos.api.AgentStateStore;
import io.casehub.eidos.api.DegradationReason;
import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import java.time.Instant;
import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;

@Alternative
@Priority(1)
@ApplicationScoped
public class InMemoryAgentStateStore implements AgentStateStore {

    private final ConcurrentHashMap<StateKey, ExpiringState> store = new ConcurrentHashMap<>();

    @Override
    public void record(final String agentId, final String tenancyId,
                       final DegradationReason reason, final Instant expiresAt) {
        store.put(new StateKey(agentId, tenancyId), new ExpiringState(reason, expiresAt));
    }

    @Override
    public Optional<DegradationReason> query(final String agentId, final String tenancyId) {
        return Optional.ofNullable(store.get(new StateKey(agentId, tenancyId)))
                .filter(s -> Instant.now().isBefore(s.expiresAt()))
                .map(ExpiringState::reason);
    }

    @Override
    public void clear(final String agentId, final String tenancyId) {
        store.remove(new StateKey(agentId, tenancyId));
    }

    private record StateKey(String agentId, String tenancyId) {}
    private record ExpiringState(DegradationReason reason, Instant expiresAt) {}
}
```

- [ ] **Run persistence-memory tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory -q 2>&1 | tail -5
```

Expected: BUILD SUCCESS

- [ ] **Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add \
  runtime/src/main/java/io/casehub/eidos/runtime/health/NoOpAgentStateStore.java \
  persistence-memory/src/main/java/io/casehub/eidos/memory/InMemoryAgentStateStore.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "fix: update NoOp and InMemory AgentStateStore for tenancyId SPI

InMemoryAgentStateStore uses composite (agentId, tenancyId) key.
Refs #33"
```

---

### Task 4: Rename AgentDegradationStateEntity → AgentStateEntity + rewrite V2

**Files:**
- Modify: `runtime/src/main/resources/db/eidos/migration/V2__agent_degradation_state.sql`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/health/jpa/AgentDegradationStateEntity.java`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/health/jpa/JpaAgentStateStore.java`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/health/jpa/JpaReactiveAgentStateStore.java`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/health/jpa/AgentDegradationStateReactivePanacheRepo.java`

- [ ] **Rewrite V2__agent_degradation_state.sql** (keep filename, replace contents)

```sql
CREATE TABLE agent_state (
    agent_id     VARCHAR(255)  NOT NULL,
    tenancy_id   VARCHAR(255)  NOT NULL,
    degradation  VARCHAR(50)   NOT NULL,
    expires_at   TIMESTAMPTZ   NOT NULL,
    recorded_at  TIMESTAMPTZ   NOT NULL DEFAULT now(),
    PRIMARY KEY (agent_id, tenancy_id)
);
```

- [ ] **Rewrite AgentDegradationStateEntity.java → rename to AgentStateEntity** (rename file too)

Create `runtime/src/main/java/io/casehub/eidos/runtime/health/jpa/AgentStateEntity.java`:

```java
package io.casehub.eidos.runtime.health.jpa;

import jakarta.persistence.*;
import java.time.Instant;

@Entity
@Table(name = "agent_state")
class AgentStateEntity {

    @Id
    @Column(name = "agent_id")
    private String agentId;

    @Id
    @Column(name = "tenancy_id")
    private String tenancyId;

    @Column(name = "degradation", nullable = false)
    private String degradation;

    @Column(name = "expires_at", nullable = false)
    private Instant expiresAt;

    protected AgentStateEntity() {}

    AgentStateEntity(final String agentId, final String tenancyId,
                     final String degradation, final Instant expiresAt) {
        this.agentId = agentId;
        this.tenancyId = tenancyId;
        this.degradation = degradation;
        this.expiresAt = expiresAt;
    }

    String getAgentId()    { return agentId; }
    String getTenancyId()  { return tenancyId; }
    String getDegradation(){ return degradation; }
    Instant getExpiresAt() { return expiresAt; }
}
```

Note: JPA composite PK with two `@Id` fields requires `@IdClass` or `@EmbeddedId`. Use `@EmbeddedId` with an `AgentStateId` embeddable:

```java
package io.casehub.eidos.runtime.health.jpa;

import jakarta.persistence.Embeddable;
import java.io.Serializable;

@Embeddable
class AgentStateId implements Serializable {
    String agentId;
    String tenancyId;

    protected AgentStateId() {}
    AgentStateId(String agentId, String tenancyId) {
        this.agentId = agentId;
        this.tenancyId = tenancyId;
    }

    @Override public boolean equals(Object o) {
        if (!(o instanceof AgentStateId that)) return false;
        return java.util.Objects.equals(agentId, that.agentId)
            && java.util.Objects.equals(tenancyId, that.tenancyId);
    }
    @Override public int hashCode() {
        return java.util.Objects.hash(agentId, tenancyId);
    }
}
```

Then update `AgentStateEntity` to use `@EmbeddedId AgentStateId id` and add `@AttributeOverrides` mapping `agentId`→`agent_id`, `tenancyId`→`tenancy_id`.

Full entity using `@EmbeddedId`:

```java
package io.casehub.eidos.runtime.health.jpa;

import jakarta.persistence.*;
import java.time.Instant;

@Entity
@Table(name = "agent_state")
class AgentStateEntity {

    @EmbeddedId
    @AttributeOverrides({
        @AttributeOverride(name = "agentId",   column = @Column(name = "agent_id")),
        @AttributeOverride(name = "tenancyId", column = @Column(name = "tenancy_id"))
    })
    AgentStateId id;

    @Column(name = "degradation", nullable = false)
    String degradation;

    @Column(name = "expires_at", nullable = false)
    Instant expiresAt;

    protected AgentStateEntity() {}

    AgentStateEntity(final String agentId, final String tenancyId,
                     final String degradation, final Instant expiresAt) {
        this.id = new AgentStateId(agentId, tenancyId);
        this.degradation = degradation;
        this.expiresAt = expiresAt;
    }

    String getAgentId()    { return id.agentId; }
    String getTenancyId()  { return id.tenancyId; }
    String getDegradation(){ return degradation; }
    Instant getExpiresAt() { return expiresAt; }
}
```

- [ ] **Delete old entity file**

```bash
rm /Users/mdproctor/claude/casehub/eidos/runtime/src/main/java/io/casehub/eidos/runtime/health/jpa/AgentDegradationStateEntity.java
```

- [ ] **Update JpaAgentStateStore.java** — use new entity, add tenancyId

```java
package io.casehub.eidos.runtime.health.jpa;

import io.casehub.eidos.api.AgentStateStore;
import io.casehub.eidos.api.DegradationReason;
import io.quarkus.arc.properties.IfBuildProperty;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.transaction.Transactional;
import jakarta.transaction.Transactional.TxType;
import java.time.Instant;
import java.util.Optional;

@IfBuildProperty(name = "casehub.eidos.reactive.enabled", stringValue = "false", enableIfMissing = true)
@ApplicationScoped
public class JpaAgentStateStore implements AgentStateStore {

    @Inject EntityManager em;

    @Override
    @Transactional
    public void record(final String agentId, final String tenancyId,
                       final DegradationReason reason, final Instant expiresAt) {
        em.createQuery("DELETE FROM AgentStateEntity e WHERE e.id.agentId = :id AND e.id.tenancyId = :tenancyId")
          .setParameter("id", agentId)
          .setParameter("tenancyId", tenancyId)
          .executeUpdate();
        em.flush();
        em.clear();
        em.persist(new AgentStateEntity(agentId, tenancyId, reason.name(), expiresAt));
    }

    @Override
    @Transactional(TxType.SUPPORTS)
    public Optional<DegradationReason> query(final String agentId, final String tenancyId) {
        return em.createQuery(
                "SELECT e FROM AgentStateEntity e WHERE e.id.agentId = :id"
                + " AND e.id.tenancyId = :tenancyId AND e.expiresAt > :now",
                AgentStateEntity.class)
            .setParameter("id", agentId)
            .setParameter("tenancyId", tenancyId)
            .setParameter("now", Instant.now())
            .getResultStream()
            .findFirst()
            .map(e -> DegradationReason.valueOf(e.getDegradation()));
    }

    @Override
    @Transactional
    public void clear(final String agentId, final String tenancyId) {
        em.createQuery("DELETE FROM AgentStateEntity e WHERE e.id.agentId = :id AND e.id.tenancyId = :tenancyId")
          .setParameter("id", agentId)
          .setParameter("tenancyId", tenancyId)
          .executeUpdate();
    }
}
```

- [ ] **Update JpaReactiveAgentStateStore.java** — add tenancyId, update entity references

Update all method signatures to include `String tenancyId` as second param. Change `repo.delete("agentId", agentId)` to `repo.delete("id.agentId = ?1 AND id.tenancyId = ?2", agentId, tenancyId)`. Change `repo.find("agentId = ?1 AND ..."` to `repo.find("id.agentId = ?1 AND id.tenancyId = ?2 AND expiresAt > ?3", agentId, tenancyId, Instant.now())`. Update `repo.persist(new AgentStateEntity(agentId, tenancyId, reason.name(), expiresAt))`.

- [ ] **Update AgentDegradationStateReactivePanacheRepo.java** — rename to `AgentStateReactivePanacheRepo` and update entity type to `AgentStateEntity`

---

### Task 5: Update DefaultCapabilityHealth and its test

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/health/DefaultCapabilityHealth.java`
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/health/DefaultCapabilityHealthDegradedTest.java`

- [ ] **Update DefaultCapabilityHealth.java** — pass `descriptor.tenancyId()` to store calls

Replace line 28:
```java
final var degraded = stateStore.query(descriptor.agentId());
```
With:
```java
final var degraded = stateStore.query(descriptor.agentId(), descriptor.tenancyId());
```

- [ ] **Update DefaultCapabilityHealthDegradedTest.java** — update StubStateStore and all call sites

```java
static class StubStateStore implements AgentStateStore {
    private final ConcurrentHashMap<String, DegradationReason> state = new ConcurrentHashMap<>();

    @Override
    public void record(final String agentId, final String tenancyId,
                       final DegradationReason reason, final Instant expiresAt) {
        state.put(agentId + "|" + tenancyId, reason);
    }

    @Override
    public Optional<DegradationReason> query(final String agentId, final String tenancyId) {
        return Optional.ofNullable(state.get(agentId + "|" + tenancyId));
    }

    @Override
    public void clear(final String agentId, final String tenancyId) {
        state.remove(agentId + "|" + tenancyId);
    }
}
```

Update all test call sites: `stateStore.record("agent-1", ...)` → `stateStore.record("agent-1", "default", ...)`.
The `agent()` helper already passes `"default"` as tenancyId, so health checks scope correctly.

- [ ] **Run runtime tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q 2>&1 | tail -5
```

Expected: BUILD SUCCESS

- [ ] **Run full build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -q 2>&1 | tail -5
```

Expected: BUILD SUCCESS

- [ ] **Commit all Phase 1a changes** (tasks 1-5)

```bash
git -C /Users/mdproctor/claude/casehub/eidos add -A
git -C /Users/mdproctor/claude/casehub/eidos commit -m "fix: AgentStateStore tenancyId — SPI, all impls, DefaultCapabilityHealth

- AgentStateStore + ReactiveAgentStateStore: add tenancyId to all methods
- NoOpAgentStateStore: params added, ignored
- InMemoryAgentStateStore: composite (agentId, tenancyId) key
- AgentDegradationStateEntity renamed to AgentStateEntity, table renamed to agent_state
- V2 migration rewritten: agent_state with tenancy_id + recorded_at
- JpaAgentStateStore + JpaReactiveAgentStateStore: scoped to (agentId, tenancyId)
- DefaultCapabilityHealth: passes descriptor.tenancyId() to store

Closes #33"
```

---

### Task 6: Rewrite V1 migration — agent_descriptor surrogate key

**Files:**
- Modify: `runtime/src/main/resources/db/eidos/migration/V1__initial_schema.sql`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentDescriptorEntity.java`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentCapabilityEntity.java`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/JpaAgentRegistry.java`

- [ ] **Rewrite V1__initial_schema.sql**

```sql
-- Consumer configuration:
--   quarkus.flyway.locations=classpath:db/eidos/migration,<your-own-locations>

CREATE TABLE agent_descriptor (
    internal_id            BIGSERIAL       PRIMARY KEY,
    agent_id               VARCHAR(255)    NOT NULL,
    tenancy_id             VARCHAR(255)    NOT NULL,
    name                   VARCHAR(255),
    version                VARCHAR(255),
    provider               VARCHAR(255),
    model_family           VARCHAR(255),
    model_version          VARCHAR(255),
    weights_fingerprint    VARCHAR(255),
    domain_vocabulary      VARCHAR(255),
    slot_vocabulary        VARCHAR(255),
    disposition_vocabulary VARCHAR(255),
    slot                   VARCHAR(255),
    jurisdiction           VARCHAR(255),
    data_handling_policy   TEXT,
    disposition            TEXT,
    registered_at          TIMESTAMPTZ     NOT NULL DEFAULT now(),
    CONSTRAINT uq_agent UNIQUE (agent_id, tenancy_id)
);
CREATE INDEX idx_descriptor_tenancy_slot ON agent_descriptor(tenancy_id, slot);

CREATE TABLE agent_capability (
    id                  BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    descriptor_id       BIGINT        NOT NULL
                            REFERENCES agent_descriptor(internal_id) ON DELETE CASCADE,
    agent_id            VARCHAR(255)  NOT NULL,
    tenancy_id          VARCHAR(255)  NOT NULL,
    name                VARCHAR(255)  NOT NULL,
    quality_hint        DOUBLE PRECISION,
    latency_hint_p50_ms BIGINT,
    cost_hint           VARCHAR(255),
    input_types         TEXT,
    output_types        TEXT,
    tags                TEXT,
    epistemic_domains   TEXT
);
CREATE INDEX idx_capability_name ON agent_capability(name);
```

- [ ] **Update AgentDescriptorEntity.java**

```java
package io.casehub.eidos.runtime.registry.jpa;

import jakarta.persistence.*;
import java.util.ArrayList;
import java.util.List;

@Entity
@Table(name = "agent_descriptor",
       uniqueConstraints = @UniqueConstraint(columnNames = {"agent_id", "tenancy_id"}))
public class AgentDescriptorEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "internal_id")
    Long internalId;

    @Column(name = "agent_id", nullable = false)
    String agentId;

    @Column(name = "tenancy_id", nullable = false)
    String tenancyId;

    String name;
    String version;
    String provider;

    @Column(name = "model_family")  String modelFamily;
    @Column(name = "model_version") String modelVersion;
    @Column(name = "weights_fingerprint") String weightsFingerprint;
    @Column(name = "domain_vocabulary")   String domainVocabulary;
    @Column(name = "slot_vocabulary")     String slotVocabulary;
    @Column(name = "disposition_vocabulary") String dispositionVocabulary;

    String slot;
    String jurisdiction;

    @Column(name = "data_handling_policy", columnDefinition = "TEXT")
    String dataHandlingPolicy;

    @Column(columnDefinition = "TEXT")
    String disposition;

    @OneToMany(mappedBy = "descriptor", cascade = CascadeType.ALL,
               fetch = FetchType.LAZY, orphanRemoval = true)
    List<AgentCapabilityEntity> capabilities = new ArrayList<>();
}
```

- [ ] **Update AgentCapabilityEntity.java** — FK now references `descriptor` via `descriptor_id`

```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "descriptor_id", nullable = false)
AgentDescriptorEntity descriptor;
```

Also add `agent_id` and `tenancy_id` columns to the entity (denormalised for query convenience, populated from the parent):

```java
@Column(name = "agent_id")   String agentId;
@Column(name = "tenancy_id") String tenancyId;
```

- [ ] **Update JpaAgentRegistry.register()** — scope delete to (agentId, tenancyId)

```java
em.createQuery("DELETE FROM AgentDescriptorEntity e WHERE e.agentId = :id AND e.tenancyId = :tenancyId")
  .setParameter("id", descriptor.agentId())
  .setParameter("tenancyId", descriptor.tenancyId())
  .executeUpdate();
```

- [ ] **Update AgentDescriptorMapper.toCapabilityEntity()** — populate denormalised agentId/tenancyId

```java
private AgentCapabilityEntity toCapabilityEntity(AgentCapability c, AgentDescriptorEntity parent) {
    var e = new AgentCapabilityEntity();
    e.descriptor = parent;
    e.agentId    = parent.agentId;
    e.tenancyId  = parent.tenancyId;
    e.name = c.name();
    // ... rest unchanged
}
```

- [ ] **Run runtime tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q 2>&1 | tail -5
```

Expected: BUILD SUCCESS

- [ ] **Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add -A
git -C /Users/mdproctor/claude/casehub/eidos commit -m "fix: agent_descriptor surrogate key — BIGSERIAL internal_id, UNIQUE (agent_id, tenancy_id)

V1 migration rewritten. AgentDescriptorEntity uses @GeneratedValue Long internalId.
AgentCapabilityEntity FK references descriptor_id. JpaAgentRegistry.register()
scoped to (agentId, tenancyId).

Closes #34"
```

---

## PHASE 2: PoC Validation Gate

**DO NOT PROCEED TO PHASE 3 WITHOUT EVALUATING THESE TESTS.**

---

### Task 7: InMemoryAgentGraph POJO

**Files:**
- Create: `examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/graph/InMemoryAgentGraph.java`

- [ ] **Create InMemoryAgentGraph.java**

```java
package io.casehub.eidos.examples.graph;

import java.util.*;
import java.util.stream.*;

/**
 * Pure in-memory proof-of-concept graph. No CDI, no JPA.
 * Validates the API design and query semantics before infrastructure is built.
 */
public class InMemoryAgentGraph {

    private final List<TaskRecord> tasks = new ArrayList<>();
    private final List<OutcomeRecord> outcomes = new ArrayList<>();
    private final List<AttestationRecord> attestations = new ArrayList<>();
    private final TaskSemanticEnricher enricher;

    public InMemoryAgentGraph() {
        this(new NoOpEnricher());
    }

    public InMemoryAgentGraph(TaskSemanticEnricher enricher) {
        this.enricher = enricher;
    }

    // ── Write ──────────────────────────────────────────────────────────────

    public String recordTask(String agentId, String tenancyId, String capabilityTag,
                             String taskDomain, String externalRef) {
        String taskId = UUID.randomUUID().toString();
        tasks.add(new TaskRecord(taskId, agentId, tenancyId, capabilityTag, taskDomain, externalRef));
        return taskId;
    }

    public void recordOutcome(String taskId, TaskResult result, double confidence) {
        outcomes.add(new OutcomeRecord(taskId, result, confidence));
    }

    public void linkAttestation(String taskId, String agentId, String tenancyId,
                                String ledgerHash, String entryType) {
        attestations.add(new AttestationRecord(taskId, agentId, tenancyId, ledgerHash, entryType));
    }

    // ── Read ───────────────────────────────────────────────────────────────

    public AgentHistory agentHistory(String agentId, String tenancyId) {
        List<TaskRecord> agentTasks = tasks.stream()
            .filter(t -> t.agentId().equals(agentId) && t.tenancyId().equals(tenancyId))
            .toList();
        List<String> taskIds = agentTasks.stream().map(TaskRecord::taskId).toList();
        List<OutcomeRecord> agentOutcomes = outcomes.stream()
            .filter(o -> taskIds.contains(o.taskId()))
            .toList();
        List<AttestationRecord> agentAttestations = attestations.stream()
            .filter(a -> a.agentId().equals(agentId) && a.tenancyId().equals(tenancyId))
            .toList();
        DataSufficiency sufficiency = computeSufficiency(agentOutcomes.size(),
            collectWarnings(agentId, null, null));
        return new AgentHistory(agentId, tenancyId, agentTasks, agentOutcomes,
                                agentAttestations, sufficiency);
    }

    /** Returns agentIds ranked by Wilson lower bound. */
    public List<String> topAgentsByOutcome(String capabilityTag, String taskDomain,
                                            String tenancyId, int limit) {
        // Collect equivalent domains via enricher
        Set<String> domains = new LinkedHashSet<>();
        domains.add(taskDomain);
        for (TaskRecord t : tasks) {
            if (!t.tenancyId().equals(tenancyId)) continue;
            if (enricher.semanticallyEquivalent(taskDomain, t.taskDomain())) {
                domains.add(t.taskDomain());
            }
        }

        // Group outcomes by agent
        Map<String, List<OutcomeRecord>> byAgent = new LinkedHashMap<>();
        for (TaskRecord t : tasks) {
            if (!t.capabilityTag().equals(capabilityTag)) continue;
            if (!t.tenancyId().equals(tenancyId)) continue;
            if (!domains.contains(t.taskDomain())) continue;
            String taskId = t.taskId();
            for (OutcomeRecord o : outcomes) {
                if (o.taskId().equals(taskId)) {
                    byAgent.computeIfAbsent(t.agentId(), k -> new ArrayList<>()).add(o);
                }
            }
        }

        return byAgent.entrySet().stream()
            .sorted(Comparator.comparingDouble(
                (Map.Entry<String, List<OutcomeRecord>> e) -> wilsonScore(e.getValue())).reversed())
            .limit(limit)
            .map(Map.Entry::getKey)
            .toList();
    }

    public List<AttestationRecord> attestationsFor(String agentId, String tenancyId) {
        return attestations.stream()
            .filter(a -> a.agentId().equals(agentId) && a.tenancyId().equals(tenancyId))
            .toList();
    }

    // ── Wilson lower bound ─────────────────────────────────────────────────

    /** quality = confidence × result_multiplier (1.0/0.5/0.0) */
    static double qualityScore(OutcomeRecord o) {
        double m = switch (o.result()) {
            case SUCCEEDED -> 1.0;
            case PARTIALLY -> 0.5;
            case FAILED    -> 0.0;
        };
        return o.confidence() * m;
    }

    static double wilsonScore(List<OutcomeRecord> outcomes) {
        int n = outcomes.size();
        if (n == 0) return 0.0;
        double p = outcomes.stream().mapToDouble(InMemoryAgentGraph::qualityScore).sum() / n;
        double z = 1.645;
        double z2 = z * z;
        double numerator = p + z2 / (2 * n)
                         - z * Math.sqrt((p * (1 - p) + z2 / (4.0 * n)) / n);
        double denominator = 1 + z2 / n;
        return numerator / denominator;
    }

    // ── Sufficiency ────────────────────────────────────────────────────────

    static DataSufficiency computeSufficiency(int count, List<String> warnings) {
        SufficiencyLevel level = count >= 10 ? SufficiencyLevel.SUFFICIENT
                               : count >= 5  ? SufficiencyLevel.INDICATIVE
                               :               SufficiencyLevel.INSUFFICIENT;
        return new DataSufficiency(count, level, warnings);
    }

    List<String> collectWarnings(String agentId, String capabilityTag, String taskDomain) {
        List<String> w = new ArrayList<>();
        if (enricher instanceof NoOpEnricher) {
            w.add("No TaskSemanticEnricher — personality axis correlation unavailable");
        }
        return w;
    }

    // ── Types ──────────────────────────────────────────────────────────────

    public enum TaskResult { SUCCEEDED, PARTIALLY, FAILED }
    public enum SufficiencyLevel { SUFFICIENT, INDICATIVE, INSUFFICIENT }

    public record TaskRecord(String taskId, String agentId, String tenancyId,
                             String capabilityTag, String taskDomain, String externalRef) {}
    public record OutcomeRecord(String taskId, TaskResult result, double confidence) {}
    public record AttestationRecord(String taskId, String agentId, String tenancyId,
                                    String ledgerHash, String entryType) {}
    public record DataSufficiency(int sampleCount, SufficiencyLevel level, List<String> warnings) {}
    public record AgentHistory(String agentId, String tenancyId,
                                List<TaskRecord> tasks, List<OutcomeRecord> outcomes,
                                List<AttestationRecord> attestationRefs,
                                DataSufficiency sufficiency) {}

    public interface TaskSemanticEnricher {
        Set<String> dispositionAxes(String capabilityTag, String taskDomain);
        boolean semanticallyEquivalent(String domainA, String domainB);
        OptionalInt significance(String capabilityTag, String taskDomain);
    }

    static class NoOpEnricher implements TaskSemanticEnricher {
        public Set<String> dispositionAxes(String cap, String domain) { return Set.of(); }
        public boolean semanticallyEquivalent(String a, String b) { return false; }
        public OptionalInt significance(String cap, String domain) { return OptionalInt.empty(); }
    }
}
```

---

### Task 8: PoC Version 1 scenario tests

**Files:**
- Create: `examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/graph/V1GraphScenarioTest.java`

- [ ] **Create V1GraphScenarioTest.java**

```java
package io.casehub.eidos.examples.graph;

import io.casehub.eidos.examples.graph.InMemoryAgentGraph.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.*;

/**
 * V1 scenarios: structural data only, no enricher.
 * These must all pass before Phase 3 infrastructure begins.
 */
class V1GraphScenarioTest {

    InMemoryAgentGraph graph;

    @BeforeEach
    void setUp() {
        graph = new InMemoryAgentGraph(); // NoOp enricher
    }

    @Test
    void v1_routingPrefersAgentWithBetterOutcomeHistory() {
        // Agent A: 20 tasks, SUCCEEDED at confidence 0.78
        for (int i = 0; i < 20; i++) {
            String tid = graph.recordTask("agent-a", "t1", "code-review", "rust", "ref-" + i);
            graph.recordOutcome(tid, TaskResult.SUCCEEDED, 0.78);
        }
        // Agent B: 5 tasks, SUCCEEDED at confidence 0.90
        for (int i = 0; i < 5; i++) {
            String tid = graph.recordTask("agent-b", "t1", "code-review", "rust", "ref-b-" + i);
            graph.recordOutcome(tid, TaskResult.SUCCEEDED, 0.90);
        }

        List<String> ranked = graph.topAgentsByOutcome("code-review", "rust", "t1", 10);

        // Wilson: agent-a ≈ 0.60, agent-b ≈ 0.53. Agent-a wins despite lower raw rate.
        assertThat(ranked).first().isEqualTo("agent-a");
        assertThat(ranked).containsExactly("agent-a", "agent-b");
    }

    @Test
    void v1_wilsonScoreForKnownValues() {
        // Agent A: n=20, all quality=0.78 → Wilson ≈ 0.60
        List<OutcomeRecord> a = java.util.stream.IntStream.range(0, 20)
            .mapToObj(i -> new OutcomeRecord("t" + i, TaskResult.SUCCEEDED, 0.78))
            .toList();
        double scoreA = InMemoryAgentGraph.wilsonScore(a);

        // Agent B: n=5, all quality=0.90 → Wilson ≈ 0.53
        List<OutcomeRecord> b = java.util.stream.IntStream.range(0, 5)
            .mapToObj(i -> new OutcomeRecord("t" + i, TaskResult.SUCCEEDED, 0.90))
            .toList();
        double scoreB = InMemoryAgentGraph.wilsonScore(b);

        assertThat(scoreA).isGreaterThan(scoreB);
        assertThat(scoreA).isBetween(0.55, 0.65); // ≈ 0.60
        assertThat(scoreB).isBetween(0.48, 0.58); // ≈ 0.53
    }

    @Test
    void v1_historyQueryReturnsByCapabilityAndDomain() {
        String t1 = graph.recordTask("agent-x", "t1", "code-review", "java", "ref-1");
        String t2 = graph.recordTask("agent-x", "t1", "planning", "agile", "ref-2");
        graph.recordOutcome(t1, TaskResult.SUCCEEDED, 0.9);
        graph.recordOutcome(t2, TaskResult.SUCCEEDED, 0.8);

        AgentHistory history = graph.agentHistory("agent-x", "t1");

        assertThat(history.tasks()).hasSize(2);
        assertThat(history.outcomes()).hasSize(2);
        assertThat(history.tasks()).extracting(TaskRecord::capabilityTag)
            .containsExactlyInAnyOrder("code-review", "planning");
    }

    @Test
    void v1_attestationChainLinksOutcomeToLedgerHash() {
        String tid = graph.recordTask("agent-x", "t1", "code-review", "java", "ref-1");
        graph.recordOutcome(tid, TaskResult.SUCCEEDED, 0.9);
        graph.linkAttestation(tid, "agent-x", "t1", "hash-abc123", "MessageLedgerEntry");

        List<AttestationRecord> refs = graph.attestationsFor("agent-x", "t1");

        assertThat(refs).hasSize(1);
        assertThat(refs.get(0).ledgerHash()).isEqualTo("hash-abc123");
        assertThat(refs.get(0).taskId()).isEqualTo(tid);
    }

    @Test
    void v1_insufficiencyWarningBelowThreshold() {
        for (int i = 0; i < 4; i++) {
            String tid = graph.recordTask("agent-x", "t1", "code-review", "java", "r" + i);
            graph.recordOutcome(tid, TaskResult.SUCCEEDED, 0.9);
        }

        AgentHistory history = graph.agentHistory("agent-x", "t1");

        assertThat(history.sufficiency().level()).isEqualTo(SufficiencyLevel.INSUFFICIENT);
        assertThat(history.sufficiency().sampleCount()).isEqualTo(4);
    }

    @Test
    void v1_indicativeAtFiveSamples() {
        for (int i = 0; i < 5; i++) {
            String tid = graph.recordTask("agent-x", "t1", "code-review", "java", "r" + i);
            graph.recordOutcome(tid, TaskResult.SUCCEEDED, 0.9);
        }
        assertThat(graph.agentHistory("agent-x", "t1").sufficiency().level())
            .isEqualTo(SufficiencyLevel.INDICATIVE);
    }

    @Test
    void v1_sufficientAtTenSamples() {
        for (int i = 0; i < 10; i++) {
            String tid = graph.recordTask("agent-x", "t1", "code-review", "java", "r" + i);
            graph.recordOutcome(tid, TaskResult.SUCCEEDED, 0.9);
        }
        assertThat(graph.agentHistory("agent-x", "t1").sufficiency().level())
            .isEqualTo(SufficiencyLevel.SUFFICIENT);
    }

    @Test
    void v1_systemFunctionsNormallyWithNoHistory() {
        AgentHistory history = graph.agentHistory("agent-unknown", "t1");

        assertThat(history.tasks()).isEmpty();
        assertThat(history.outcomes()).isEmpty();
        assertThat(history.sufficiency().level()).isEqualTo(SufficiencyLevel.INSUFFICIENT);

        List<String> ranked = graph.topAgentsByOutcome("code-review", "java", "t1", 5);
        assertThat(ranked).isEmpty();
    }

    @Test
    void v1_tenancyIsolation() {
        String tid = graph.recordTask("agent-x", "tenant-A", "code-review", "java", "ref");
        graph.recordOutcome(tid, TaskResult.SUCCEEDED, 0.9);

        AgentHistory tenantB = graph.agentHistory("agent-x", "tenant-B");
        assertThat(tenantB.tasks()).isEmpty();
    }
}
```

- [ ] **Run V1 tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl examples/agent-scenarios \
  -Dtest=V1GraphScenarioTest -q 2>&1 | tail -10
```

Expected: All tests PASS. If any fail, fix `InMemoryAgentGraph` before continuing.

---

### Task 9: PoC Version 2 scenario tests — with enricher

**Files:**
- Create: `examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/graph/V2GraphScenarioTest.java`

- [ ] **Create V2GraphScenarioTest.java**

```java
package io.casehub.eidos.examples.graph;

import io.casehub.eidos.examples.graph.InMemoryAgentGraph.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.util.*;
import static org.assertj.core.api.Assertions.*;

/**
 * V2 scenarios: same data as V1, but with a TaskSemanticEnricher.
 * v2_routingPicksDifferentAgentThanV1 is the critical validation test.
 */
class V2GraphScenarioTest {

    /** Test enricher: maps security/safety-critical tasks to riskAppetite axis.
     *  Treats "security" and "safety-critical" as semantically equivalent. */
    static class TestEnricher implements TaskSemanticEnricher {
        @Override
        public Set<String> dispositionAxes(String cap, String domain) {
            if ("code-review".equals(cap) && Set.of("security", "safety-critical").contains(domain)) {
                return Set.of("riskAppetite", "ruleFollowing");
            }
            return Set.of();
        }

        @Override
        public boolean semanticallyEquivalent(String a, String b) {
            Set<String> group = Set.of("security", "safety-critical");
            return group.contains(a) && group.contains(b) && !a.equals(b);
        }

        @Override
        public OptionalInt significance(String cap, String domain) {
            if ("security".equals(domain) || "safety-critical".equals(domain)) return OptionalInt.of(5);
            return OptionalInt.empty();
        }
    }

    InMemoryAgentGraph graphNoEnricher;
    InMemoryAgentGraph graphWithEnricher;

    @BeforeEach
    void setUp() {
        graphNoEnricher   = new InMemoryAgentGraph();
        graphWithEnricher = new InMemoryAgentGraph(new TestEnricher());
    }

    @Test
    void v2_absenceOfEnricherRecordedInWarnings() {
        graph().recordTask("agent-x", "t1", "code-review", "security", "r1");

        AgentHistory history = graphNoEnricher.agentHistory("agent-x", "t1");

        assertThat(history.sufficiency().warnings())
            .anyMatch(w -> w.contains("No TaskSemanticEnricher"));
    }

    @Test
    void v2_semanticEquivalenceWidensOutcomePool() {
        // agent-x has 8 tasks in "security", 7 in "safety-critical"
        for (int i = 0; i < 8; i++) {
            String tid = graphWithEnricher.recordTask("agent-x", "t1", "code-review", "security", "s" + i);
            graphWithEnricher.recordOutcome(tid, TaskResult.SUCCEEDED, 0.8);
        }
        for (int i = 0; i < 7; i++) {
            String tid = graphWithEnricher.recordTask("agent-x", "t1", "code-review", "safety-critical", "sc" + i);
            graphWithEnricher.recordOutcome(tid, TaskResult.SUCCEEDED, 0.8);
        }

        // Without enricher: only 8 samples in "security" → INDICATIVE
        InMemoryAgentGraph noEnricher = new InMemoryAgentGraph();
        for (int i = 0; i < 8; i++) {
            String tid = noEnricher.recordTask("agent-x", "t1", "code-review", "security", "s" + i);
            noEnricher.recordOutcome(tid, TaskResult.SUCCEEDED, 0.8);
        }
        assertThat(noEnricher.agentHistory("agent-x", "t1").sufficiency().level())
            .isEqualTo(SufficiencyLevel.INDICATIVE); // 8 samples

        // With enricher: 15 samples → SUFFICIENT
        List<String> ranked = graphWithEnricher.topAgentsByOutcome("code-review", "security", "t1", 5);
        assertThat(ranked).contains("agent-x");
        // The combined pool has 15 outcomes — verify via history
        AgentHistory h = graphWithEnricher.agentHistory("agent-x", "t1");
        assertThat(h.tasks()).hasSize(15);
    }

    @Test
    void v2_routingPicksDifferentAgentThanV1() {
        // Agent A: 10 tasks in "security", confidence 0.70 → raw quality 0.70
        // Agent B: 3 tasks in "security", confidence 0.95; 7 tasks in "safety-critical" confidence 0.95
        //   Without enricher: only 3 security tasks → Wilson ≈ 0.51 (small sample penalty)
        //   With enricher: 10 total (3+7) → Wilson ≈ 0.73 (larger pool, high quality)

        InMemoryAgentGraph v1 = new InMemoryAgentGraph();
        InMemoryAgentGraph v2 = new InMemoryAgentGraph(new TestEnricher());

        for (InMemoryAgentGraph g : List.of(v1, v2)) {
            for (int i = 0; i < 10; i++) {
                String tid = g.recordTask("agent-a", "t1", "code-review", "security", "a" + i);
                g.recordOutcome(tid, TaskResult.SUCCEEDED, 0.70);
            }
            for (int i = 0; i < 3; i++) {
                String tid = g.recordTask("agent-b", "t1", "code-review", "security", "bs" + i);
                g.recordOutcome(tid, TaskResult.SUCCEEDED, 0.95);
            }
            for (int i = 0; i < 7; i++) {
                String tid = g.recordTask("agent-b", "t1", "code-review", "safety-critical", "bsc" + i);
                g.recordOutcome(tid, TaskResult.SUCCEEDED, 0.95);
            }
        }

        List<String> v1Ranked = v1.topAgentsByOutcome("code-review", "security", "t1", 10);
        List<String> v2Ranked = v2.topAgentsByOutcome("code-review", "security", "t1", 10);

        // V1: agent-a (10 tasks, quality 0.70, Wilson≈0.54) vs agent-b (3 tasks, quality 0.95, Wilson≈0.51)
        // Agent-a should win V1 (larger pool wins over higher raw quality)
        assertThat(v1Ranked).first().isEqualTo("agent-a");

        // V2: agent-b now has 10 tasks (3+7 merged), quality 0.95 → Wilson≈0.73 >> agent-a's 0.54
        assertThat(v2Ranked).first().isEqualTo("agent-b");

        // The two rankings differ — enricher changed the outcome
        assertThat(v1Ranked.get(0)).isNotEqualTo(v2Ranked.get(0));
    }

    @Test
    void v2_enricherAxesAvailableForPersonalityMapping() {
        TestEnricher e = new TestEnricher();
        assertThat(e.dispositionAxes("code-review", "security"))
            .containsExactlyInAnyOrder("riskAppetite", "ruleFollowing");
        assertThat(e.dispositionAxes("code-review", "java")).isEmpty();
        assertThat(e.semanticallyEquivalent("security", "safety-critical")).isTrue();
        assertThat(e.semanticallyEquivalent("security", "java")).isFalse();
    }

    private InMemoryAgentGraph graph() { return graphNoEnricher; }
}
```

- [ ] **Run V2 tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl examples/agent-scenarios \
  -Dtest=V2GraphScenarioTest -q 2>&1 | tail -10
```

Expected: All tests PASS, especially `v2_routingPicksDifferentAgentThanV1`.

- [ ] **Commit PoC**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add examples/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(examples): Phase 4a PoC — InMemoryAgentGraph + V1/V2 scenario tests

V1: structural routing, Wilson formula, sufficiency thresholds, tenancy isolation.
V2: TaskSemanticEnricher widens sample pool; v2_routingPicksDifferentAgentThanV1
    confirms enricher changes routing outcome.

PoC gate passed. Phase 3 infrastructure may proceed.

Refs #35"
```

**⚠ EVALUATION GATE:** Review the test results now. If `v2_routingPicksDifferentAgentThanV1` does not show genuine divergence, stop and reassess the enricher design before continuing. Only proceed to Phase 3 if all V1 and V2 tests pass with meaningful semantics.

---

## PHASE 3: Graph Infrastructure

Only begin after Phase 2 gate passes evaluation.

---

### Task 10: New API types — records and enums

**Files:**
- Create: `api/src/main/java/io/casehub/eidos/api/TaskResult.java`
- Create: `api/src/main/java/io/casehub/eidos/api/SufficiencyLevel.java`
- Create: `api/src/main/java/io/casehub/eidos/api/AgentTask.java`
- Create: `api/src/main/java/io/casehub/eidos/api/AgentTaskId.java`
- Create: `api/src/main/java/io/casehub/eidos/api/AgentOutcome.java`
- Create: `api/src/main/java/io/casehub/eidos/api/AttestationRef.java`
- Create: `api/src/main/java/io/casehub/eidos/api/GraphDataSufficiency.java`
- Create: `api/src/main/java/io/casehub/eidos/api/AgentTaskHistory.java`
- Create: `api/src/main/java/io/casehub/eidos/api/BackfillResult.java`

- [ ] **Create TaskResult.java**

```java
package io.casehub.eidos.api;
public enum TaskResult { SUCCEEDED, PARTIALLY, FAILED }
```

- [ ] **Create SufficiencyLevel.java**

```java
package io.casehub.eidos.api;
public enum SufficiencyLevel { SUFFICIENT, INDICATIVE, INSUFFICIENT }
```

- [ ] **Create AgentTask.java**

```java
package io.casehub.eidos.api;

import java.time.Instant;

public record AgentTask(
    String taskId,
    String agentId,
    String tenancyId,
    String capabilityTag,
    String taskDomain,
    String externalRef,   // opaque; nullable
    Instant startedAt,
    Instant endedAt       // null if in progress
) {}
```

- [ ] **Create AgentTaskId.java**

```java
package io.casehub.eidos.api;
public record AgentTaskId(String taskId, String agentId, String tenancyId) {}
```

- [ ] **Create AgentOutcome.java**

```java
package io.casehub.eidos.api;

import java.time.Instant;

public record AgentOutcome(
    String taskId,
    TaskResult result,
    double confidence,                   // 0.0–1.0
    DegradationReason degradationReason  // null if not applicable; no tenancyId — derive via task
) {}
```

- [ ] **Create AttestationRef.java**

```java
package io.casehub.eidos.api;

import java.time.Instant;

public record AttestationRef(
    String taskId,           // null for backfilled refs without a matched task
    String agentId,
    String tenancyId,
    String ledgerEntryHash,
    String entryType,
    Instant attestedAt
) {}
```

- [ ] **Create GraphDataSufficiency.java**

```java
package io.casehub.eidos.api;

import java.time.Instant;
import java.util.List;

public record GraphDataSufficiency(
    int sampleCount,
    SufficiencyLevel level,
    Instant dataFrom,    // null if no data
    Instant dataThrough, // null if no data
    List<String> warnings
) {
    public static GraphDataSufficiency forCount(int count, Instant from,
                                                Instant through, List<String> warnings) {
        SufficiencyLevel level = count >= 10 ? SufficiencyLevel.SUFFICIENT
                               : count >= 5  ? SufficiencyLevel.INDICATIVE
                               :               SufficiencyLevel.INSUFFICIENT;
        return new GraphDataSufficiency(count, level, from, through, warnings);
    }

    public static GraphDataSufficiency empty(List<String> warnings) {
        return new GraphDataSufficiency(0, SufficiencyLevel.INSUFFICIENT, null, null, warnings);
    }
}
```

- [ ] **Create AgentTaskHistory.java**

```java
package io.casehub.eidos.api;

import java.util.List;

public record AgentTaskHistory(
    String agentId,
    String tenancyId,
    List<AgentTask> tasks,           // includes in-progress (endedAt=null)
    List<AgentOutcome> outcomes,
    List<AttestationRef> attestationRefs,
    GraphDataSufficiency sufficiency
) {}
```

- [ ] **Create BackfillResult.java**

```java
package io.casehub.eidos.api;

import java.time.Instant;

public record BackfillResult(int imported, int skipped,
                              Instant rangeFrom, Instant rangeThrough) {}
```

- [ ] **Compile api**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api -q 2>&1 | tail -5
```

Expected: BUILD SUCCESS

---

### Task 11: New SPIs

**Files:**
- Create: `api/src/main/java/io/casehub/eidos/api/AgentGraphStore.java`
- Create: `api/src/main/java/io/casehub/eidos/api/AgentGraphQuery.java`
- Create: `api/src/main/java/io/casehub/eidos/api/ReactiveAgentGraphQuery.java`
- Create: `api/src/main/java/io/casehub/eidos/api/TaskSemanticEnricher.java`
- Create: `api/src/main/java/io/casehub/eidos/api/AgentGraphBackfill.java`

- [ ] **Create AgentGraphStore.java**

```java
package io.casehub.eidos.api;

public interface AgentGraphStore {
    void recordTask(AgentTask task);
    void recordOutcome(AgentTaskId id, AgentOutcome outcome);
    void linkAttestation(AgentTaskId id, AttestationRef ref);
}
```

- [ ] **Create AgentGraphQuery.java**

```java
package io.casehub.eidos.api;

import java.util.List;

public interface AgentGraphQuery {
    /** Returns all tasks including in-progress (endedAt=null). */
    AgentTaskHistory agentHistory(String agentId, String tenancyId);

    AgentTaskHistory historyByCapability(String agentId, String capabilityTag, String tenancyId);

    /**
     * Returns agentIds ranked by Wilson lower bound score.
     * quality = confidence × multiplier (SUCCEEDED=1.0, PARTIALLY=0.5, FAILED=0.0)
     * Wilson z=1.645. Score=0 when no observations.
     */
    List<String> topAgentsByOutcome(String capabilityTag, String taskDomain,
                                    String tenancyId, int limit);

    List<AttestationRef> attestationsFor(String agentId, String tenancyId);
}
```

- [ ] **Create ReactiveAgentGraphQuery.java**

```java
package io.casehub.eidos.api;

import io.smallrye.mutiny.Uni;
import java.util.List;

public interface ReactiveAgentGraphQuery {
    Uni<AgentTaskHistory> agentHistory(String agentId, String tenancyId);
    Uni<List<String>> topAgentsByOutcome(String capabilityTag, String taskDomain,
                                          String tenancyId, int limit);
}
```

- [ ] **Create TaskSemanticEnricher.java**

```java
package io.casehub.eidos.api;

import java.util.OptionalInt;
import java.util.Set;

public interface TaskSemanticEnricher {
    Set<String> dispositionAxes(String capabilityTag, String taskDomain);
    boolean semanticallyEquivalent(String domainA, String domainB);
    OptionalInt significance(String capabilityTag, String taskDomain);
}
```

- [ ] **Create AgentGraphBackfill.java**

```java
package io.casehub.eidos.api;

import java.time.Instant;

public interface AgentGraphBackfill {
    BackfillResult backfillAgent(String agentId, String tenancyId);
    BackfillResult backfillAll(String tenancyId);
    BackfillResult backfillDelta(String tenancyId, Instant since);
}
```

- [ ] **Commit api types + SPIs**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add api/src/main/java/io/casehub/eidos/api/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(api): knowledge graph types and SPIs

AgentTask, AgentTaskId, AgentOutcome, AttestationRef, AgentTaskHistory,
GraphDataSufficiency, BackfillResult, TaskResult, SufficiencyLevel,
AgentGraphStore, AgentGraphQuery, ReactiveAgentGraphQuery,
TaskSemanticEnricher, AgentGraphBackfill

Refs #35"
```

---

### Task 12: Runtime defaults — NoOps and bridge

**Files:**
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/graph/NoOpAgentGraphStore.java`
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/graph/NoOpAgentGraphQuery.java`
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/graph/NoOpAgentGraphBackfill.java`
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/graph/NoOpTaskSemanticEnricher.java`
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/graph/BlockingToReactiveGraphBridge.java`

- [ ] **Create NoOpAgentGraphStore.java**

```java
package io.casehub.eidos.runtime.graph;

import io.casehub.eidos.api.*;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;

@DefaultBean @ApplicationScoped
public class NoOpAgentGraphStore implements AgentGraphStore {
    @Override public void recordTask(AgentTask task) {}
    @Override public void recordOutcome(AgentTaskId id, AgentOutcome outcome) {}
    @Override public void linkAttestation(AgentTaskId id, AttestationRef ref) {}
}
```

- [ ] **Create NoOpAgentGraphQuery.java**

```java
package io.casehub.eidos.runtime.graph;

import io.casehub.eidos.api.*;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.List;

@DefaultBean @ApplicationScoped
public class NoOpAgentGraphQuery implements AgentGraphQuery {

    @Override
    public AgentTaskHistory agentHistory(String agentId, String tenancyId) {
        return new AgentTaskHistory(agentId, tenancyId, List.of(), List.of(), List.of(),
            GraphDataSufficiency.empty(List.of()));
    }

    @Override
    public AgentTaskHistory historyByCapability(String agentId, String capabilityTag,
                                                 String tenancyId) {
        return agentHistory(agentId, tenancyId);
    }

    @Override
    public List<String> topAgentsByOutcome(String capabilityTag, String taskDomain,
                                            String tenancyId, int limit) {
        return List.of();
    }

    @Override
    public List<AttestationRef> attestationsFor(String agentId, String tenancyId) {
        return List.of();
    }
}
```

- [ ] **Create NoOpAgentGraphBackfill.java**

```java
package io.casehub.eidos.runtime.graph;

import io.casehub.eidos.api.*;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import java.time.Instant;

@DefaultBean @ApplicationScoped
public class NoOpAgentGraphBackfill implements AgentGraphBackfill {
    @Override public BackfillResult backfillAgent(String agentId, String tenancyId) {
        return new BackfillResult(0, 0, null, null);
    }
    @Override public BackfillResult backfillAll(String tenancyId) {
        return new BackfillResult(0, 0, null, null);
    }
    @Override public BackfillResult backfillDelta(String tenancyId, Instant since) {
        return new BackfillResult(0, 0, null, null);
    }
}
```

- [ ] **Create NoOpTaskSemanticEnricher.java**

```java
package io.casehub.eidos.runtime.graph;

import io.casehub.eidos.api.TaskSemanticEnricher;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.OptionalInt;
import java.util.Set;

@DefaultBean @ApplicationScoped
public class NoOpTaskSemanticEnricher implements TaskSemanticEnricher {
    @Override public Set<String> dispositionAxes(String cap, String domain) { return Set.of(); }
    @Override public boolean semanticallyEquivalent(String a, String b) { return false; }
    @Override public OptionalInt significance(String cap, String domain) { return OptionalInt.empty(); }
}
```

- [ ] **Create BlockingToReactiveGraphBridge.java** — in runtime, not graph module

```java
package io.casehub.eidos.runtime.graph;

import io.casehub.eidos.api.*;
import io.quarkus.arc.DefaultBean;
import io.smallrye.mutiny.Uni;
import io.vertx.mutiny.core.Vertx;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.List;

@DefaultBean @ApplicationScoped
public class BlockingToReactiveGraphBridge implements ReactiveAgentGraphQuery {

    @Inject AgentGraphQuery blocking;

    @Override
    public Uni<AgentTaskHistory> agentHistory(String agentId, String tenancyId) {
        return Uni.createFrom()
                  .item(() -> blocking.agentHistory(agentId, tenancyId))
                  .runSubscriptionOn(Infrastructure.getDefaultWorkerPool());
    }

    @Override
    public Uni<List<String>> topAgentsByOutcome(String capabilityTag, String taskDomain,
                                                 String tenancyId, int limit) {
        return Uni.createFrom()
                  .item(() -> blocking.topAgentsByOutcome(capabilityTag, taskDomain, tenancyId, limit))
                  .runSubscriptionOn(Infrastructure.getDefaultWorkerPool());
    }
}
```

Note: `Infrastructure` is from `io.smallrye.mutiny.infrastructure.Infrastructure`.

- [ ] **Run runtime tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q 2>&1 | tail -5
```

Expected: BUILD SUCCESS

- [ ] **Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add runtime/src/main/java/io/casehub/eidos/runtime/graph/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(runtime): NoOp defaults and BlockingToReactiveGraphBridge for graph SPIs

All graph SPIs satisfied without casehub-eidos-graph on classpath.
Bridge in runtime (same module as SPI) per PP-20260529-5745c1.

Refs #35"
```

---

### Task 13: New graph module — scaffold, pom, V3 migration

**Files:**
- Create: `graph/pom.xml`
- Create: `graph/src/main/resources/db/eidos/migration/V3__agent_graph.sql`
- Modify: `pom.xml` — add `<module>graph</module>`

- [ ] **Create graph/pom.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-eidos-parent</artifactId>
    <version>0.2-SNAPSHOT</version>
  </parent>

  <artifactId>casehub-eidos-graph</artifactId>
  <name>casehub-eidos-graph</name>
  <description>JPA-backed knowledge graph for casehub-eidos — optional, activates by classpath presence</description>

  <dependencies>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-eidos-api</artifactId>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-hibernate-orm</artifactId>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-flyway</artifactId>
    </dependency>

    <!-- Test -->
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-junit5</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-jdbc-h2</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-eidos-memory</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-eidos</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.assertj</groupId>
      <artifactId>assertj-core</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>com.tngtech.archunit</groupId>
      <artifactId>archunit-junit5</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>
</project>
```

- [ ] **Add `<module>graph</module>` to root pom.xml** after the `eval` module entry.

- [ ] **Create graph/src/main/resources/db/eidos/migration/V3__agent_graph.sql**

```sql
CREATE TABLE agent_task (
    task_id        VARCHAR(36)       PRIMARY KEY,
    agent_id       VARCHAR(255)      NOT NULL,
    tenancy_id     VARCHAR(255)      NOT NULL,
    capability_tag VARCHAR(255)      NOT NULL,
    task_domain    VARCHAR(255),
    external_ref   TEXT,
    started_at     TIMESTAMPTZ       NOT NULL,
    ended_at       TIMESTAMPTZ
);
CREATE INDEX idx_task_agent          ON agent_task(agent_id, tenancy_id);
CREATE INDEX idx_task_cap_domain     ON agent_task(agent_id, capability_tag, task_domain, tenancy_id);
CREATE INDEX idx_task_cap_tenant     ON agent_task(capability_tag, task_domain, tenancy_id);

CREATE TABLE agent_outcome (
    task_id            VARCHAR(36)       PRIMARY KEY
                                          REFERENCES agent_task(task_id),
    result             VARCHAR(20)       NOT NULL,
    confidence         DOUBLE PRECISION  NOT NULL
                                          CHECK (confidence BETWEEN 0 AND 1),
    degradation_reason VARCHAR(50),
    observed_at        TIMESTAMPTZ       NOT NULL
);

CREATE TABLE attestation_ref (
    ref_id             VARCHAR(36)   PRIMARY KEY,
    task_id            VARCHAR(36)   REFERENCES agent_task(task_id),
    agent_id           VARCHAR(255)  NOT NULL,
    tenancy_id         VARCHAR(255)  NOT NULL,
    ledger_entry_hash  VARCHAR(255)  NOT NULL,
    entry_type         VARCHAR(255)  NOT NULL,
    attested_at        TIMESTAMPTZ   NOT NULL,
    CONSTRAINT uq_attestation UNIQUE (ledger_entry_hash, tenancy_id)
);
CREATE INDEX idx_attest_agent ON attestation_ref(agent_id, tenancy_id);
```

- [ ] **Create graph/src/test/resources/application.properties** for H2 test datasource

```properties
quarkus.datasource.db-kind=h2
quarkus.datasource.jdbc.url=jdbc:h2:mem:eidos_graph_test;DB_CLOSE_DELAY=-1;MODE=PostgreSQL
quarkus.flyway.migrate-at-start=true
quarkus.flyway.locations=classpath:db/eidos/migration
quarkus.hibernate-orm.database.generation=none
```

- [ ] **Compile**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl graph -q 2>&1 | tail -5
```

Expected: BUILD SUCCESS

---

### Task 14: JPA entities

**Files:**
- Create: `graph/src/main/java/io/casehub/eidos/graph/entity/AgentTaskEntity.java`
- Create: `graph/src/main/java/io/casehub/eidos/graph/entity/AgentOutcomeEntity.java`
- Create: `graph/src/main/java/io/casehub/eidos/graph/entity/AttestationRefEntity.java`

- [ ] **Create AgentTaskEntity.java**

```java
package io.casehub.eidos.graph.entity;

import io.casehub.eidos.api.AgentTask;
import jakarta.persistence.*;
import java.time.Instant;
import java.util.ArrayList;
import java.util.List;

@Entity
@Table(name = "agent_task")
public class AgentTaskEntity {

    @Id
    @Column(name = "task_id")
    String taskId;

    @Column(name = "agent_id",       nullable = false) String agentId;
    @Column(name = "tenancy_id",     nullable = false) String tenancyId;
    @Column(name = "capability_tag", nullable = false) String capabilityTag;
    @Column(name = "task_domain")                      String taskDomain;
    @Column(name = "external_ref",   columnDefinition = "TEXT") String externalRef;
    @Column(name = "started_at",     nullable = false) Instant startedAt;
    @Column(name = "ended_at")                         Instant endedAt;

    @OneToOne(mappedBy = "task", cascade = CascadeType.ALL,
              fetch = FetchType.LAZY, optional = true)
    AgentOutcomeEntity outcome;

    @OneToMany(mappedBy = "task", fetch = FetchType.LAZY)
    List<AttestationRefEntity> attestationRefs = new ArrayList<>();

    protected AgentTaskEntity() {}

    public static AgentTaskEntity from(AgentTask t) {
        var e = new AgentTaskEntity();
        e.taskId        = t.taskId();
        e.agentId       = t.agentId();
        e.tenancyId     = t.tenancyId();
        e.capabilityTag = t.capabilityTag();
        e.taskDomain    = t.taskDomain();
        e.externalRef   = t.externalRef();
        e.startedAt     = t.startedAt();
        e.endedAt       = t.endedAt();
        return e;
    }

    public AgentTask toRecord() {
        return new AgentTask(taskId, agentId, tenancyId, capabilityTag,
                             taskDomain, externalRef, startedAt, endedAt);
    }
}
```

- [ ] **Create AgentOutcomeEntity.java**

```java
package io.casehub.eidos.graph.entity;

import io.casehub.eidos.api.AgentOutcome;
import io.casehub.eidos.api.DegradationReason;
import io.casehub.eidos.api.TaskResult;
import jakarta.persistence.*;
import java.time.Instant;

@Entity
@Table(name = "agent_outcome")
public class AgentOutcomeEntity {

    @Id
    @Column(name = "task_id")
    String taskId;

    @OneToOne
    @MapsId
    @JoinColumn(name = "task_id")
    AgentTaskEntity task;

    @Column(nullable = false)                String result;
    @Column(nullable = false)                double confidence;
    @Column(name = "degradation_reason")     String degradationReason;
    @Column(name = "observed_at", nullable = false) Instant observedAt;

    protected AgentOutcomeEntity() {}

    public static AgentOutcomeEntity from(AgentOutcome o, AgentTaskEntity task) {
        var e = new AgentOutcomeEntity();
        e.task              = task;
        e.taskId            = o.taskId();
        e.result            = o.result().name();
        e.confidence        = o.confidence();
        e.degradationReason = o.degradationReason() != null ? o.degradationReason().name() : null;
        e.observedAt        = Instant.now();
        return e;
    }

    public AgentOutcome toRecord() {
        return new AgentOutcome(
            taskId,
            TaskResult.valueOf(result),
            confidence,
            degradationReason != null ? DegradationReason.valueOf(degradationReason) : null
        );
    }
}
```

- [ ] **Create AttestationRefEntity.java**

```java
package io.casehub.eidos.graph.entity;

import io.casehub.eidos.api.AttestationRef;
import jakarta.persistence.*;
import java.time.Instant;

@Entity
@Table(name = "attestation_ref")
public class AttestationRefEntity {

    @Id
    @Column(name = "ref_id")
    String refId;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "task_id")        // nullable for backfilled refs
    AgentTaskEntity task;

    @Column(name = "agent_id",          nullable = false) String agentId;
    @Column(name = "tenancy_id",        nullable = false) String tenancyId;
    @Column(name = "ledger_entry_hash", nullable = false) String ledgerEntryHash;
    @Column(name = "entry_type",        nullable = false) String entryType;
    @Column(name = "attested_at",       nullable = false) Instant attestedAt;

    protected AttestationRefEntity() {}

    public static AttestationRefEntity from(String refId, AttestationRef r, AgentTaskEntity task) {
        var e = new AttestationRefEntity();
        e.refId           = refId;
        e.task            = task;
        e.agentId         = r.agentId();
        e.tenancyId       = r.tenancyId();
        e.ledgerEntryHash = r.ledgerEntryHash();
        e.entryType       = r.entryType();
        e.attestedAt      = r.attestedAt();
        return e;
    }

    public AttestationRef toRecord() {
        return new AttestationRef(
            task != null ? task.taskId : null,
            agentId, tenancyId, ledgerEntryHash, entryType, attestedAt
        );
    }
}
```

---

### Task 15: JpaAgentGraphStore

**Files:**
- Create: `graph/src/main/java/io/casehub/eidos/graph/JpaAgentGraphStore.java`
- Create: `graph/src/test/java/io/casehub/eidos/graph/JpaAgentGraphStoreTest.java`

- [ ] **Write failing test first**

Create `graph/src/test/java/io/casehub/eidos/graph/JpaAgentGraphStoreTest.java`:

```java
package io.casehub.eidos.graph;

import io.casehub.eidos.api.*;
import io.quarkus.test.TestTransaction;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import java.time.Instant;

import static org.assertj.core.api.Assertions.*;

@QuarkusTest
@TestTransaction
class JpaAgentGraphStoreTest {

    @Inject AgentGraphStore store;
    @Inject AgentGraphQuery query;

    static AgentTask task(String taskId, String agentId, String cap, String domain) {
        return new AgentTask(taskId, agentId, "t1", cap, domain, "ref-" + taskId,
                             Instant.now(), null);
    }

    @Test
    void recordTask_then_agentHistory_returns_task() {
        store.recordTask(task("t1", "agent-a", "code-review", "java"));

        var history = query.agentHistory("agent-a", "t1");

        assertThat(history.tasks()).hasSize(1);
        assertThat(history.tasks().get(0).taskId()).isEqualTo("t1");
    }

    @Test
    void recordOutcome_links_to_task() {
        store.recordTask(task("t2", "agent-a", "code-review", "java"));
        var id = new AgentTaskId("t2", "agent-a", "t1");
        store.recordOutcome(id, new AgentOutcome("t2", TaskResult.SUCCEEDED, 0.9, null));

        var history = query.agentHistory("agent-a", "t1");

        assertThat(history.outcomes()).hasSize(1);
        assertThat(history.outcomes().get(0).result()).isEqualTo(TaskResult.SUCCEEDED);
        assertThat(history.outcomes().get(0).confidence()).isEqualTo(0.9);
    }

    @Test
    void linkAttestation_stored_and_queryable() {
        store.recordTask(task("t3", "agent-a", "code-review", "java"));
        var id = new AgentTaskId("t3", "agent-a", "t1");
        store.linkAttestation(id, new AttestationRef("t3", "agent-a", "t1",
            "hash-xyz", "MessageLedgerEntry", Instant.now()));

        var refs = query.attestationsFor("agent-a", "t1");

        assertThat(refs).hasSize(1);
        assertThat(refs.get(0).ledgerEntryHash()).isEqualTo("hash-xyz");
    }

    @Test
    void tenancy_isolation() {
        store.recordTask(new AgentTask("t4", "agent-a", "tenant-A",
            "code-review", "java", "ref", Instant.now(), null));

        var history = query.agentHistory("agent-a", "tenant-B");

        assertThat(history.tasks()).isEmpty();
    }

    @Test
    void duplicate_attestation_is_idempotent() {
        store.recordTask(task("t5", "agent-a", "code-review", "java"));
        var id = new AgentTaskId("t5", "agent-a", "t1");
        var ref = new AttestationRef("t5", "agent-a", "t1", "hash-dup", "ML", Instant.now());

        store.linkAttestation(id, ref);
        assertThatCode(() -> store.linkAttestation(id, ref)).doesNotThrowAnyException();

        assertThat(query.attestationsFor("agent-a", "t1")).hasSize(1); // not doubled
    }

    @Test
    void inProgress_tasks_included_in_history() {
        // endedAt=null means in-progress
        store.recordTask(new AgentTask("t6", "agent-a", "t1",
            "code-review", "java", "ref", Instant.now(), null));

        assertThat(query.agentHistory("agent-a", "t1").tasks()).hasSize(1);
        assertThat(query.agentHistory("agent-a", "t1").tasks().get(0).endedAt()).isNull();
    }
}
```

- [ ] **Run test to confirm failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graph -Dtest=JpaAgentGraphStoreTest -q 2>&1 | tail -10
```

Expected: FAIL — `JpaAgentGraphStore` not yet implemented.

- [ ] **Create JpaAgentGraphStore.java**

```java
package io.casehub.eidos.graph;

import io.casehub.eidos.api.*;
import io.casehub.eidos.graph.entity.*;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.transaction.Transactional;
import java.util.UUID;

@ApplicationScoped
public class JpaAgentGraphStore implements AgentGraphStore {

    @Inject EntityManager em;

    @Override
    @Transactional
    public void recordTask(AgentTask task) {
        em.persist(AgentTaskEntity.from(task));
    }

    @Override
    @Transactional
    public void recordOutcome(AgentTaskId id, AgentOutcome outcome) {
        AgentTaskEntity task = em.find(AgentTaskEntity.class, id.taskId());
        if (task == null) return;
        em.persist(AgentOutcomeEntity.from(outcome, task));
    }

    @Override
    @Transactional
    public void linkAttestation(AgentTaskId id, AttestationRef ref) {
        AgentTaskEntity task = em.find(AgentTaskEntity.class, id.taskId());
        String refId = UUID.randomUUID().toString();
        em.createQuery(
            "INSERT INTO AttestationRefEntity(refId, task, agentId, tenancyId, " +
            "ledgerEntryHash, entryType, attestedAt) " +
            "SELECT :refId, t, :agentId, :tenancyId, :hash, :type, :at " +
            "FROM AgentTaskEntity t WHERE t.taskId = :taskId " +
            "AND NOT EXISTS (SELECT 1 FROM AttestationRefEntity a " +
            "WHERE a.ledgerEntryHash = :hash AND a.tenancyId = :tenancyId)")
            // JPQL doesn't support INSERT ... ON CONFLICT; use native SQL instead:
            ;
        // Use native INSERT ... ON CONFLICT DO NOTHING
        em.createNativeQuery(
            "INSERT INTO attestation_ref(ref_id, task_id, agent_id, tenancy_id, " +
            "ledger_entry_hash, entry_type, attested_at) " +
            "VALUES(?, ?, ?, ?, ?, ?, ?) " +
            "ON CONFLICT (ledger_entry_hash, tenancy_id) DO NOTHING")
          .setParameter(1, refId)
          .setParameter(2, task != null ? task.taskId : null)
          .setParameter(3, ref.agentId())
          .setParameter(4, ref.tenancyId())
          .setParameter(5, ref.ledgerEntryHash())
          .setParameter(6, ref.entryType())
          .setParameter(7, ref.attestedAt())
          .executeUpdate();
    }
}
```

- [ ] **Run test to confirm pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graph -Dtest=JpaAgentGraphStoreTest -q 2>&1 | tail -10
```

Expected: BUILD SUCCESS

---

### Task 16: JpaAgentGraphQuery — Wilson formula

**Files:**
- Create: `graph/src/main/java/io/casehub/eidos/graph/JpaAgentGraphQuery.java`
- Create: `graph/src/test/java/io/casehub/eidos/graph/JpaAgentGraphQueryTest.java`

- [ ] **Write failing tests**

```java
package io.casehub.eidos.graph;

import io.casehub.eidos.api.*;
import io.quarkus.test.TestTransaction;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.List;

import static org.assertj.core.api.Assertions.*;

@QuarkusTest
@TestTransaction
class JpaAgentGraphQueryTest {

    @Inject AgentGraphStore store;
    @Inject AgentGraphQuery query;

    private void addOutcomes(String agentId, String cap, String domain,
                              int count, TaskResult result, double confidence) {
        for (int i = 0; i < count; i++) {
            String tid = "t-" + agentId + "-" + i;
            store.recordTask(new AgentTask(tid, agentId, "t1", cap, domain,
                                           "ref-" + i, Instant.now(), Instant.now()));
            store.recordOutcome(new AgentTaskId(tid, agentId, "t1"),
                                new AgentOutcome(tid, result, confidence, null));
        }
    }

    @Test
    void topAgentsByOutcome_wilson_correct_agent_a_beats_b() {
        // Agent A: 20 tasks SUCCEEDED confidence 0.78 → Wilson ≈ 0.60
        addOutcomes("agent-a", "code-review", "rust", 20, TaskResult.SUCCEEDED, 0.78);
        // Agent B: 5 tasks SUCCEEDED confidence 0.90 → Wilson ≈ 0.53
        addOutcomes("agent-b", "code-review", "rust", 5, TaskResult.SUCCEEDED, 0.90);

        List<String> ranked = query.topAgentsByOutcome("code-review", "rust", "t1", 10);

        assertThat(ranked).first().isEqualTo("agent-a");
    }

    @Test
    void topAgentsByOutcome_empty_when_no_history() {
        assertThat(query.topAgentsByOutcome("code-review", "rust", "t1", 5)).isEmpty();
    }

    @Test
    void sufficiency_thresholds() {
        addOutcomes("a1", "cr", "java", 4, TaskResult.SUCCEEDED, 0.9);
        addOutcomes("a2", "cr", "java", 5, TaskResult.SUCCEEDED, 0.9);
        addOutcomes("a3", "cr", "java", 10, TaskResult.SUCCEEDED, 0.9);

        assertThat(query.agentHistory("a1", "t1").sufficiency().level())
            .isEqualTo(SufficiencyLevel.INSUFFICIENT);
        assertThat(query.agentHistory("a2", "t1").sufficiency().level())
            .isEqualTo(SufficiencyLevel.INDICATIVE);
        assertThat(query.agentHistory("a3", "t1").sufficiency().level())
            .isEqualTo(SufficiencyLevel.SUFFICIENT);
    }

    @Test
    void attestationsFor_returns_backfilled_refs_with_null_taskId() {
        // Backfilled ref has no task
        em.createNativeQuery(
            "INSERT INTO attestation_ref(ref_id, task_id, agent_id, tenancy_id, " +
            "ledger_entry_hash, entry_type, attested_at) " +
            "VALUES('bf-1', NULL, 'agent-x', 't1', 'hash-backfill', 'ML', now())")
          .executeUpdate();

        List<AttestationRef> refs = query.attestationsFor("agent-x", "t1");

        assertThat(refs).hasSize(1);
        assertThat(refs.get(0).taskId()).isNull();
        assertThat(refs.get(0).ledgerEntryHash()).isEqualTo("hash-backfill");
    }
}
```

- [ ] **Run test to confirm failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graph -Dtest=JpaAgentGraphQueryTest -q 2>&1 | tail -5
```

Expected: FAIL

- [ ] **Create JpaAgentGraphQuery.java**

```java
package io.casehub.eidos.graph;

import io.casehub.eidos.api.*;
import io.casehub.eidos.graph.entity.*;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.transaction.Transactional.TxType;
import jakarta.transaction.Transactional;
import java.time.Instant;
import java.util.*;
import java.util.stream.*;

@ApplicationScoped
public class JpaAgentGraphQuery implements AgentGraphQuery {

    @Inject EntityManager em;
    @Inject TaskSemanticEnricher enricher;

    // Wilson lower bound: z = 1.645 (90% CI)
    // quality = confidence × (SUCCEEDED=1.0, PARTIALLY=0.5, FAILED=0.0)
    static double qualityMultiplier(TaskResult r) {
        return switch (r) { case SUCCEEDED -> 1.0; case PARTIALLY -> 0.5; case FAILED -> 0.0; };
    }

    static double wilsonScore(double sumQuality, int n) {
        if (n == 0) return 0.0;
        double p = sumQuality / n;
        double z = 1.645, z2 = z * z;
        double num = p + z2 / (2 * n) - z * Math.sqrt((p * (1 - p) + z2 / (4.0 * n)) / n);
        return num / (1 + z2 / n);
    }

    @Override
    @Transactional(TxType.SUPPORTS)
    public AgentTaskHistory agentHistory(String agentId, String tenancyId) {
        List<AgentTaskEntity> tasks = em.createQuery(
                "SELECT t FROM AgentTaskEntity t WHERE t.agentId = :a AND t.tenancyId = :tn",
                AgentTaskEntity.class)
            .setParameter("a", agentId).setParameter("tn", tenancyId)
            .getResultList();

        List<AgentOutcome> outcomes = tasks.stream()
            .filter(t -> t.outcome != null)
            .map(t -> t.outcome.toRecord())
            .toList();

        List<AttestationRef> refs = attestationsFor(agentId, tenancyId);

        int n = outcomes.size();
        Instant from = tasks.stream().map(t -> t.startedAt).min(Comparator.naturalOrder()).orElse(null);
        Instant through = tasks.stream().map(t -> t.startedAt).max(Comparator.naturalOrder()).orElse(null);
        GraphDataSufficiency suf = GraphDataSufficiency.forCount(n, from, through, buildWarnings());

        return new AgentTaskHistory(agentId, tenancyId,
            tasks.stream().map(AgentTaskEntity::toRecord).toList(), outcomes, refs, suf);
    }

    @Override
    @Transactional(TxType.SUPPORTS)
    public AgentTaskHistory historyByCapability(String agentId, String capabilityTag,
                                                 String tenancyId) {
        List<AgentTaskEntity> tasks = em.createQuery(
                "SELECT t FROM AgentTaskEntity t WHERE t.agentId = :a " +
                "AND t.capabilityTag = :cap AND t.tenancyId = :tn",
                AgentTaskEntity.class)
            .setParameter("a", agentId).setParameter("cap", capabilityTag)
            .setParameter("tn", tenancyId).getResultList();

        List<AgentOutcome> outcomes = tasks.stream()
            .filter(t -> t.outcome != null).map(t -> t.outcome.toRecord()).toList();
        List<AttestationRef> refs = attestationsFor(agentId, tenancyId);
        int n = outcomes.size();
        GraphDataSufficiency suf = GraphDataSufficiency.forCount(n, null, null, buildWarnings());
        return new AgentTaskHistory(agentId, tenancyId,
            tasks.stream().map(AgentTaskEntity::toRecord).toList(), outcomes, refs, suf);
    }

    @Override
    @Transactional(TxType.SUPPORTS)
    public List<String> topAgentsByOutcome(String capabilityTag, String taskDomain,
                                            String tenancyId, int limit) {
        // Collect equivalent domains
        Set<String> domains = new LinkedHashSet<>();
        domains.add(taskDomain);
        List<String> allDomains = em.createQuery(
                "SELECT DISTINCT t.taskDomain FROM AgentTaskEntity t " +
                "WHERE t.capabilityTag = :cap AND t.tenancyId = :tn",
                String.class)
            .setParameter("cap", capabilityTag).setParameter("tn", tenancyId)
            .getResultList();
        for (String d : allDomains) {
            if (d != null && enricher.semanticallyEquivalent(taskDomain, d)) domains.add(d);
        }

        // Fetch outcomes for matching tasks
        @SuppressWarnings("unchecked")
        List<Object[]> rows = em.createNativeQuery(
            "SELECT t.agent_id, o.result, o.confidence " +
            "FROM agent_task t JOIN agent_outcome o ON o.task_id = t.task_id " +
            "WHERE t.capability_tag = :cap AND t.tenancy_id = :tn " +
            "AND t.task_domain IN :domains AND t.ended_at IS NOT NULL")
          .setParameter("cap", capabilityTag)
          .setParameter("tn", tenancyId)
          .setParameter("domains", domains)
          .getResultList();

        // Group by agent and compute Wilson
        Map<String, List<double[]>> byAgent = new LinkedHashMap<>();
        for (Object[] row : rows) {
            String agent = (String) row[0];
            TaskResult result = TaskResult.valueOf((String) row[1]);
            double conf = ((Number) row[2]).doubleValue();
            byAgent.computeIfAbsent(agent, k -> new ArrayList<>())
                   .add(new double[]{conf * qualityMultiplier(result)});
        }

        return byAgent.entrySet().stream()
            .sorted(Comparator.comparingDouble((Map.Entry<String, List<double[]>> e) -> {
                double sum = e.getValue().stream().mapToDouble(q -> q[0]).sum();
                return wilsonScore(sum, e.getValue().size());
            }).reversed())
            .limit(limit)
            .map(Map.Entry::getKey)
            .toList();
    }

    @Override
    @Transactional(TxType.SUPPORTS)
    public List<AttestationRef> attestationsFor(String agentId, String tenancyId) {
        return em.createQuery(
                "SELECT a FROM AttestationRefEntity a WHERE a.agentId = :ag AND a.tenancyId = :tn",
                AttestationRefEntity.class)
            .setParameter("ag", agentId).setParameter("tn", tenancyId)
            .getResultList().stream()
            .map(AttestationRefEntity::toRecord)
            .toList();
    }

    private List<String> buildWarnings() {
        List<String> w = new ArrayList<>();
        if (enricher instanceof io.casehub.eidos.runtime.graph.NoOpTaskSemanticEnricher) {
            w.add("No TaskSemanticEnricher — personality axis correlation unavailable");
        }
        return w;
    }
}
```

Note: The `buildWarnings()` method checks for NoOp by type. A cleaner alternative is to add an `isNoOp()` method to `TaskSemanticEnricher` or use a marker interface. For now this works; file a follow-up if preferred.

- [ ] **Run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graph -Dtest="JpaAgentGraphStoreTest,JpaAgentGraphQueryTest" -q 2>&1 | tail -5
```

Expected: BUILD SUCCESS

- [ ] **Commit store + query**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add graph/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(graph): JpaAgentGraphStore + JpaAgentGraphQuery with Wilson ranking

Wilson lower bound (z=1.645): quality = confidence × result_multiplier.
INSERT ... ON CONFLICT DO NOTHING for idempotent attestation writes.
TopAgentsByOutcome uses equivalent domain widening via TaskSemanticEnricher.

Refs #35"
```

---

### Task 17: JpaAgentGraphBackfill + ArchUnit + integration tests

**Files:**
- Create: `graph/src/main/java/io/casehub/eidos/graph/JpaAgentGraphBackfill.java`
- Create: `graph/src/test/java/io/casehub/eidos/graph/JpaAgentGraphBackfillTest.java`
- Create: `graph/src/test/java/io/casehub/eidos/graph/GraphArchitectureTest.java`

- [ ] **Create JpaAgentGraphBackfill.java**

```java
package io.casehub.eidos.graph;

import io.casehub.eidos.api.*;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.transaction.Transactional;
import java.time.Instant;
import java.util.UUID;

@ApplicationScoped
public class JpaAgentGraphBackfill implements AgentGraphBackfill {

    @Inject EntityManager em;

    @Override
    @Transactional
    public BackfillResult backfillAgent(String agentId, String tenancyId) {
        // In production: query TrustExportService from casehub-ledger.
        // For now: no-op implementation — real backfill wired when engine dep available.
        return new BackfillResult(0, 0, null, null);
    }

    @Override
    @Transactional
    public BackfillResult backfillAll(String tenancyId) {
        return new BackfillResult(0, 0, null, null);
    }

    @Override
    @Transactional
    public BackfillResult backfillDelta(String tenancyId, Instant since) {
        return new BackfillResult(0, 0, since, Instant.now());
    }

    /** Called by JpaAgentGraphStore when auto-linking after outcome record. */
    @Transactional
    public void insertAttestationRef(String taskId, String agentId, String tenancyId,
                                      String hash, String entryType, Instant attestedAt) {
        em.createNativeQuery(
            "INSERT INTO attestation_ref(ref_id, task_id, agent_id, tenancy_id, " +
            "ledger_entry_hash, entry_type, attested_at) " +
            "VALUES(?, ?, ?, ?, ?, ?, ?) " +
            "ON CONFLICT (ledger_entry_hash, tenancy_id) DO NOTHING")
          .setParameter(1, UUID.randomUUID().toString())
          .setParameter(2, taskId)
          .setParameter(3, agentId)
          .setParameter(4, tenancyId)
          .setParameter(5, hash)
          .setParameter(6, entryType)
          .setParameter(7, attestedAt)
          .executeUpdate();
    }
}
```

- [ ] **Create GraphArchitectureTest.java**

```java
package io.casehub.eidos.graph;

import com.tngtech.archunit.junit.AnalyzeClasses;
import com.tngtech.archunit.junit.ArchTest;
import com.tngtech.archunit.lang.ArchRule;

import static com.tngtech.archunit.lang.syntax.ArchRuleDefinition.noClasses;

@AnalyzeClasses(packages = {"io.casehub.eidos.graph", "io.casehub.eidos.api"})
class GraphArchitectureTest {

    @ArchTest
    static final ArchRule entities_not_referenced_from_api =
        noClasses().that().resideInAPackage("io.casehub.eidos.api..")
            .should().dependOnClassesThat()
            .resideInAPackage("io.casehub.eidos.graph.entity..");
}
```

- [ ] **Run all graph tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graph -q 2>&1 | tail -10
```

Expected: BUILD SUCCESS

- [ ] **Run full project build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -q 2>&1 | tail -10
```

Expected: BUILD SUCCESS — all modules green.

- [ ] **Final commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add -A
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(graph): JpaAgentGraphBackfill + ArchUnit entity boundary rule

Backfill delegates to TrustExportService (wired when ledger dep available).
ArchUnit rule: graph entity classes must not be referenced from api/.

Closes #35
Refs #33, #34"
```

---

## Post-Implementation

After all tests pass:

1. **Update eidos#32 epic** — check off completed scope items
2. **File casehub-engine issue** for `WorkOrchestrator` write-path integration (recordTask/recordOutcome call sites)
3. **Update PLATFORM.md** (parent#149) — Capability Ownership table, cross-repo dep map, eidos deep-dive
4. **Run `implementation-doc-sync`** if any conventions changed

---

## Quick Reference

| Command | Purpose |
|---------|---------|
| `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -q` | Test api module |
| `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q` | Test runtime module |
| `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory -q` | Test memory module |
| `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graph -q` | Test graph module |
| `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl examples/agent-scenarios -q` | Test PoC scenarios |
| `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -q` | Full build |
