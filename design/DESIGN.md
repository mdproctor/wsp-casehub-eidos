# CaseHub Eidos — Design Document

## AgentRegistry

`AgentRegistry` (blocking) and `ReactiveAgentRegistry` (reactive, build-gated) provide the SPI for registering and discovering `AgentDescriptor` instances. All `findById(agentId, tenancyId)` implementations enforce `Objects.requireNonNull` on both `agentId` and `tenancyId`. Without explicit guards, failure modes differ by backend: `ConcurrentHashMap` throws an unhelpful messageless NPE; JPA silently returns empty. The null contract is documented via `@throws NullPointerException` on both SPI interfaces and on `AgentQuery`'s compact constructor. All null messages use the bare field name convention (`"agentId"`, `"tenancyId"`).

Implementations:
- `InMemoryAgentRegistry` / `InMemoryReactiveAgentRegistry` — in-memory, `casehub-eidos-memory`, `@Alternative @Priority(1)`
- `JpaAgentRegistry` — default, `@ApplicationScoped`, active when `casehub.eidos.reactive.enabled` is false or absent
- `JpaReactiveAgentRegistry` — build-gated via `@IfBuildProperty(name="casehub.eidos.reactive.enabled", stringValue="true")`

## AgentOutcome — observedAt and compact constructor validation

`observedAt` is the instant the agent produced the result, as known to the caller at the time of `recordOutcome()`. For live recording: `Instant.now()` at the call site. For backfill (still a stub): `AttestationRef.attestedAt()` — the closest ledger proxy for historical task completion time.

Field order: `taskId, result, confidence, observedAt (required), degradationReason (nullable)` — required fields before nullable.

Compact constructor validation:
- `taskId`, `result`, `observedAt`: `Objects.requireNonNull` with bare field name message (consistent with `AgentQuery` convention)
- `confidence` range: `Double.isNaN(confidence) || confidence < 0.0 || confidence > 1.0` — `isNaN` guard is required because IEEE 754 comparison with NaN always returns `false`; the range check alone silently accepts NaN
- `AgentValidationException("confidence", "must be between 0.0 and 1.0")` is the correct exception type

## ReactiveAgentGraphQuery — bridge constructor pattern

`ReactiveAgentGraphQuery` is **not** build-gated (unlike `ReactiveAgentRegistry` / `ReactiveCapabilityHealth`). `BlockingToReactiveGraphBridge` is `@DefaultBean @ApplicationScoped`, always active.

Mutiny bridge pattern: `Uni.createFrom().item(Supplier).runSubscriptionOn(Infrastructure.getDefaultWorkerPool())`. `runSubscriptionOn` moves supplier execution onto the worker pool; `emitOn` only moves downstream processing — wrong for offloading blocking calls.

CDI + test constructor pair: adding any explicit constructor removes the compiler-generated no-arg. Weld needs the no-arg to instantiate `@ApplicationScoped` beans before `@Inject` field injection. Both are required: `public Bridge() {}` (CDI) + package-private `Bridge(Dep dep)` (test path).

`Uni<List<AttestationRef>>` over `Multi<AttestationRef>` is deliberate — bridge wraps synchronous JPA; `Multi` would not change behaviour. Bridge is worker-thread-per-call, not genuine async I/O.
