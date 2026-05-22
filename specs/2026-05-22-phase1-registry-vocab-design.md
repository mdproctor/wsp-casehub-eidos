# Phase 1 Design — Agent Registry, Vocabulary Registry, and Well-Known Vocabularies

**Branch:** `issue-1-phase-1-registry-vocab`
**Issue:** casehubio/eidos#1
**Date:** 2026-05-22

---

## Summary

Phase 1 implements all concrete backing for the SPIs defined in `casehub-eidos-api`. After this phase the extension is usable: agents can register descriptors, consumers can query by slot and capability, vocabulary terms are discoverable and cross-referenceable, and the well-known SVO / Conscientiousness / CasehubSlot vocabularies are available as optional CDI beans.

---

## API Amendments (`casehub-eidos-api`)

### AgentDescriptor — add `tenancyId`

Tenancy filtering must be unconditional and present from day one per `no-conditional-tenancy-filtering` protocol. `tenancyId` is added as the last field:

```java
public record AgentDescriptor(
    String agentId,
    String name,
    String version,
    String provider,
    String modelFamily,
    String modelVersion,
    String weightsFingerprint,
    String domainVocabulary,
    String slotVocabulary,
    String dispositionVocabulary,
    String slot,
    List<AgentCapability> capabilities,
    AgentDisposition disposition,
    String jurisdiction,
    String dataHandlingPolicy,
    String tenancyId          // always required — never null
) {}
```

Phase 1 callers use `"default"` (matching `TenancyConstants.DEFAULT_TENANT_ID` in `casehub-platform-api`). The value is stored and always applied in queries — no conditionals.

### AgentQuery (new record)

Replaces the three `findBySlot / findByCapability / findBySlotAndCapability` methods with a single `find(AgentQuery)`. Satisfies auth-retrofit-readiness Rule 4 (filter hook) and makes future dimensions (visibility, pagination) additive.

```java
public record AgentQuery(
    String slot,           // null = any slot
    String capabilityName, // null = any capability
    String tenancyId       // required — never null
) {
    public AgentQuery {
        Objects.requireNonNull(tenancyId, "tenancyId must not be null");
    }

    public static AgentQuery bySlot(String slot, String tenancyId) {
        return new AgentQuery(slot, null, tenancyId);
    }
    public static AgentQuery byCapability(String capabilityName, String tenancyId) {
        return new AgentQuery(null, capabilityName, tenancyId);
    }
    public static AgentQuery bySlotAndCapability(String slot, String capabilityName, String tenancyId) {
        return new AgentQuery(slot, capabilityName, tenancyId);
    }
    public static AgentQuery all(String tenancyId) {
        return new AgentQuery(null, null, tenancyId);
    }
}
```

### AgentRegistry — replace query methods

```java
public interface AgentRegistry {
    void register(AgentDescriptor descriptor);
    Optional<AgentDescriptor> findById(String agentId);
    List<AgentDescriptor> find(AgentQuery query);
}
```

### ReactiveAgentRegistry — parity + replace query methods

```java
public interface ReactiveAgentRegistry {
    Uni<Void> register(AgentDescriptor descriptor);
    Uni<Optional<AgentDescriptor>> findById(String agentId);
    Uni<List<AgentDescriptor>> find(AgentQuery query);
}
```

Parity with blocking API is structural and enforced by naming convention (ArchUnit `BlockingReactiveParityTest` pattern from casehub-ledger — to be added in Phase 1 runtime tests).

---

## Module Structure and Pom Changes

No new modules. Three pom updates:

| Module | Change |
|--------|--------|
| `runtime/` | Add `quarkus-hibernate-reactive-panache` |
| `persistence-memory/` | Add Jandex plugin (required for CDI bean discovery when consumed as JAR) |
| `vocab/` | Add Jandex plugin (same reason) |

`deployment/` gets a new `EidosBuildTimeConfig` class — no pom change needed.

---

## JPA Schema (`V1__initial_schema.sql`)

Migration path: `runtime/src/main/resources/db/eidos/migration/V1__initial_schema.sql`

The extension does **not** set `quarkus.flyway.locations` — per `quarkus-extension-flyway-locations-explicit` protocol. Consumers must add `classpath:db/eidos/migration` to their Flyway locations.

```sql
CREATE TABLE agent_descriptor (
    agent_id               VARCHAR(255)  NOT NULL PRIMARY KEY,
    tenancy_id             VARCHAR(255)  NOT NULL,
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
    disposition            TEXT          -- JSON: AgentDisposition
);

CREATE TABLE agent_capability (
    id                  BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    agent_id            VARCHAR(255)  NOT NULL
                            REFERENCES agent_descriptor(agent_id) ON DELETE CASCADE,
    name                VARCHAR(255)  NOT NULL,
    quality_hint        DOUBLE PRECISION,
    latency_hint_p50_ms BIGINT,
    cost_hint           VARCHAR(255),
    input_types         TEXT,    -- JSON: List<String>
    output_types        TEXT,    -- JSON: List<String>
    tags                TEXT,    -- JSON: List<String>
    epistemic_domains   TEXT     -- JSON: Map<String, Double>
);

CREATE INDEX idx_agent_descriptor_tenancy_slot ON agent_descriptor(tenancy_id, slot);
CREATE INDEX idx_agent_capability_name         ON agent_capability(name);
```

Notes:
- `BIGINT GENERATED ALWAYS AS IDENTITY` — ANSI SQL:2003, works in H2 `MODE=PostgreSQL` and PostgreSQL
- `DOUBLE PRECISION` — per `sql-type-portability` protocol
- `TEXT` for JSON fields — portable; Phase 2 can `ALTER COLUMN TYPE JSONB` for PostgreSQL capability indexing
- `ON DELETE CASCADE` — unregistering an agent removes all capabilities atomically
- Composite index on `(tenancy_id, slot)` covers the most common query pattern

---

## JPA Entities and Mapper

All package-private (`io.casehub.eidos.runtime.registry.jpa`).

**`AgentDescriptorEntity`** — `@Entity @Table("agent_descriptor")`. `@Id agentId`. `@Column(nullable=false) tenancyId`. `@OneToMany(mappedBy="descriptor", cascade=ALL, fetch=LAZY, orphanRemoval=true) List<AgentCapabilityEntity> capabilities`. `@Column(columnDefinition="TEXT") disposition` (JSON).

**`AgentCapabilityEntity`** — `@Entity @Table("agent_capability")`. `@Id @GeneratedValue(IDENTITY) Long id`. `@ManyToOne(fetch=LAZY) @JoinColumn("agent_id") AgentDescriptorEntity descriptor`. `name`, `qualityHint`, `latencyHintP50Ms`, `costHint` as typed columns. `inputTypes`, `outputTypes`, `tags`, `epistemicDomains` as `@Column(columnDefinition="TEXT")` JSON.

All queries use `JOIN FETCH a.capabilities` — never relying on EAGER loading, which behaves differently between blocking ORM and Hibernate Reactive.

**`AgentDescriptorMapper`** — injected `ObjectMapper` handles JSON fields. `toEntity(AgentDescriptor)` and `toRecord(AgentDescriptorEntity)`. Upsert strategy: delete existing by `agentId` then persist new (simpler than merge with orphanRemoval on a collection).

---

## Registries

### JpaAgentRegistry (blocking)

`@ApplicationScoped` in `runtime/`. Uses `@Inject EntityManager`. Transactional boundary on `register()` via `@Transactional`.

Query for `find(AgentQuery)`:
```jpql
SELECT DISTINCT a FROM AgentDescriptorEntity a
JOIN FETCH a.capabilities c
WHERE a.tenancyId = :tenancyId
  [AND a.slot = :slot]
  [AND c.name = :capabilityName]
```

Predicates added conditionally from `AgentQuery` fields. `tenancyId` predicate always applied — no conditionals skipping it.

### JpaReactiveAgentRegistry (reactive, build-gated)

`@IfBuildProperty(name="casehub.eidos.reactive.enabled", stringValue="true") @ApplicationScoped`.

Uses `@Inject AgentDescriptorReactivePanacheRepo` (package-private, also `@IfBuildProperty`-gated). Write methods annotated `@WithTransaction`. Read queries use JPQL with `JOIN FETCH`. Same tenancy guarantee as blocking tier.

### EidosBuildTimeConfig

```java
@ConfigMapping(prefix = "casehub.eidos")
@ConfigRoot(phase = ConfigPhase.BUILD_TIME)
public interface EidosBuildTimeConfig {
    ReactiveConfig reactive();
    interface ReactiveConfig {
        @WithDefault("false")
        boolean enabled();
    }
}
```

Placed in `deployment/`. Mirrors the qhorus pattern exactly.

### InMemoryAgentRegistry

`@Alternative @Priority(1) @ApplicationScoped` in `persistence-memory/`. `ConcurrentHashMap<String, AgentDescriptor>`. Implements `AgentRegistry`. All `find()` queries apply `tenancyId` filter unconditionally via stream predicate — no conditionals.

### InMemoryReactiveAgentRegistry

`@Alternative @Priority(1) @ApplicationScoped` in `persistence-memory/`. Implements `ReactiveAgentRegistry`. Delegates to `InMemoryAgentRegistry`, wraps in `Uni.createFrom().item(...)`. In-memory operations are instantaneous — no `runSubscriptionOn` needed.

When `persistence-memory` is on the classpath, both implementations win over the JPA tier via the CDI priority ladder.

---

## VocabularyRegistry

### CdiVocabularyRegistry

`@DefaultBean @ApplicationScoped` in `runtime/`.

Discovers all CDI `Vocabulary` beans via `@Inject @Any Instance<Vocabulary>` at `@PostConstruct`. Also holds a `ConcurrentHashMap<String, Vocabulary>` for programmatic `register()` calls at runtime. CDI-discovered beans populate the map first; programmatic calls can override by URI.

`find(uri)` — map lookup.

`resolve(vocabularyUri, value)` — finds the first term in the vocabulary where `term.value().equals(value)` or `term.aliases().contains(value)`.

`equivalentValues(vocabularyUri, value, targetVocabularyUri)` — finds all terms matching value/alias in the source vocabulary, collects their `exactMatches.get(targetVocabularyUri)` values into a `Set<String>`.

No persistence — vocabularies are code artifacts. No separate in-memory or JPA implementation needed.

---

## Defaults

### NoOpCapabilityHealth

`@DefaultBean @ApplicationScoped` in `runtime/`. Always returns `new CapabilityStatus.Ready()`. Phase 2 replaces this with real probe logic against declared capabilities and runtime state.

---

## casehub-eidos-vocab

Three `@ApplicationScoped` producer classes in `vocab/`. Each has a `@Produces @ApplicationScoped Vocabulary` method. `@Any Instance<Vocabulary>` in `CdiVocabularyRegistry` collects all — no qualifiers needed since no code injects `Vocabulary` as a singular unqualified bean.

**SvoVocabularyProducer** — URI `urn:casehub:vocab:svo`. Terms:

| Value | Aliases | exactMatches → casehub-slot |
|-------|---------|-----|
| `performer` | actor, executor | `executor` |
| `evaluator` | reviewer, judge | `reviewer` |
| `coordinator` | planner, orchestrator | `planner` |

**ConscientiousnessVocabularyProducer** — URI `urn:casehub:vocab:conscientiousness`. Covers all four `AgentDisposition` axes:

- `ruleFollowing`: `strict`, `principled`, `flexible`
- `riskAppetite`: `conservative`, `measured`, `bold`
- `socialOrient`: `collaborative`, `independent`, `facilitative`
- `autonomy`: `directed`, `semi-autonomous`, `autonomous`

**CasehubSlotVocabularyProducer** — URI `urn:casehub:vocab:casehub-slot`. Terms:

| Value | Aliases | exactMatches → svo |
|-------|---------|-----|
| `planner` | orchestrator | `coordinator` |
| `reviewer` | evaluator, judge | `evaluator` |
| `executor` | performer | `performer` |
| `supervisor` | overseer | — |

Cross-vocabulary `exactMatches` are bidirectional — SVO ↔ CasehubSlot resolution works in both directions.

---

## Testing Strategy

### api/ — Plain JUnit 5

- `AgentQueryTest` — constructor validation: null tenancyId throws, static factories produce correct fields
- `AgentDescriptorTest` — record construction, all fields accessible
- SPI contract tests: anonymous implementation of each SPI compiles with no default methods (confirms contracts are abstract)

### runtime/ — @QuarkusTest, H2 MODE=PostgreSQL

`application.properties`:
```properties
quarkus.datasource.db-kind=h2
quarkus.datasource.jdbc.url=jdbc:h2:mem:eidostest;MODE=PostgreSQL;DB_CLOSE_DELAY=-1
quarkus.flyway.locations=classpath:db/eidos/migration
```

- `JpaAgentRegistryTest` — register, findById, find by slot, find by capability, find by slot+capability, upsert replaces existing, tenancy isolation (agent in tenancy "a" not returned for tenancy "b")
- `JpaReactiveAgentRegistryTest` — same operations via reactive API (test profile: `casehub.eidos.reactive.enabled=true`)
- `CdiVocabularyRegistryTest` — CDI bean discovery (with vocab on classpath), programmatic register, find, resolve, equivalentValues
- `NoOpCapabilityHealthTest` — probe returns Ready for any input
- `BlockingReactiveParityTest` (ArchUnit) — confirms `JpaReactiveAgentRegistry` has a `Uni<T>` equivalent for every method on `JpaAgentRegistry`

### persistence-memory/ — @QuarkusTest, no datasource

- `InMemoryAgentRegistryTest` — all five operations including tenancy isolation
- `InMemoryReactiveAgentRegistryTest` — same via reactive API

### vocab/ — Plain JUnit 5

- `SvoVocabularyTest` — all terms present, aliases correct, exactMatches reference valid casehub-slot URIs
- `ConscientiousnessVocabularyTest` — axes and values correct
- `CasehubSlotVocabularyTest` — all terms present, exactMatches reference valid SVO URIs; cross-reference consistency (if SVO.performer → executor, then casehub-slot.executor → performer)

---

## Platform Coherence Final Check

| Check | Status |
|-------|--------|
| Already exists elsewhere? | No — new capability |
| Correct repo? | Yes — eidos is the agent identity layer |
| Consolidation opportunity? | None |
| Module tier structure | ✅ api=Tier1, runtime=Tier3, persistence-memory=Tier3 alternative |
| CDI priority ladder | ✅ @DefaultBean → @ApplicationScoped → @Alternative @Priority(1) |
| Reactive build gating | ✅ @IfBuildProperty + EidosBuildTimeConfig @ConfigRoot(BUILD_TIME) |
| Flyway | ✅ V1, scoped path db/eidos/migration, extension does not set locations |
| Module naming | ✅ api, runtime, persistence-memory, deployment, vocab |
| Jandex | ✅ persistence-memory and vocab |
| No platform-api additions | ✅ AgentDescriptor et al. stay in casehub-eidos-api |
| Auth retrofit readiness | ✅ find(AgentQuery), tenancyId unconditional |
| No-conditional-tenancy-filtering | ✅ tenancyId always applied in queries and stream filters |
| SPI signatures free of auth types | ✅ |
| PLATFORM.md update needed | After implementation — capability ownership table |
