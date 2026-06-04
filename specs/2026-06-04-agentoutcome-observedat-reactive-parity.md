# Design Spec: AgentOutcome.observedAt + ReactiveAgentGraphQuery Parity

**Issues:** eidos#36, eidos#37  
**Branch:** issue-36-agentoutcome-observedat  
**Date:** 2026-06-04  
**Batched:** #36 and #37 co-located by explicit decision (handover: "Batch with #37"). Same module, same test run, same PR. Independent changes travelling together.

---

## Problem

`AgentOutcome` has no `observedAt` field. The entity stamps `Instant.now()` at persistence time in `from()`, ignoring when the outcome actually occurred. `toRecord()` drops the timestamp entirely. This blocks accurate ledger backfill: `AgentGraphBackfill.backfillAgent()` will eventually populate outcomes from historical `AttestationRef` records (using `attestedAt` as the proxy for task completion time), but has nowhere to put the timestamp.

`ReactiveAgentGraphQuery` is missing `historyByCapability` and `attestationsFor`, both of which exist on the blocking `AgentGraphQuery`. `BlockingToReactiveGraphBridge` is therefore incomplete.

---

## Scope boundary — backfill is a stub

`JpaAgentGraphBackfill` is an explicit stub: all three methods return `BackfillResult(0, 0, ...)`. The production wiring (querying `TrustExportService` from casehub-ledger) is deferred to a separate engine integration issue. Adding `observedAt` to `AgentOutcome` is the transport-layer prerequisite for that future work — the field must exist before the backfill logic can be written. The backfill wiring itself is **out of scope** for this PR.

---

## #36 — AgentOutcome.observedAt

### Semantic definition

`observedAt` is the instant the agent produced the result, as known to the caller at the time of `recordOutcome()`.

- **Live recording:** `Instant.now()` captured at the call site, immediately after the outcome is known. Callers in reactive contexts must capture this before entering the reactive chain — the moment a Uni supplier fires is non-deterministic.
- **Backfill (future):** `AttestationRef.attestedAt()` — this accessor exists on `AttestationRef` today as the sixth field. When backfill is wired, `attestedAt` is the timestamp recorded in the ledger when the attestation was created; it is the closest available proxy for historical task completion time, which may differ from actual task completion by minutes.

No future-timestamp validation. Backfill uses past timestamps; live callers use `Instant.now()`. Rejecting futures adds complexity with no practical benefit.

### Record change

Field order: required before nullable. `AgentOutcome` currently has no compact constructor. The following is the **complete new compact constructor body**:

```java
public record AgentOutcome(
    String taskId,
    TaskResult result,
    double confidence,           // 0.0–1.0, enforced
    Instant observedAt,          // required — when the outcome occurred (see semantic above)
    DegradationReason degradationReason  // nullable — null if not applicable
) {
    public AgentOutcome {
        Objects.requireNonNull(taskId, "taskId must not be null");
        Objects.requireNonNull(result, "result must not be null");
        if (confidence < 0.0 || confidence > 1.0)
            throw new AgentValidationException("confidence", "must be between 0.0 and 1.0");
        Objects.requireNonNull(observedAt, "observedAt must not be null");
    }
}
```

Notes:
- `taskId` and `result` null checks are explicit — consistent with `observedAt`. The enum type system and JPA store contract do not prevent null at construction time.
- The `confidence` range guard is new — not currently enforced at the Java layer (only by the DB `CHECK` constraint). Adding it here fails fast at construction time.

Nullability contract for all five fields:
- `taskId` — required (non-null, compact constructor enforces)
- `result` — required (non-null, compact constructor enforces)
- `confidence` — required (primitive double, 0.0–1.0, enforced in compact constructor)
- `observedAt` — required (non-null, compact constructor enforces)
- `degradationReason` — nullable; null means no degradation applies

`AgentDescriptorValidator` only handles `String` fields. `Objects.requireNonNull` is the correct pattern for required object fields; `AgentValidationException(String fieldName, String message)` exists at `io.casehub.eidos.api.AgentValidationException` and is the correct exception for range violations.

### Entity changes

`AgentOutcomeEntity.from(AgentOutcome o, AgentTaskEntity task)`:
```java
e.observedAt = o.observedAt();   // was: Instant.now()
```

`AgentOutcomeEntity.toRecord()`:
```java
return new AgentOutcome(taskId, TaskResult.valueOf(result), confidence, observedAt,
                        degradationReason != null ? DegradationReason.valueOf(degradationReason) : null);
```

### Schema

No change. `observed_at TIMESTAMP WITH TIME ZONE NOT NULL` is already in V3__agent_graph.sql. `TIMESTAMP WITH TIME ZONE` in H2/PostgreSQL preserves microsecond precision; nanoseconds are truncated at the JDBC boundary.

### Call sites

Five existing `AgentOutcome(...)` constructions in tests — all need `Instant.now()` inserted as the fourth arg (before `degradationReason`) after the reorder. The field position change breaks every positional construction site — that is the contract.

### Cross-repo impact

Engine depends on `casehub-eidos-api` but contains no `AgentOutcome` constructions today. The write-path integration (`WorkOrchestrator → AgentGraphStore.recordOutcome()`) is not yet wired — deferred per eidos#32. When engine implements it, it will pass `Instant.now()` at the point the outcome is observed. That is a future compile-time break — the correct forcing function. No other downstream repo constructs `AgentOutcome` today.

### Record evolution position

Records are correct for `AgentOutcome`. It is a value type exchanged within the eidos module. Compile-time breakage at every construction site is the contract — it forces every caller to be explicit about `observedAt`. No Builder or `@With` pattern.

### Test

**Round-trip assertion** in `JpaAgentGraphStoreTest.recordOutcome_links_to_task`:

```java
Instant before = Instant.now().truncatedTo(ChronoUnit.MICROS);
store.recordOutcome(new AgentTaskId("t2", "agent-a", "t1"),
    new AgentOutcome("t2", TaskResult.SUCCEEDED, 0.9, before, null));
var outcome = query.agentHistory("agent-a", "t1").outcomes().get(0);
assertThat(outcome.observedAt()).isEqualTo(before);
```

`truncatedTo(ChronoUnit.MICROS)` accounts for nanosecond precision loss at the JDBC boundary — prevents a false failure. `isEqualTo` is correct: `Instant.equals()` and `compareTo()` both compare epoch-seconds and nanos, producing identical results for equal instants. No special comparator needed.

---

## #37 — ReactiveAgentGraphQuery Parity

### SPI change

Add to `ReactiveAgentGraphQuery`:

```java
Uni<AgentTaskHistory> historyByCapability(String agentId, String capabilityTag, String tenancyId);
Uni<List<AttestationRef>> attestationsFor(String agentId, String tenancyId);
```

Both `AttestationRef` and `AgentTaskHistory` are defined in `casehub-eidos-api`. No cross-module dependency.

**`Uni<List<AttestationRef>>` vs `Multi<AttestationRef>`:** `Multi` is the idiomatic reactive form for collections. `Uni<List<T>>` is used here deliberately — it matches the blocking return type and the bridge wraps synchronous JPA anyway. When the bridge materialises the full list before emitting, the distinction carries no practical consequence. A genuinely non-blocking implementation can return `Multi` independently; the bridge is not the place to introduce that asymmetry.

### Bridge change

Add to `BlockingToReactiveGraphBridge`:

```java
@Override
public Uni<AgentTaskHistory> historyByCapability(String agentId, String capabilityTag, String tenancyId) {
    return Uni.createFrom()
              .item(() -> blocking.historyByCapability(agentId, capabilityTag, tenancyId))
              .runSubscriptionOn(Infrastructure.getDefaultWorkerPool());
}

@Override
public Uni<List<AttestationRef>> attestationsFor(String agentId, String tenancyId) {
    return Uni.createFrom()
              .item(() -> blocking.attestationsFor(agentId, tenancyId))
              .runSubscriptionOn(Infrastructure.getDefaultWorkerPool());
}
```

**Mutiny pattern:** `Uni.createFrom().item(Supplier<T>)` — the lazy supplier form. `runSubscriptionOn(Infrastructure.getDefaultWorkerPool())` moves supplier execution (including the blocking JPA call) onto the worker pool. `emitOn` would only move downstream processing, not the supplier itself — incorrect for this use case.

**Exception propagation:** Mutiny wraps any supplier exception as a Uni failure automatically. `JpaAgentGraphQuery` declares no checked exceptions. No wrapping needed — consistent with the two existing bridge methods.

**Architectural note:** This bridge is worker-thread-per-call, not genuine async I/O. A Vert.x reactive PG driver implementation is the correct long-term path; out of scope here.

### Bridge testability

`BlockingToReactiveGraphBridge` has `@Inject AgentGraphQuery blocking` — no constructor accepting the dependency. A plain Mockito unit test cannot construct it without CDI.

**Fix:** add both an explicit no-arg constructor (CDI path) and a package-private constructor (test path) to `BlockingToReactiveGraphBridge`:

```java
public BlockingToReactiveGraphBridge() {}           // CDI — Weld uses this for instantiation before @Inject

BlockingToReactiveGraphBridge(AgentGraphQuery blocking) { // test path
    this.blocking = blocking;
}
```

The explicit no-arg constructor is mandatory: adding any explicit constructor removes the compiler-generated default. Without it, Weld cannot instantiate the bean and the application fails at startup. The CDI container uses the no-arg constructor + `@Inject` field path. Tests use the package-private constructor directly.

### Bridge test

New class `BlockingToReactiveGraphBridgeTest`, plain Mockito, no CDI container:

```java
class BlockingToReactiveGraphBridgeTest {

    AgentGraphQuery mockBlocking = mock(AgentGraphQuery.class);
    BlockingToReactiveGraphBridge bridge = new BlockingToReactiveGraphBridge(mockBlocking);

    @Test
    void historyByCapability_delegates_to_blocking() {
        var expected = new AgentTaskHistory(...);
        when(mockBlocking.historyByCapability("a", "cr", "t1")).thenReturn(expected);
        assertThat(bridge.historyByCapability("a", "cr", "t1").await().indefinitely())
            .isEqualTo(expected);
    }

    @Test
    void attestationsFor_delegates_to_blocking() {
        var expected = List.of(new AttestationRef(...));
        when(mockBlocking.attestationsFor("a", "t1")).thenReturn(expected);
        assertThat(bridge.attestationsFor("a", "t1").await().indefinitely())
            .isEqualTo(expected);
    }
}
```

Two tests: one per new method. Verifies delegation only — blocking query correctness is covered by `JpaAgentGraphQueryTest`.

---

## Cross-cutting

### Wilson ranking

Wilson scoring uses `confidence × qualityMultiplier(result)` — no `observedAt` weighting. Ranking unaffected.

### ArchUnit

Single rule in `GraphArchitectureTest`: `entities_must_not_be_referenced_from_api`. `AgentOutcome` is an api record; `ReactiveAgentGraphQuery` is an api interface; `BlockingToReactiveGraphBridge` is in `runtime`. No rule is touched.

### REST surface

Eidos is a library module with no REST layer. `AgentOutcome` is not serialized in any HTTP response body. No OpenAPI impact.

### Reactive store path

`AgentGraphStore` has no reactive counterpart. Only write path is through `JpaAgentGraphStore.recordOutcome()`. No additional change needed.
