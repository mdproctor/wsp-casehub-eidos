# casehub-eidos Phase 4: Knowledge Graph

**Issue:** casehubio/eidos#32  
**Date:** 2026-06-02 (reviewed 2026-06-03)  
**Status:** Design approved — PoC gate open

---

## What This Is and Why

casehub-eidos currently knows what agents **claim to be**: declared capabilities,
disposition axes, slot, epistemic domain confidence. What the platform lacks is any
record of what agents have **actually done** and how those actions went.

The knowledge graph closes that gap. It records the observable history of agent
activity — tasks dispatched, outcomes observed, attestations from the ledger that
back those outcomes. The result is:

- **Evidence-grounded trust scores.** casehub-ledger can trace a trust score to
  specific task outcomes and ledger entry hashes, rather than accumulating abstract
  attestation weights.
- **Reputation-informed routing.** casehub-engine can ask not just "is this agent's
  capability operable now?" but "has this agent done this successfully before, and
  how often?"
- **Production personality validation.** The personality axes defined by the real-world
  eval profiles (eidos#23) — riskAppetite, socialOrient, ruleFollowing, autonomy — can
  be tested against observed behavior, not just declared values.
- **Auditable agent lineage.** A compliance query "why does agent X have a 0.73 trust
  score?" becomes: walk descriptor → task history → outcomes → ledger hash chain.

**What the graph does not do:** compute trust scores (casehub-ledger), route tasks
(casehub-engine), store messages (casehub-qhorus), or hold any domain-specific
knowledge. It is domain-agnostic infrastructure that application-tier repos can
optionally enhance with semantic context.

---

## Design Philosophy

**The graph is domain-agnostic by construction.** eidos records `capabilityTag`,
`taskDomain`, `result`, and `RetentionScore`. It does not know what a "code-review in
rust" task means for riskAppetite, or whether "security" and "safety-critical" are
semantically equivalent. That knowledge belongs in application repos.

**Semantic interpretation is a pull interface, not a push interface.** When eidos
needs to know whether two domains are equivalent, or which disposition axes a task
type expresses, it asks a `TaskSemanticEnricher` at query time. Applications provide
enrichers. eidos asks; the enricher answers.

**Partial coverage is honest and documented.** The graph is reliable for structural
queries without any enricher. Personality axis correlation and semantic domain
equivalence require one. Every query result carries a `GraphDataSufficiency`
assessment that tells callers exactly how much they can trust the result.

**The DB abstraction seam is the SPI, not a wrapper.** SPIs speak graph semantics.
The JPA implementation is entirely internal — entities never appear in `api/`.
A future graph DB implementation is `@Alternative @Priority(1)` against the same
SPIs.

---

## Pre-requisite: PoC Validation Gate

**Phase 4b (full infrastructure) must not begin until Phase 4a passes.**

### Phase 4a — Pure In-Memory Proof of Concept

Implement `InMemoryAgentGraph` as a plain Java class in `examples/agent-scenarios/`.
No CDI, no JPA, no new modules — just the candidate API exercised directly against
in-memory data. Write the following plain JUnit 5 tests:

**Version 1 — Inference from structural data only (no enricher)**

| Test | Pass criteria |
|------|--------------|
| `v1_routingPrefersAgentWithBetterOutcomeHistory` | Agent A (20 tasks, 0.78 avg quality) beats Agent B (5 tasks, 0.90 avg quality) — Wilson correctly penalises smaller sample |
| `v1_historyQueryReturnsByCapabilityAndDomain` | Retrieval returns correct tasks, ordered by time |
| `v1_attestationChainLinksOutcomeToLedgerHash` | Walk descriptor → task → outcome → attestation ref → hash |
| `v1_insufficiencyWarningBelowThreshold` | Fewer than 5 samples → `INSUFFICIENT` + warning text |
| `v1_systemFunctionsNormallyWithNoHistory` | Zero graph data → no routing change, no exception |

**Version 2 — With `TaskSemanticEnricher`**

| Test | Pass criteria |
|------|--------------|
| `v2_personalityMismatchDetectedViaEnricher` | Agent declares bold riskAppetite; enricher maps security tasks to riskAppetite; outcomes show conservative confidence → mismatch diagnosed |
| `v2_semanticEquivalenceWidensOutcomePool` | Enricher says "security" ≡ "safety-critical" → sample count rises, sufficiency level improves |
| `v2_routingPicksDifferentAgentThanV1` | Same data — V1 and V2 reach different routing decisions; divergence proves enricher adds value |
| `v2_absenceOfEnricherRecordedInWarnings` | No enricher → `warnings` contains "No TaskSemanticEnricher — personality axis correlation unavailable" |

**`v1_routingPrefersAgentWithBetterOutcomeHistory` must validate the Wilson formula
specifically:** use Agent A (20 outcomes, all SUCCEEDED at confidence 0.78 → Wilson
score ≈ 0.62) and Agent B (5 outcomes, all SUCCEEDED at confidence 0.90 → Wilson
score ≈ 0.53). Agent A must rank above Agent B. This validates that Wilson correctly
discounts smaller samples even when raw quality is higher.

**`v2_routingPicksDifferentAgentThanV1` is the critical test.** If V1 and V2 always
agree, the enricher adds no discriminating value — scope back before building
infrastructure.

**Evaluation:** V1 tests 1–4 must pass cleanly. `v2_routingPicksDifferentAgentThanV1`
must produce genuine divergence. If the PoC reveals limitations, scope them out of
Phase 4b and file issues. Partial coverage (70% of use cases) is acceptable and
honest. Infrastructure that cannot demonstrate validated utility is not.

---

## Architecture

```
casehub-engine
    │  calls AgentGraphStore SPI at dispatch and completion
    ▼
AgentTask → AgentOutcome  (written to eidos graph)
                │
                └── AttestationRef ←── eidos reads casehub-ledger by actorId
                                        (write-forward linkage + batch backfill)

Queries:
AgentGraphQuery.agentHistory()          → task list + outcomes + sufficiency
AgentGraphQuery.topAgentsByOutcome()    → ranked agents for routing
AgentGraphQuery.attestationsFor()       → evidence chain for trust audit

TaskSemanticEnricher (optional, application-tier, pulled at query time):
    eidos asks → enricher answers → eidos applies
```

**The abstraction seam:**
```
AgentGraphQuery / AgentGraphStore  (SPI — casehub-eidos-api — pure Java)
    │
    ▼  JpaAgentGraphQuery / JpaAgentGraphStore  (@ApplicationScoped — casehub-eidos-graph)
    │
    ▼  AgentTaskEntity / AgentOutcomeEntity / AttestationRefEntity  (internal entities)
    │
    ▼  agent_task / agent_outcome / attestation_ref  (PostgreSQL)

Future graph DB:
    ▼  Neo4jAgentGraphQuery  (@Alternative @Priority(1) — casehub-eidos-graph-neo4j)
```

---

## Module Structure

```
casehub-eidos/
├── api/              casehub-eidos-api  — new types and SPIs added
├── runtime/          casehub-eidos      — runtime-module defaults added; schema redesigned
├── persistence-memory/  casehub-eidos-memory  — InMemoryAgentStateStore key updated (A1)
├── deployment/          casehub-eidos-deployment  — unchanged
├── vocab/               casehub-eidos-vocab  — unchanged
├── graph/  ← NEW        casehub-eidos-graph  (Jandex library — no quarkus:build goal)
│   ├── src/main/java/io/casehub/eidos/graph/
│   │   ├── entity/         (internal — never exported, never in api/)
│   │   │   ├── AgentTaskEntity.java
│   │   │   ├── AgentOutcomeEntity.java
│   │   │   └── AttestationRefEntity.java
│   │   ├── JpaAgentGraphStore.java     @ApplicationScoped
│   │   ├── JpaAgentGraphQuery.java     @ApplicationScoped
│   │   └── JpaAgentGraphBackfill.java  @ApplicationScoped
│   └── src/main/resources/db/eidos/migration/
│       └── V3__agent_graph.sql         (graph tables only — V1 and V2 are in runtime)
├── eval/             (unchanged — test-only)
└── examples/agent-scenarios/
    └── src/test/java/.../
        ├── V1GraphScenarioTest.java   ← PoC V1 (plain JUnit)
        └── V2GraphScenarioTest.java   ← PoC V2 (plain JUnit)
```

**casehub-eidos-graph is a Jandex library** (same pattern as `casehub-eidos-memory`,
`casehub-eidos-vocab`) — no quarkus:build goal, no deployment module. `@ApplicationScoped`
beans activate by CDI discovery. Migrations at `db/eidos/migration/` are covered by
`EidosProcessor`'s existing glob — the glob scans the full classpath, so V3 from the
graph JAR is automatically included alongside V1 and V2 from the runtime JAR. No
change to EidosProcessor needed.

**casehub-eidos-graph activates by classpath presence.** `JpaAgentGraphStore
@ApplicationScoped` displaces `NoOpAgentGraphStore @DefaultBean` automatically.
Deployments without the graph module see NoOp behaviour — no exceptions, no
recording.

---

## Schema Changes Required in Existing Runtime Module

The runtime module's V1 and V2 migrations need redesign. Since no deployed instances
exist, both files are rewritten directly (no additional migration needed).

### V1 redesign: agent_descriptor — surrogate key

Current V1 uses `agent_id VARCHAR PRIMARY KEY`. This must change to a BIGSERIAL
surrogate key with a UNIQUE constraint on `(agent_id, tenancy_id)`. The `agent_capability`
FK must reference the surrogate key.

```sql
-- V1__initial_schema.sql (runtime module — full rewrite)
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

`AgentDescriptorEntity` must change `@Id String agentId` to
`@Id @GeneratedValue Long internalId` with `agent_id` as a plain `@Column`. Update
`AgentCapabilityEntity` FK to reference `internalId`. Update `AgentDescriptorMapper`
accordingly. `agentId + tenancyId` remains the lookup key via the UNIQUE constraint.

### V2 redesign: agent_degradation_state → agent_state

Current V2 creates `agent_degradation_state(agent_id, degradation_reason, expires_at)`
with no `tenancy_id`. This is renamed and extended. The `AgentStateStore` SPI change
(below) makes tenancy_id mandatory.

```sql
-- V2__agent_degradation_state.sql (runtime module — full rewrite)
CREATE TABLE agent_state (
    agent_id     VARCHAR(255)  NOT NULL,
    tenancy_id   VARCHAR(255)  NOT NULL,
    degradation  VARCHAR(50)   NOT NULL,
    expires_at   TIMESTAMPTZ   NOT NULL,
    recorded_at  TIMESTAMPTZ   NOT NULL DEFAULT now(),
    PRIMARY KEY (agent_id, tenancy_id)
);
```

Update `JpaAgentStateStore` entity mapping, `DefaultReactiveAgentStateStore`,
and all JPA state store tests to use the new table name and schema.

---

## AgentStateStore SPI Extension (Breaking Change)

The current `AgentStateStore` SPI has no `tenancyId`. A `JpaAgentStateStore` backed
by `agent_state(agent_id, tenancy_id, ...)` cannot correctly scope queries without it.
agentId is not globally unique across tenancies.

**New SPI:**
```java
public interface AgentStateStore {
    void record(String agentId, String tenancyId, DegradationReason reason,
                Instant expiresAt);
    Optional<DegradationReason> query(String agentId, String tenancyId);
    void clear(String agentId, String tenancyId);
}
```

**All implementations must be updated:**
- `NoOpAgentStateStore` — add tenancyId params, ignore them
- `InMemoryAgentStateStore` — change map key from `String agentId` to
  composite `(agentId, tenancyId)` pair
- `JpaAgentStateStore` — scope queries by `(agent_id, tenancy_id)`
- `DefaultCapabilityHealth` — pass `descriptor.tenancyId()` to all `stateStore` calls
- All test callsites

No users exist — this is the right time for this change.

---

## New API Types (`casehub-eidos-api`)

### Records

```java
public record AgentTask(
    String taskId,          // eidos-assigned UUID
    String agentId,
    String tenancyId,
    String capabilityTag,
    String taskDomain,      // subject domain — "rust", "security", etc.
    String externalRef,     // opaque — PlanItem ID, WorkItem ID, or any ref; nullable
    Instant startedAt,
    Instant endedAt         // null if in progress — agentHistory() includes these
) {}

public record AgentTaskId(String taskId, String agentId, String tenancyId) {}

public record AgentOutcome(
    String taskId,
    TaskResult result,
    double confidence,                   // 0.0–1.0
    DegradationReason degradationReason  // null if not applicable
    // Note: no tenancyId — derive via join to AgentTask when needed
) {}

public record AttestationRef(
    String taskId,              // null for backfilled refs without a matched task
    String agentId,
    String tenancyId,
    String ledgerEntryHash,
    String entryType,           // "MessageLedgerEntry" etc. — opaque to eidos
    Instant attestedAt
) {}

public record AgentTaskHistory(
    String agentId,
    String tenancyId,
    List<AgentTask> tasks,           // includes in-progress (endedAt=null)
    List<AgentOutcome> outcomes,
    List<AttestationRef> attestationRefs,
    GraphDataSufficiency sufficiency
) {}

public record GraphDataSufficiency(
    int sampleCount,
    SufficiencyLevel level,
    Instant dataFrom,           // null if no data
    Instant dataThrough,        // null if no data
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
        return new GraphDataSufficiency(0, SufficiencyLevel.INSUFFICIENT,
                                        null, null, warnings);
    }
}

public record BackfillResult(int imported, int skipped,
                              Instant rangeFrom, Instant rangeThrough) {}
```

### Enums

```java
public enum TaskResult    { SUCCEEDED, PARTIALLY, FAILED }
public enum SufficiencyLevel { SUFFICIENT, INDICATIVE, INSUFFICIENT }
```

### SPIs (`api/spi/`)

```java
// Write path — casehub-engine calls this at dispatch and completion
public interface AgentGraphStore {
    void recordTask(AgentTask task);
    void recordOutcome(AgentTaskId id, AgentOutcome outcome);
    void linkAttestation(AgentTaskId id, AttestationRef ref);
}

// Read path — routing, audit, trust grounding
public interface AgentGraphQuery {
    // Returns all tasks including in-progress (endedAt=null).
    // Callers that want only completed tasks filter on endedAt != null.
    AgentTaskHistory agentHistory(String agentId, String tenancyId);

    AgentTaskHistory historyByCapability(String agentId, String capabilityTag,
                                         String tenancyId);

    // Returns agentIds ranked by Wilson lower bound score — see formula below.
    List<String> topAgentsByOutcome(String capabilityTag, String taskDomain,
                                    String tenancyId, int limit);

    List<AttestationRef> attestationsFor(String agentId, String tenancyId);
}

// Reactive mirror — build-gated on casehub.eidos.graph.reactive.enabled
public interface ReactiveAgentGraphQuery {
    Uni<AgentTaskHistory> agentHistory(String agentId, String tenancyId);
    Uni<List<String>> topAgentsByOutcome(String capabilityTag, String taskDomain,
                                          String tenancyId, int limit);
}

// Semantic extension — application-tier, pulled at query time
public interface TaskSemanticEnricher {
    Set<String> dispositionAxes(String capabilityTag, String taskDomain);
    boolean semanticallyEquivalent(String domainA, String domainB);
    OptionalInt significance(String capabilityTag, String taskDomain);
}

// Backfill — ingests historical ledger attestations
public interface AgentGraphBackfill {
    BackfillResult backfillAgent(String agentId, String tenancyId);
    BackfillResult backfillAll(String tenancyId);
    BackfillResult backfillDelta(String tenancyId, Instant since);
}
```

### topAgentsByOutcome — Ranking Formula

`topAgentsByOutcome` ranks agents by **Wilson lower bound** on confidence-weighted
fractional quality scores. This solves cold-start (small sample counts don't
outrank larger samples with slightly lower raw rates) while incorporating the
confidence quality signal. No recency weighting — acute degradation is handled by
`CapabilityHealth.probe()` and `AgentStateStore`. The graph is a long-term reputation
signal; those two mechanisms handle short-term health. Separating them prevents
conflict.

```
quality_score(outcome) = confidence × result_multiplier
    where result_multiplier: SUCCEEDED=1.0, PARTIALLY=0.5, FAILED=0.0

Per agent, n = total outcome count for (capabilityTag, taskDomain, tenancyId):
    p = SUM(quality_score) / n
    z = 1.645  [90% confidence interval]
    score = (p + z²/(2n) − z × sqrt((p(1−p) + z²/(4n)) / n)) / (1 + z²/n)
    score = 0.0 when n = 0
```

Agents with score = 0.0 (no observations) appear last. The trust maturity model
handles cold-start routing (Phase 0 availability routing); the graph only augments
when data exists.

This formula appears verbatim in `InMemoryAgentGraph`, `JpaAgentGraphQuery`, and is
validated by `v1_routingPrefersAgentWithBetterOutcomeHistory`.

When `TaskSemanticEnricher.semanticallyEquivalent()` returns true for two domains,
`JpaAgentGraphQuery` widens the `task_domain IN (:d1, :d2)` clause before computing
the score — the enricher's answer becomes a query parameter.

**Default confidence values when engine does not yet produce structured signal:**
SUCCEEDED → 1.0, PARTIALLY → 0.5, FAILED → 0.0.

---

## Runtime-Module Defaults (in `casehub-eidos`)

| Class | Type | Behaviour |
|-------|------|----------|
| `NoOpAgentGraphStore` | No-op | All methods silent no-ops |
| `NoOpAgentGraphQuery` | No-op | Returns empty `AgentTaskHistory` with `INSUFFICIENT`; never null |
| `NoOpAgentGraphBackfill` | No-op | Returns `BackfillResult(0, 0, null, null)` |
| `NoOpTaskSemanticEnricher` | No-op | Returns `Set.of()`, `false`, `OptionalInt.empty()` |
| `BlockingToReactiveGraphBridge` | Bridge | Wraps `AgentGraphQuery` as `ReactiveAgentGraphQuery` — see below |

**`BlockingToReactiveGraphBridge`** has no JPA or graph dependency — it belongs in
the runtime module (same module as the SPI), not the graph module. Per
PP-20260529-5745c1, it must be `@DefaultBean @ApplicationScoped` with no
`@IfBuildProperty` gate, and must call `.runSubscriptionOn()`:

```java
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
                  .item(() -> blocking.topAgentsByOutcome(capabilityTag, taskDomain,
                                                           tenancyId, limit))
                  .runSubscriptionOn(Infrastructure.getDefaultWorkerPool());
    }
}
```

---

## Graph Module Schema (V3 only)

```sql
-- V3__agent_graph.sql (casehub-eidos-graph — new graph tables only)

CREATE TABLE agent_task (
    task_id        VARCHAR(36)       PRIMARY KEY,
    agent_id       VARCHAR(255)      NOT NULL,
    tenancy_id     VARCHAR(255)      NOT NULL,
    capability_tag VARCHAR(255)      NOT NULL,
    task_domain    VARCHAR(255),
    external_ref   TEXT,                        -- nullable: in-process tasks may lack ref
    started_at     TIMESTAMPTZ       NOT NULL,
    ended_at       TIMESTAMPTZ
);
CREATE INDEX idx_task_agent          ON agent_task(agent_id, tenancy_id);
CREATE INDEX idx_task_cap_domain     ON agent_task(agent_id, capability_tag,
                                                   task_domain, tenancy_id);
CREATE INDEX idx_task_cap_tenant     ON agent_task(capability_tag, task_domain,
                                                   tenancy_id);  -- topAgentsByOutcome

CREATE TABLE agent_outcome (
    task_id            VARCHAR(36)       PRIMARY KEY
                                          REFERENCES agent_task(task_id),
    result             VARCHAR(20)       NOT NULL,
    confidence         DOUBLE PRECISION  NOT NULL
                                          CHECK (confidence BETWEEN 0 AND 1),
    degradation_reason VARCHAR(50),               -- nullable
    observed_at        TIMESTAMPTZ       NOT NULL
    -- no tenancy_id: derive via join to agent_task when needed
);

CREATE TABLE attestation_ref (
    ref_id             VARCHAR(36)   PRIMARY KEY,
    task_id            VARCHAR(36)   REFERENCES agent_task(task_id),  -- nullable for backfill
    agent_id           VARCHAR(255)  NOT NULL,
    tenancy_id         VARCHAR(255)  NOT NULL,
    ledger_entry_hash  VARCHAR(255)  NOT NULL,
    entry_type         VARCHAR(255)  NOT NULL,
    attested_at        TIMESTAMPTZ   NOT NULL,
    CONSTRAINT uq_attestation UNIQUE (ledger_entry_hash, tenancy_id)
);
CREATE INDEX idx_attest_agent ON attestation_ref(agent_id, tenancy_id);
```

The `UNIQUE (ledger_entry_hash, tenancy_id)` constraint makes both the write-time
auto-link and backfill idempotent — both paths use
`INSERT INTO attestation_ref ... ON CONFLICT (ledger_entry_hash, tenancy_id) DO NOTHING`.

---

## Integration

### casehub-engine write path

`WorkOrchestrator` already holds an optional compile dep on `casehub-eidos-api`.
Two call sites, both guarded by `worker.hasDescriptor()`:

**At dispatch:**
```java
graphStore.recordTask(new AgentTask(
    UUID.randomUUID().toString(),
    worker.agentDescriptor().agentId(),
    worker.agentDescriptor().tenancyId(),
    planItem.capabilityTag(),
    probeContext.taskDomain(),
    planItem.id().toString(),   // externalRef — opaque PlanItem ID; nullable if absent
    Instant.now(),
    null
));
```

**At completion:**
```java
graphStore.recordOutcome(taskId, new AgentOutcome(
    taskId.taskId(),
    outcome.succeeded() ? TaskResult.SUCCEEDED : TaskResult.FAILED,
    outcome.confidence(),       // engine signal; default 1.0/0.5/0.0 if unstructured
    outcome.degradationReason() // null if clean success
));
```

### casehub-ledger attestation linkage

After `recordOutcome()`, `JpaAgentGraphStore` queries ledger by `actorId` for recent
entries within the task time window. Uses
`INSERT INTO attestation_ref ... ON CONFLICT (ledger_entry_hash, tenancy_id) DO NOTHING`
— idempotent by design. Best-effort — if no ledger entry is found, the outcome node
exists without an attestation link. The graph never blocks on ledger availability.

### Backfill

`JpaAgentGraphBackfill` calls `TrustExportService.exportActor()` (casehub-ledger).
Backfilled `AttestationRef` nodes have `taskId=null`. Also uses
`INSERT ... ON CONFLICT DO NOTHING` — safe to run repeatedly.

`backfillDelta(tenancyId, since)` is the operational pattern: run `backfillAll` once
at deployment, then `backfillDelta` on a schedule.

---

## Coverage

| Use case | Without enricher | With `TaskSemanticEnricher` |
|----------|-----------------|----------------------------|
| Agent task history | ✅ Reliable | — |
| Routing by outcome stats | ✅ Reliable | ✅ Wider sample pool |
| Trust score lineage (hash chain) | ✅ Reliable | — |
| Attestation backfill | ✅ Reliable | — |
| Personality axis correlation | ❌ Not possible | ✅ Enabled |
| Semantic domain equivalence | ❌ Exact string match only | ✅ Enabled |
| Cross-domain outcome comparison | ⚠️ Exact strings only | ✅ Semantic grouping |

Engineers wanting personality axis correlation must provide a `TaskSemanticEnricher`
CDI bean. The system warns in `GraphDataSufficiency.warnings` when queries that could
benefit from an enricher are called without one.

---

## Testing Strategy

**Layer 1 — PoC (plain JUnit, `examples/agent-scenarios/`)**  
`V1GraphScenarioTest` and `V2GraphScenarioTest` against `InMemoryAgentGraph`. No CDI,
no H2. Must pass before Phase 4b begins.

**Layer 2 — NoOp contract tests (`casehub-eidos` runtime)**  
Verify NoOp impls satisfy contracts — return empty-but-valid results, never throw,
never null.

**Layer 3 — JPA integration tests (`@QuarkusTest`, H2 `MODE=PostgreSQL`,
`casehub-eidos-graph`)**

Required coverage:
- Write/read round-trip: task, outcome, attestation
- `topAgentsByOutcome` ordering and Wilson formula correctness
- Sufficiency thresholds: 4 → INSUFFICIENT, 5 → INDICATIVE, 10 → SUFFICIENT
- Backfill with `taskId=null` still queryable
- `backfillDelta(since)` excludes entries before cutoff
- Duplicate attestation insert is a no-op (ON CONFLICT DO NOTHING)
- Tenancy isolation — agent in tenancy A never visible in tenancy B query
- `agentHistory()` includes in-progress tasks (endedAt=null)

**Layer 4 — ArchUnit (`casehub-eidos-graph`)**  
Entity classes in `io.casehub.eidos.graph.entity` must not be referenced from
`io.casehub.eidos.api`.

---

## Issues to Create Before Implementation

**In casehub-eidos:**
- Add tenancyId to AgentStateStore SPI — extend SPI, update NoOp, InMemory, Jpa impls,
  DefaultCapabilityHealth, all test callsites (addressed in this phase)

**In casehub-engine:**
- WorkOrchestrator: add `AgentGraphStore` write-path call sites at dispatch and
  completion; pass `descriptor.tenancyId()` to graph store (requires A1 tenancyId)

**In casehub-parent:**
- Update Capability Ownership table: add rows for `AgentGraphStore`, `AgentGraphQuery`,
  `AgentGraphBackfill`, `TaskSemanticEnricher` (all owned by `casehub-eidos`)
- Update cross-repo dependency map: new row `casehub-eidos-api ← casehub-engine`
  (write-path SPI calls from WorkOrchestrator)
- Update casehub-eidos deep-dive: add `casehub-eidos-graph` module row, update
  current state
- Consider: file issue for `casehub-engine-ledger`'s `TrustWeightedAgentStrategy`
  to eventually consume `AgentGraphQuery.topAgentsByOutcome()` as higher-fidelity
  routing signal