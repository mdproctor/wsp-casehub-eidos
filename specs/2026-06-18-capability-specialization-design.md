# Capability Sub-Specialization Metadata (eidos#55)

**Date:** 2026-06-18  
**Issue:** casehubio/eidos#55  
**Parent epic:** casehubio/parent#258

---

## Problem

`AgentCapability` declares what an agent can do (`name`, `epistemicDomains` for positive confidence) but has no way to express what it cannot do. When agents DECLINE work repeatedly on a specific domain (e.g. Rust security reviews), that signal is lost — the engine keeps routing Rust reviews to an agent that will decline them.

Two population paths are needed:
1. **Declared** — at registration, admin states explicit exclusions (`"not: [rust, go]"`)
2. **Learned** — casehub-ledger/CBR accumulates DECLINE attestations and pushes the result into eidos

---

## Design

### 1. `AgentCapability` — add `excludedDomains: Set<String>`

New field alongside `epistemicDomains`:

```java
public record AgentCapability(
    String name,
    Double qualityHint,
    Long latencyHintP50Ms,
    String costHint,
    List<String> inputTypes,
    List<String> outputTypes,
    List<String> tags,
    Map<String, Double> epistemicDomains,   // declared positive confidence per domain
    Set<String> excludedDomains             // declared negative specialization
)
```

- Nullable — null means no declared exclusions.
- Validated: each domain string ≤ `MAX_CAPABILITY_STRING` (200 chars), same character-set rules as all other capability fields (no C0/C1, BiDi, zero-width).
- Defensively copied in compact constructor to unmodifiable set.
- Builder added (record now has 9 fields; positional construction is fragile at this arity). All existing call sites migrated to the builder as part of this issue.

**Relationship to `epistemicDomains`:** these are complementary, not overlapping. `epistemicDomains` expresses graded confidence (`{java: 0.95, kotlin: 0.80}`). `excludedDomains` expresses categorical exclusion (`{rust, go}`). A domain in `excludedDomains` takes priority over any `epistemicDomains` entry for the same domain.

### 2. `CapabilitySpecializationStore` SPI — new, in `casehub-eidos-api`

```java
public interface CapabilitySpecializationStore {
    /**
     * Records one DECLINE for the given agent, capability, and domain.
     * Called by casehub-ledger/CBR per qualifying DECLINE attestation.
     */
    void recordDecline(String agentId, String tenancyId, String capabilityName, String domain);

    /**
     * Retracts all learned data for a capability (e.g. agent re-trained or evidence invalidated).
     */
    void clearDeclines(String agentId, String tenancyId, String capabilityName);

    /**
     * Returns domain → decline count for all domains with at least 1 recorded decline.
     * Empty map when no declines recorded. Never null.
     */
    Map<String, Integer> learnedExclusions(String agentId, String tenancyId, String capabilityName);
}
```

**Implementations:**
- `NoOpCapabilitySpecializationStore` — `@DefaultBean @ApplicationScoped` in `runtime/health/`. Returns `Map.of()`, ignores writes.
- `InMemoryCapabilitySpecializationStore` — `@Alternative @Priority(1) @ApplicationScoped` in `persistence-memory/`. Compound key `agentId:tenancyId:capabilityName:domain` → `AtomicInteger`. Thread-safe via `ConcurrentHashMap.compute()`. `clearDeclines` removes all keys with matching `agentId:tenancyId:capabilityName` prefix.

**JPA-backed implementation** deferred to eidos#62 (separate module, separate Flyway migration).

### 3. `CapabilityStatus` — add `Excluded` variant

```java
sealed interface CapabilityStatus permits
        CapabilityStatus.Ready,
        CapabilityStatus.EpistemicallyWeak,
        CapabilityStatus.Excluded,
        CapabilityStatus.Degraded,
        CapabilityStatus.Unavailable {

    record Ready() implements CapabilityStatus {}
    record EpistemicallyWeak(String domain, double confidence) implements CapabilityStatus {}
    record Excluded(String domain, ExclusionSource source, int declineCount) implements CapabilityStatus {}
    record Degraded(DegradationReason reason, String detail) implements CapabilityStatus {}
    record Unavailable(String reason) implements CapabilityStatus {}

    enum ExclusionSource { DECLARED, LEARNED }
}
```

- `Excluded(domain, DECLARED, 0)` — for domains in `capability.excludedDomains()`.
- `Excluded(domain, LEARNED, N)` — for domains where `learnedExclusions` count ≥ threshold.
- `declineCount = 0` for `DECLARED` (count is meaningless; admin stated it directly).
- All existing `CapabilityStatus` switch sites update to handle `Excluded` — sealed interface guarantees compile-time exhaustiveness.

### 4. `DefaultCapabilityHealth` — updated probe logic

Inject `CapabilitySpecializationStore` alongside `AgentStateStore`. Inject `Preferences` for per-tenancy threshold resolution (replacing `@ConfigProperty` for the new exclude threshold).

**Probe priority order** (unchanged positions 1, 2, 5, 6; new positions 3, 4):

1. `AgentStateStore.query()` → `Degraded` if present
2. Capability not declared → `Unavailable`
3. `taskDomain != null` AND `taskDomain ∈ capability.excludedDomains()` → `Excluded(domain, DECLARED, 0)`
4. `taskDomain != null` AND `learnedExclusions.get(taskDomain) >= excludeThreshold` → `Excluded(domain, LEARNED, count)`
5. `taskDomain != null` AND `epistemicDomains.get(taskDomain) < weakThreshold` → `EpistemicallyWeak`
6. `Ready`

Steps 3 and 4 only run when `context.taskDomain()` is non-null. Step 3 short-circuits before the store lookup — declared exclusion takes priority and avoids an unnecessary map call.

**`Preferences`-backed threshold:**

```java
// casehub-eidos runtime
public final class EidosPreferenceKeys {
    public static final PreferenceKey<ExcludeThresholdPreference> EXCLUDE_THRESHOLD =
        new PreferenceKey<>("casehub.eidos.specialization.exclude-threshold",
                            ExcludeThresholdPreference.class);
}

public record ExcludeThresholdPreference(int value) implements Preference {
    public static final ExcludeThresholdPreference DEFAULT = new ExcludeThresholdPreference(3);
}
```

Resolved at probe time: `preferences.get(EXCLUDE_THRESHOLD, SettingsScope.of(Path.of(tenancyId), Instant.now())).orElse(DEFAULT).value()`.

**Default = 3** is provisional — 1 would be correct if DECLINE attestations are already filtered to capability-based declines before `recordDecline` is called; 3 provides noise tolerance when raw DECLINE events flow in. Revisit once casehub-ledger/CBR integration is live and real DECLINE data is available.

`weakThreshold` (existing) stays as `@ConfigProperty` — it's a deployment-wide confidence interpretation, not per-tenancy routing policy.

`DefaultReactiveCapabilityHealth` mirrors this update. `CapabilitySpecializationStore` is a blocking map lookup (no I/O), safe to call inline in the reactive path.

### 5. Persistence

**`V1__initial_schema.sql`** — add column to `agent_capability` (modified directly; no deployed instances):

```sql
excluded_domains   TEXT    -- JSON array: ["rust","go"] or null
```

**`AgentCapabilityEntity`:**

```java
@Column(name = "excluded_domains", columnDefinition = "TEXT") String excludedDomains;
```

**`AgentDescriptorMapper`:**

```java
// toCapability
readJson(c.excludedDomains, new TypeReference<Set<String>>() {})

// toCapabilityEntity
e.excludedDomains = writeJson(c.excludedDomains());
```

### 6. A2A_CARD Rendering

`excludedDomains` added to capability JSON in `A2A_CARD` format only (routing signal — following PP-20260611-228599). Omitted from MARKDOWN/PROSE. Omitted from the JSON object entirely when null or empty (not `"excludedDomains": null`).

```json
{
  "name": "security-review",
  "qualityHint": 0.9,
  "epistemicDomains": {"java": 0.95, "kotlin": 0.80},
  "excludedDomains": ["rust", "go"],
  "inputTypes": [...],
  "outputTypes": [...]
}
```

---

## Testing

### `AgentCapabilityTest`
- Builder round-trips all 9 fields correctly
- `excludedDomains` validation: blank domain rejected, null OK, over-length rejected, banned chars rejected
- Defensive copy: mutating input set after construction doesn't affect the record

### `DefaultCapabilityHealthTest`
- `taskDomain` in `excludedDomains` → `Excluded(domain, DECLARED, 0)`, store not consulted
- `learnedExclusions` count below threshold → continues to `Ready`
- `learnedExclusions` count at threshold → `Excluded(domain, LEARNED, count)`
- Threshold resolved from `Preferences` mock returning per-tenancy value
- `null` `taskDomain` → neither new check runs; existing behaviour preserved
- `Degraded` from `AgentStateStore` still takes priority over declared exclusion
- Declared exclusion takes priority over learned exclusion

### `InMemoryCapabilitySpecializationStoreTest`
- `recordDecline` accumulates count per domain correctly
- Concurrent `recordDecline` is safe (AtomicInteger)
- `clearDeclines` removes all entries for the capability, not others with shared prefix
- `learnedExclusions` returns only entries for the requested triple, not other agents or tenancies
- Returns empty map when no declines recorded

### `DefaultCapabilityHealthDegradedTest` (existing — extend)
- Add: `Excluded` status paths confirmed; `Degraded` wins over both

### Mapper / JPA round-trip
- `excludedDomains` serialises and deserialises correctly
- Null `excludedDomains` round-trips as null (not empty set)

### A2A_CARD renderer
- `excludedDomains` present in capability JSON when populated
- `excludedDomains` absent from JSON when null or empty
- `excludedDomains` absent from MARKDOWN and PROSE renders

---

## Deferred

| Issue | What |
|-------|------|
| eidos#61 | `AgentQuery.taskDomain` — pre-filter at registry level for efficiency |
| eidos#62 | JPA-backed `CapabilitySpecializationStore` for production persistence |
| parent#281 | PLATFORM.md capability ownership + cross-repo dependency map update |

---

## Not in scope

- DECLINE attestation structure in casehub-ledger (parent#258 scope)
- casehub-engine `AgentRoutingStrategy` changes (parent#258 scope)
- CBR aggregation logic in casehub-ledger (parent#258 scope)
- `AgentQuery` domain filtering (eidos#61)
