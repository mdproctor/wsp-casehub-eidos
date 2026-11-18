# Dialect-Aware Upsert for JPA Signal Stores

**Issue:** casehubio/eidos#249
**Scale:** S | **Complexity:** Low

## Problem

`JpaDispositionSignalStore.recordActivation()` uses SQL `MERGE INTO` for atomic upsert. H2 handles this atomically, but PostgreSQL's `MERGE` does not use the unique index for conflict detection on the NOT MATCHED path. Two concurrent threads that both see no existing row will both attempt INSERT, producing a unique constraint violation — one activation is silently lost.

`JpaBehavioralSignalStore.record()` uses find-then-persist, which has the same concurrent-insert race: two threads both find no existing entity and both persist, violating the PK constraint.

## Solution

Convert both stores to native SQL upsert with dialect-aware SQL selection:

- **H2:** `MERGE INTO` (SQL standard, atomically correct on H2)
- **PostgreSQL:** `INSERT ... ON CONFLICT (...) DO UPDATE` (uses the unique index for conflict detection)

Detect the dialect at `@PostConstruct` via the JDBC URL from EntityManager properties. Extract detection into a shared utility to avoid duplication.

## Changes

### New: `JpaDialect` utility

Static utility in `io.casehub.eidos.runtime.health.jpa`:

```java
final class JpaDialect {
    static boolean isPostgres(EntityManager em) {
        // Check JDBC URL from entity manager factory properties
    }
}
```

### Modified: `JpaDispositionSignalStore`

Add `@PostConstruct` to detect dialect. `recordActivation()` selects SQL:

**PostgreSQL:**
```sql
INSERT INTO disposition_signal (agent_id, tenancy_id, function_term, weight)
VALUES (:agentId, :tenancyId, :functionTerm, 1.0)
ON CONFLICT (agent_id, tenancy_id, function_term)
DO UPDATE SET weight = disposition_signal.weight + 1
```

**H2:** Keep existing `MERGE INTO` (unchanged).

### Modified: `JpaBehavioralSignalStore`

Replace find-then-persist with native SQL upsert in `record()`.

**PostgreSQL:**
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

**H2:**
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

### New: `JpaDispositionSignalStoreTest`

Currently missing. Add test class matching `JpaBehavioralSignalStoreTest` patterns:
- Record activation creates new entry
- Record activation increments existing entry (upsert path)
- Activation weights returned correctly
- Decay reduces weights
- Clear removes all entries

### Modified: `JpaBehavioralSignalStoreTest`

Add test for upsert-idempotency: calling `record()` twice for the same key increments `signal_count` to 2 (verifies upsert, not duplicate insert).

## Out of Scope

- Schema changes (tables and PKs are correct as-is)
- Reactive store variants (use different patterns)
- In-memory store variants (no SQL involved)

## References

- `runtime/.../jpa/JpaDispositionSignalStore.java:24-38` — current MERGE SQL
- `runtime/.../jpa/JpaBehavioralSignalStore.java:40-54` — current find-then-persist
- `runtime/.../jpa/DispositionSignalEntity.java` — PK: (agent_id, tenancy_id, function_term)
- `runtime/.../jpa/BehavioralSignalEntity.java` — PK: (agent_id, tenancy_id, capability_name, qualifier, signal_type)
- `V1__initial_schema.sql:117-123` — disposition_signal DDL
- `V1__initial_schema.sql:80-90` — behavioral_signal DDL
- casehubio/eidos#244 — previous MERGE fix that surfaced this issue
