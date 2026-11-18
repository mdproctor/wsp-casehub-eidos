# Dialect-Aware Upsert Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #249 — dialect-aware upsert for JpaDispositionSignalStore
**Issue group:** #249

**Goal:** Eliminate concurrent-insert races in both JPA signal stores by using dialect-aware native SQL upsert (PostgreSQL `ON CONFLICT`, H2 `MERGE`).

**Architecture:** Detect the database dialect at `@PostConstruct` via the JDBC URL from EntityManager properties. Each store selects the appropriate SQL string for its upsert operation. A shared `JpaDialect` utility encapsulates detection to avoid duplication.

**Tech Stack:** Java 21, Quarkus 3.32.2, JPA/Hibernate, H2 (tests), PostgreSQL (production target)

## Global Constraints

- Tests run against H2 with `MODE=PostgreSQL` (`jdbc:h2:mem:eidostest;MODE=PostgreSQL;DB_CLOSE_DELAY=-1`)
- Both stores are `@IfBuildProperty(name = "casehub.eidos.reactive.enabled", stringValue = "false", enableIfMissing = true)`
- No schema changes — existing PKs support `ON CONFLICT` directly
- Use `ide_create_file` for new files, `ide_replace_member` / `ide_insert_member` for modifications

---

## Batch 1: Dialect-aware upsert for both JPA signal stores

### Task 1: JpaDialect utility and JpaDispositionSignalStore upsert

**Files:**
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/health/jpa/JpaDialect.java`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/health/jpa/JpaDispositionSignalStore.java`
- Create: `runtime/src/test/java/io/casehub/eidos/runtime/health/jpa/JpaDispositionSignalStoreTest.java`

**Interfaces:**
- Produces: `JpaDialect.isPostgres(EntityManager) → boolean` — used by Task 2

- [ ] **Step 1: Write failing test — JpaDispositionSignalStoreTest**

Create test matching the existing `JpaBehavioralSignalStoreTest` pattern (`@QuarkusTest`, `@TestTransaction`):

```java
package io.casehub.eidos.runtime.health.jpa;

import io.casehub.eidos.api.DispositionSignalStore;
import io.quarkus.test.TestTransaction;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
@TestTransaction
class JpaDispositionSignalStoreTest {

    @Inject DispositionSignalStore store;

    @Test
    void recordActivation_creates_new_entry() {
        store.recordActivation("a1", "t1", "Ti");
        assertThat(store.activationWeights("a1", "t1"))
                .containsEntry("Ti", 1.0);
    }

    @Test
    void recordActivation_increments_existing_entry() {
        store.recordActivation("a1", "t1", "Ti");
        store.recordActivation("a1", "t1", "Ti");
        store.recordActivation("a1", "t1", "Ti");
        assertThat(store.activationWeights("a1", "t1"))
                .containsEntry("Ti", 3.0);
    }

    @Test
    void activationWeights_returns_multiple_terms() {
        store.recordActivation("a1", "t1", "Ti");
        store.recordActivation("a1", "t1", "Fe");
        store.recordActivation("a1", "t1", "Fe");
        var weights = store.activationWeights("a1", "t1");
        assertThat(weights).containsEntry("Ti", 1.0).containsEntry("Fe", 2.0);
    }

    @Test
    void activationWeights_empty_when_none() {
        assertThat(store.activationWeights("a1", "t1")).isEmpty();
    }

    @Test
    void decay_reduces_weights() {
        store.recordActivation("a1", "t1", "Ti");
        store.recordActivation("a1", "t1", "Ti");
        store.decay("a1", "t1", 0.5);
        assertThat(store.activationWeights("a1", "t1"))
                .containsEntry("Ti", 1.0);
    }

    @Test
    void decay_removes_zero_weight_entries() {
        store.recordActivation("a1", "t1", "Ti");
        store.decay("a1", "t1", 0.0);
        assertThat(store.activationWeights("a1", "t1")).isEmpty();
    }

    @Test
    void clear_removes_all_entries() {
        store.recordActivation("a1", "t1", "Ti");
        store.recordActivation("a1", "t1", "Fe");
        store.clear("a1", "t1");
        assertThat(store.activationWeights("a1", "t1")).isEmpty();
    }

    @Test
    void isolation_by_agent_id() {
        store.recordActivation("a1", "t1", "Ti");
        assertThat(store.activationWeights("a2", "t1")).isEmpty();
    }

    @Test
    void isolation_by_tenancy_id() {
        store.recordActivation("a1", "t1", "Ti");
        assertThat(store.activationWeights("a1", "t2")).isEmpty();
    }
}
```

- [ ] **Step 2: Run test to verify it passes (baseline)**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=JpaDispositionSignalStoreTest -f /Users/mdproctor/claude/casehub/eidos/pom.xml`

Expected: PASS — the existing MERGE SQL works on H2. These tests establish baseline coverage before we change the SQL.

- [ ] **Step 3: Create JpaDialect utility**

```java
package io.casehub.eidos.runtime.health.jpa;

import jakarta.persistence.EntityManager;

final class JpaDialect {

    private JpaDialect() {}

    static boolean isPostgres(EntityManager em) {
        var props = em.getEntityManagerFactory().getProperties();
        var url = (String) props.get("jakarta.persistence.jdbc.url");
        if (url == null) {
            url = (String) props.get("javax.persistence.jdbc.url");
        }
        return url != null && url.startsWith("jdbc:postgresql");
    }
}
```

- [ ] **Step 4: Add dialect-aware upsert to JpaDispositionSignalStore**

Add a `boolean postgres` field and `@PostConstruct init()` method. Replace the `recordActivation` body with dialect-selected SQL:

```java
// New field:
private boolean postgres;

// New method:
@PostConstruct
void init() {
    postgres = JpaDialect.isPostgres(em);
}
```

Replace `recordActivation` body — when `postgres`:
```sql
INSERT INTO disposition_signal (agent_id, tenancy_id, function_term, weight)
VALUES (:agentId, :tenancyId, :functionTerm, 1.0)
ON CONFLICT (agent_id, tenancy_id, function_term)
DO UPDATE SET weight = disposition_signal.weight + 1
```

When not postgres, keep existing MERGE SQL unchanged.

- [ ] **Step 5: Run tests to verify they still pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=JpaDispositionSignalStoreTest -f /Users/mdproctor/claude/casehub/eidos/pom.xml`

Expected: PASS — H2 path unchanged, dialect correctly detected as not-postgres.

- [ ] **Step 6: Commit**

```bash
git add runtime/src/main/java/io/casehub/eidos/runtime/health/jpa/JpaDialect.java \
       runtime/src/main/java/io/casehub/eidos/runtime/health/jpa/JpaDispositionSignalStore.java \
       runtime/src/test/java/io/casehub/eidos/runtime/health/jpa/JpaDispositionSignalStoreTest.java
git commit -m "feat: dialect-aware upsert for JpaDispositionSignalStore

Add JpaDialect utility for JDBC URL-based dialect detection.
JpaDispositionSignalStore selects PostgreSQL INSERT ON CONFLICT
or H2 MERGE at @PostConstruct.

Add JpaDispositionSignalStoreTest (9 tests).

Refs #249"
```

### Task 2: JpaBehavioralSignalStore native SQL upsert

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/health/jpa/JpaBehavioralSignalStore.java`
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/health/jpa/JpaBehavioralSignalStoreTest.java`

**Interfaces:**
- Consumes: `JpaDialect.isPostgres(EntityManager) → boolean` from Task 1

- [ ] **Step 1: Add upsert-idempotency test to JpaBehavioralSignalStoreTest**

Add test verifying that double-recording the same key increments rather than duplicates (this test passes with find-then-persist but validates the native SQL path too):

```java
@Test
void record_same_key_twice_increments_not_duplicates() {
    store.record("a1", "t1", "code-review", "rust", DECLINE);
    store.record("a1", "t1", "code-review", "rust", DECLINE);
    assertThat(store.count("a1", "t1", "code-review", "rust", DECLINE)).isEqualTo(2);
    assertThat(store.learned("a1", "t1", "code-review", DECLINE)).containsEntry("rust", 2);
}
```

- [ ] **Step 2: Run tests to verify baseline passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=JpaBehavioralSignalStoreTest -f /Users/mdproctor/claude/casehub/eidos/pom.xml`

Expected: PASS — existing find-then-persist handles sequential calls correctly.

- [ ] **Step 3: Convert JpaBehavioralSignalStore.record() to native SQL upsert**

Add `boolean postgres` field and `@PostConstruct init()` (same pattern as Task 1). Replace the `record()` method body:

When `postgres`:
```sql
INSERT INTO behavioral_signal (agent_id, tenancy_id, capability_name, qualifier, signal_type,
                               signal_count, last_recorded, expires_at)
VALUES (:agentId, :tenancyId, :capabilityName, :qualifier, :signalType,
        1, :lastRecorded, :expiresAt)
ON CONFLICT (agent_id, tenancy_id, capability_name, qualifier, signal_type)
DO UPDATE SET signal_count = behavioral_signal.signal_count + 1,
              last_recorded = EXCLUDED.last_recorded,
              expires_at = EXCLUDED.expires_at
```

When not postgres (H2):
```sql
MERGE INTO behavioral_signal AS bs
USING (VALUES (CAST(:agentId AS VARCHAR), CAST(:tenancyId AS VARCHAR),
               CAST(:capabilityName AS VARCHAR), CAST(:qualifier AS VARCHAR),
               CAST(:signalType AS VARCHAR),
               CAST(1 AS INT), CAST(:lastRecorded AS TIMESTAMP WITH TIME ZONE),
               CAST(:expiresAt AS TIMESTAMP WITH TIME ZONE)))
AS src(agent_id, tenancy_id, capability_name, qualifier, signal_type,
       signal_count, last_recorded, expires_at)
ON bs.agent_id = src.agent_id
AND bs.tenancy_id = src.tenancy_id
AND bs.capability_name = src.capability_name
AND bs.qualifier = src.qualifier
AND bs.signal_type = src.signal_type
WHEN MATCHED THEN UPDATE SET signal_count = bs.signal_count + 1,
                             last_recorded = src.last_recorded,
                             expires_at = src.expires_at
WHEN NOT MATCHED THEN INSERT (agent_id, tenancy_id, capability_name, qualifier, signal_type,
                              signal_count, last_recorded, expires_at)
                     VALUES (src.agent_id, src.tenancy_id, src.capability_name, src.qualifier,
                             src.signal_type, src.signal_count, src.last_recorded, src.expires_at)
```

Important: After the native SQL, add `em.flush(); em.clear();` to evict any stale persistence context — subsequent `em.find()` calls in `count()` will then read from the DB.

- [ ] **Step 4: Run all JPA store tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="JpaBehavioralSignalStoreTest,JpaDispositionSignalStoreTest" -f /Users/mdproctor/claude/casehub/eidos/pom.xml`

Expected: ALL PASS

- [ ] **Step 5: Run full runtime test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -f /Users/mdproctor/claude/casehub/eidos/pom.xml`

Expected: ALL PASS — no regressions

- [ ] **Step 6: Commit**

```bash
git add runtime/src/main/java/io/casehub/eidos/runtime/health/jpa/JpaBehavioralSignalStore.java \
       runtime/src/test/java/io/casehub/eidos/runtime/health/jpa/JpaBehavioralSignalStoreTest.java
git commit -m "feat: dialect-aware upsert for JpaBehavioralSignalStore

Replace find-then-persist with native SQL upsert. PostgreSQL uses
INSERT ON CONFLICT, H2 uses MERGE. Eliminates concurrent-insert race
on the NOT MATCHED path.

Refs #249"
```

## References

- [specs/issue-249-dialect-aware-upsert/2026-11-18-dialect-aware-upsert-design.md] — design spec
- [runtime/.../jpa/JpaDispositionSignalStore.java] — current MERGE SQL
- [runtime/.../jpa/JpaBehavioralSignalStore.java] — current find-then-persist
- [runtime/.../jpa/JpaBehavioralSignalStoreTest.java] — test pattern
- [runtime/src/test/resources/application.properties] — H2 test config
- [V1__initial_schema.sql:117-123] — disposition_signal DDL
- [V1__initial_schema.sql:80-90] — behavioral_signal DDL
- [GitHub #249] — dialect-aware upsert for JpaDispositionSignalStore
- [GitHub #244] — previous MERGE fix that surfaced this issue
