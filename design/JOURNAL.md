# Design Journal — issue-36-agentoutcome-observedat

## AgentOutcome.observedAt — semantic and validation design

**Issue:** eidos#36

`observedAt` is defined as the instant the agent produced the result, as known to the caller at the time of `recordOutcome()`. For live recording this is `Instant.now()` at the call site. For future backfill, it will be `AttestationRef.attestedAt()` — the closest ledger proxy for historical task completion time.

Field order decision: required fields before nullable. New order is `taskId, result, confidence, observedAt (required), degradationReason (nullable)`. The reorder costs nothing since it's already a breaking API change.

**Compact constructor validation decisions:**
- `taskId`, `result`, `observedAt`: `Objects.requireNonNull` with bare field name message (consistent with `AgentQuery` convention, not `AgentDescriptorValidator` verbose form)
- `confidence` range: `Double.isNaN(confidence) || confidence < 0.0 || confidence > 1.0` — the `isNaN` guard is required because IEEE 754 comparison with NaN always returns false, so the range check alone silently accepts NaN. The DB `CHECK` constraint does catch it at persist time, but the compact constructor contract must hold independently of the persistence layer.
- `AgentValidationException("confidence", "must be between 0.0 and 1.0")` is the correct exception type — it already exists at `io.casehub.eidos.api.AgentValidationException`.

**Backfill scope:** `JpaAgentGraphBackfill` remains a stub. The `observedAt` field is the transport-layer prerequisite; the backfill wiring (reading `TrustExportService` from casehub-ledger) is deferred to a separate engine integration issue.

**No schema change:** `observed_at TIMESTAMP WITH TIME ZONE NOT NULL` already existed in V3__agent_graph.sql. Entity `from()` was simply fixed to use `o.observedAt()` instead of `Instant.now()`.

## ReactiveAgentGraphQuery parity — bridge constructor pattern

**Issue:** eidos#37

`ReactiveAgentGraphQuery` is not build-gated (unlike `ReactiveAgentRegistry` / `ReactiveCapabilityHealth`). `BlockingToReactiveGraphBridge` is `@DefaultBean @ApplicationScoped`, always active, wraps blocking JPA via `Uni.createFrom().item(Supplier).runSubscriptionOn(Infrastructure.getDefaultWorkerPool())`.

**Constructor pair decision:** Adding any explicit constructor removes the compiler-generated implicit no-arg constructor. CDI (Weld) requires a no-arg constructor to instantiate the bean before `@Inject` field injection. Both constructors are required:
- `public BlockingToReactiveGraphBridge() {}` — CDI path
- `BlockingToReactiveGraphBridge(AgentGraphQuery blocking)` (package-private) — test path

**Mutiny pattern:** `runSubscriptionOn` moves the supplier execution (including the blocking JPA call) onto the worker pool. `emitOn` only moves downstream processing — wrong for offloading blocking calls. `item(Supplier<T>)` is the lazy form; `item(T)` evaluates eagerly before subscription.

**`Uni<List<AttestationRef>>` vs `Multi<AttestationRef>`:** deliberate. Bridge wraps synchronous JPA; `Multi` would not change behaviour. `Uni<List>` matches the blocking return type for bridge simplicity.

**Bridge architectural note:** This is worker-thread-per-call, not genuine async I/O. Acknowledged as a non-blocking fallback pending a Vert.x reactive PG driver implementation.
