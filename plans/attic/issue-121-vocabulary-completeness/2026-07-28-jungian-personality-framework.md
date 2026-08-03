# Jungian Personality Framework — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> executing-plans to implement this plan task-by-task. Each task follows
> TDD (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #107 — epic: Jungian personality framework
**Issue group:** #108, #109, #110, #111, #112, #113, #114, #115, #116, #117

**Goal:** Integrate Jungian cognitive functions into eidos as a vocabulary-grounded,
weighted disposition system with personality health probing and evolution.

**Architecture:** Three layers — vocabulary (JungianFunctionTerm, MbtiTypeTerm,
cross-vocab equivalences in `casehub-eidos-vocab`), API evolution (DispositionValue,
weighted AgentDisposition, DispositionSignalStore, DispositionHealth, DispositionEvolution
SPIs in `casehub-eidos-api`), and runtime (auto-derivation, rendering, probe/evolution
implementations in `casehub-eidos` runtime).

**Tech Stack:** Java 21 (on Java 26 JVM), Quarkus 3.32.2, LangChain4j (ChatModel for
LLM-adjudicated evolution), JPA/Flyway (schema), JUnit 5 + @QuarkusTest

## Global Constraints

- **Spec:** `/Users/mdproctor/claude/casehub/worktrees/43/work/eidos/specs/issue-107-jungian-personality-framework/2026-07-28-jungian-personality-framework-design.md`
- **Pre-release:** no backward compatibility concern — fix the design, break callers
- **IntelliJ MCP mandatory** for all .java edits — `ide_edit_member`, `ide_insert_member`, `ide_replace_member`, `ide_create_file`, `ide_refactor_rename`
- **Build command:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- **Test command (single module):** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl <module>`
- **Project path for IntelliJ:** `/Users/mdproctor/claude/casehub/worktrees/43/eidos`
- **NaN anti-pattern:** always guard `Double.isNaN(weight)` before range checks (ARC42STORIES §8 Anti-pattern 4)
- **CDI displacement ladder:** every new SPI needs `@DefaultBean` NoOp in runtime, `@Alternative @Priority(1)` in-memory in persistence-memory
- **Weight normalization:** profile weights must sum to 1.0 at probe time; per-axis weights normalized during auto-derivation
- **All commits reference issues:** `Refs #N` (ongoing) or `Closes #N` (done)

---

### Task 1: DispositionValue record + AgentDisposition API evolution (#111, #112)

The foundational API change. Everything else depends on this.

**Files:**
- Create: `api/src/main/java/io/casehub/eidos/api/DispositionValue.java`
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentDisposition.java`
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentDescriptor.java` (Builder convenience methods)
- Modify: `api/src/main/java/io/casehub/eidos/api/BehavioralExpectations.java`
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentDescriptorComparator.java`
- Test: `api/src/test/java/io/casehub/eidos/api/DispositionValueTest.java`
- Test: `api/src/test/java/io/casehub/eidos/api/AgentDispositionTest.java` (new)
- Modify: all existing API tests that construct AgentDisposition

**Interfaces:**
- Produces: `DispositionValue(String term, double weight)` — used by every subsequent task
- Produces: `AgentDisposition` with `List<DispositionValue>` axes + `List<DispositionValue> dispositionProfile` + `boolean delegation`
- Produces: `AgentDisposition.get(DispositionAxis)` returns `List<DispositionValue>` (was `Optional<String>`)
- Produces: `AgentDisposition.Builder` with `socialOrient(String)` convenience (wraps to single-entry list with weight 1.0)
- Produces: `AgentDisposition.Builder.dispositionProfile(DispositionValue...)` varargs builder

- [ ] **Step 1: Write DispositionValue tests**

```java
// api/src/test/java/io/casehub/eidos/api/DispositionValueTest.java
package io.casehub.eidos.api;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

class DispositionValueTest {

    @Test void valid_construction() {
        var dv = new DispositionValue("independent", 0.7);
        assertThat(dv.term()).isEqualTo("independent");
        assertThat(dv.weight()).isEqualTo(0.7);
    }

    @Test void default_weight_is_one() {
        var dv = DispositionValue.of("independent");
        assertThat(dv.weight()).isEqualTo(1.0);
    }

    @Test void null_term_throws() {
        assertThatThrownBy(() -> new DispositionValue(null, 0.5))
            .isInstanceOf(IllegalArgumentException.class);
    }

    @Test void blank_term_throws() {
        assertThatThrownBy(() -> new DispositionValue("  ", 0.5))
            .isInstanceOf(IllegalArgumentException.class);
    }

    @Test void negative_weight_throws() {
        assertThatThrownBy(() -> new DispositionValue("x", -0.1))
            .isInstanceOf(IllegalArgumentException.class);
    }

    @Test void weight_above_one_throws() {
        assertThatThrownBy(() -> new DispositionValue("x", 1.1))
            .isInstanceOf(IllegalArgumentException.class);
    }

    @Test void nan_weight_throws() {
        assertThatThrownBy(() -> new DispositionValue("x", Double.NaN))
            .isInstanceOf(IllegalArgumentException.class);
    }

    @Test void zero_weight_is_valid() {
        var dv = new DispositionValue("x", 0.0);
        assertThat(dv.weight()).isEqualTo(0.0);
    }

    @Test void weight_one_is_valid() {
        var dv = new DispositionValue("x", 1.0);
        assertThat(dv.weight()).isEqualTo(1.0);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=DispositionValueTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — class does not exist

- [ ] **Step 3: Create DispositionValue record**

Use `ide_create_file`:

```java
// api/src/main/java/io/casehub/eidos/api/DispositionValue.java
package io.casehub.eidos.api;

public record DispositionValue(String term, double weight) {
    public DispositionValue {
        if (term == null || term.isBlank())
            throw new IllegalArgumentException("term required");
        if (Double.isNaN(weight) || weight < 0.0 || weight > 1.0)
            throw new IllegalArgumentException("weight must be 0.0–1.0, got " + weight);
    }

    public static DispositionValue of(String term) {
        return new DispositionValue(term, 1.0);
    }
}
```

- [ ] **Step 4: Run DispositionValue tests — verify pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=DispositionValueTest`
Expected: ALL PASS

- [ ] **Step 5: Evolve AgentDisposition record**

Use `ide_edit_member` with `member=AgentDisposition` to replace the record declaration.

Change from:
```java
public record AgentDisposition(
    String socialOrient,
    String ruleFollowing,
    String riskAppetite,
    String autonomy,
    String conflictMode,
    boolean delegation
)
```

To:
```java
public record AgentDisposition(
    List<DispositionValue> socialOrient,
    List<DispositionValue> ruleFollowing,
    List<DispositionValue> riskAppetite,
    List<DispositionValue> autonomy,
    List<DispositionValue> conflictMode,
    boolean delegation,
    List<DispositionValue> dispositionProfile
)
```

Update the compact constructor to validate lists (null → empty, defensive copy).
Update `get(DispositionAxis)` to return `List<DispositionValue>` instead of `Optional<String>`.
Update the Builder: each axis setter accepts both `String` (convenience, wraps to single weighted value) and `List<DispositionValue>`. Add `dispositionProfile(DispositionValue... values)` varargs builder.

**Key pattern for Builder backward compat:**
```java
public Builder socialOrient(String v) {
    this.socialOrient = v == null ? List.of() : List.of(DispositionValue.of(v));
    return this;
}
public Builder socialOrient(List<DispositionValue> v) {
    this.socialOrient = v == null ? List.of() : List.copyOf(v);
    return this;
}
public Builder socialOrient(DispositionValue... values) {
    this.socialOrient = List.of(values);
    return this;
}
```

- [ ] **Step 6: Update AgentDescriptor.Builder.disposition() and related**

The builder already accepts `AgentDisposition` — no change needed. But verify the descriptor's compact constructor handles the new list fields (null-safe).

- [ ] **Step 7: Update BehavioralExpectations**

`escalationExpected()` currently calls `disposition.autonomy()` which returns `String`. Now it returns `List<DispositionValue>`. Update to extract the primary term (first entry, or highest weighted):

```java
// In BehavioralExpectations.escalationExpected():
// Was: String autonomyValue = disposition.autonomy();
// Now:
String autonomyValue = disposition.autonomy().isEmpty() ? null
    : disposition.autonomy().get(0).term();
```

- [ ] **Step 8: Update AgentDescriptorComparator**

`compareDisposition()` compares String axis values. Now it compares `List<DispositionValue>`. Update the comparison logic to compare the full weighted lists.

- [ ] **Step 9: Fix all API-tier tests**

Update every test that calls `AgentDisposition.builder().socialOrient("independent")` etc. The String convenience overload means most builder calls compile unchanged. Tests that read axis values via `d.socialOrient()` now get `List<DispositionValue>` instead of `String` — update assertions accordingly.

Key files:
- `AgentDescriptorTest.java` — ~10 builder calls
- `AgentDescriptorComparatorTest.java` — ~8 builder calls + comparison assertions
- `BehavioralExpectationsTest.java` — ~10 builder calls + autonomy accessor
- `CapabilityResolverTest.java` — StubVocabularyRegistry

- [ ] **Step 10: Verify API module compiles and tests pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api`
Expected: ALL PASS

- [ ] **Step 11: Commit**

```
git add api/
git commit -m "feat(#111): DispositionValue record + weighted AgentDisposition

Evolves AgentDisposition from String-per-axis to List<DispositionValue>.
Adds dispositionProfile field for holistic disposition vocabularies
(Jungian functions, weighted DISC). Builder preserves String convenience
methods for backward compatibility.

Refs #111, Refs #112"
```

---

### Task 2: Schema migration + JPA entity updates

**Files:**
- Create: `runtime/src/main/resources/db/migration/V9__disposition_weighted_profile.sql`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentDescriptorEntity.java`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentDescriptorMapper.java`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/registrar/ClasspathYamlDescriptorRegistrar.java`
- Modify: all runtime tests constructing AgentDisposition

**Interfaces:**
- Consumes: `DispositionValue`, evolved `AgentDisposition` from Task 1
- Produces: JPA persistence for weighted disposition values and disposition profile

- [ ] **Step 1: Write schema migration**

```sql
-- V9__disposition_weighted_profile.sql

CREATE TABLE disposition_value (
    id          BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    descriptor_id BIGINT NOT NULL REFERENCES agent_descriptor(id) ON DELETE CASCADE,
    entry_type  VARCHAR(10) NOT NULL,
    axis        VARCHAR(30) NOT NULL,
    term        VARCHAR(100) NOT NULL,
    weight      DOUBLE PRECISION NOT NULL,
    ordinal     INT NOT NULL DEFAULT 0,
    CONSTRAINT uq_disposition_value UNIQUE (descriptor_id, axis, term),
    CONSTRAINT ck_entry_type CHECK (entry_type IN ('AXIS', 'PROFILE'))
);

CREATE TABLE disposition_signal (
    id            BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    agent_id      VARCHAR(255) NOT NULL,
    tenancy_id    VARCHAR(255) NOT NULL,
    function_term VARCHAR(30) NOT NULL,
    count         INT NOT NULL DEFAULT 0,
    CONSTRAINT uq_disposition_signal UNIQUE (agent_id, tenancy_id, function_term)
);

-- Remove old single-string disposition columns from agent_descriptor
ALTER TABLE agent_descriptor DROP COLUMN IF EXISTS social_orient;
ALTER TABLE agent_descriptor DROP COLUMN IF EXISTS rule_following;
ALTER TABLE agent_descriptor DROP COLUMN IF EXISTS risk_appetite;
ALTER TABLE agent_descriptor DROP COLUMN IF EXISTS autonomy;
ALTER TABLE agent_descriptor DROP COLUMN IF EXISTS conflict_mode;
-- delegation stays as boolean on agent_descriptor
```

- [ ] **Step 2: Update AgentDescriptorEntity**

Replace single-String disposition fields with a `@OneToMany` relationship to `DispositionValueEntity`. Or use JSON column for simplicity (existing pattern for `disposition` is JSON column via `AgentDescriptorMapper.readJson()`).

Check current pattern: `AgentDescriptorMapper.toRecord()` reads `e.disposition` as JSON into `AgentDisposition.class`. The simplest migration: keep the JSON column but now it contains the evolved structure with `List<DispositionValue>` fields.

If the current approach is JSON column, the schema migration just needs to handle the JSON format change, not create new tables. Read the current entity to confirm.

- [ ] **Step 3: Update AgentDescriptorMapper**

Update `toRecord()` and `toEntity()` to handle the evolved `AgentDisposition` JSON structure. Jackson will handle `List<DispositionValue>` serialization/deserialization automatically if the record has the right shape.

- [ ] **Step 4: Update ClasspathYamlDescriptorRegistrar**

Currently constructs `new AgentDisposition(socialOrient, ruleFollowing, ...)` with String values. Update to wrap in `List.of(DispositionValue.of(value))` for each axis.

- [ ] **Step 5: Fix runtime tests — verify pass**

Update all runtime tests that construct AgentDisposition. Key files:
- `EidosSystemPromptRendererTest.java`
- `EidosRenderPipelineTest.java`
- `JpaAgentRegistryTest.java`
- `ClasspathYamlDescriptorRegistrarTest.java`
- `DefaultCapabilityHealthTest.java` (and variants)

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: ALL PASS

- [ ] **Step 6: Fix persistence-memory tests**

Update `InMemoryAgentRegistryTest.java` and `TestVocabularyRegistry.java`.

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory`
Expected: ALL PASS

- [ ] **Step 7: Fix eval and example tests**

Update all eval module tests and example tests. Key files:
- `eval/`: `AgentProfileLoaderTest.java`, `ProximityJudgeTest.java`, `EvalDataset.java`, `AgentProfileLoader.java`, `ProximityJudge.java`, `TraitExpressionJudge.java`, `VocabularyExpressivenessJudge.java`, `PersonalityPreservationReport.java`
- `examples/agent-scenarios/`: `MultiAgentTeamTest.java`, `SystemPromptRendererTest.java`, `TenancyIsolationTest.java`, `EpistemicDomainMatchingTest.java`, `DegradationAndRecoveryTest.java`, `CapabilitySubsumptionScenarioTest.java`, `LearnedExclusionSubsumptionTest.java`
- `vocab/`: `ImpliesSupervisionTest.java`

- [ ] **Step 8: Full build — verify all modules pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS

- [ ] **Step 9: Commit**

```
git commit -m "feat(#111): schema migration + JPA/runtime/eval ripple for weighted disposition

V9 migration adds disposition_value and disposition_signal tables.
Updates all callers across runtime, eval, examples, and vocab modules
to use the evolved AgentDisposition with List<DispositionValue> axes.

Refs #111"
```

---

### Task 3: JungianFunctionTerm vocabulary (#108, #109)

**Files:**
- Create: `vocab/src/main/java/io/casehub/eidos/vocab/FunctionCategory.java`
- Create: `vocab/src/main/java/io/casehub/eidos/vocab/FunctionAttitude.java`
- Create: `vocab/src/main/java/io/casehub/eidos/vocab/JungianFunctionTerm.java`
- Create: `vocab/src/main/java/io/casehub/eidos/vocab/JungianVocabRegistrar.java`
- Test: `vocab/src/test/java/io/casehub/eidos/vocab/JungianFunctionTermTest.java`

**Interfaces:**
- Consumes: `VocabularyTerm`, `VocabularyMetadata`, `DispositionAxis`, `ConscientiousnessTerm`, `ThomasKilmannTerm`
- Produces: 8 JungianFunctionTerm constants with `axisExactMatch`, `shadow()`, `category()`, `attitude()`, `compatibleAuxiliaries()`

- [ ] **Step 1: Write JungianFunctionTerm tests**

Test axisExactMatch for all 8 functions × 5 axes = 40 mappings (table from spec §1.1).
Test shadow() for all 4 pairs.
Test compatibleAuxiliaries() for at least 2 functions.
Test weight tier constants exist.

```java
@Test void ti_maps_to_independent_on_social_orientation() {
    assertThat(JungianFunctionTerm.TI.axisExactMatch(
        ConscientiousnessTerm.class, DispositionAxis.SOCIAL_ORIENTATION))
        .contains(ConscientiousnessTerm.INDEPENDENT);
}

@Test void ti_shadow_is_te() {
    assertThat(JungianFunctionTerm.TI.shadow()).isEqualTo(JungianFunctionTerm.TE);
}

@Test void ti_compatible_auxiliaries_are_perceiving_functions() {
    assertThat(JungianFunctionTerm.TI.compatibleAuxiliaries())
        .containsExactlyInAnyOrder(
            JungianFunctionTerm.SE, JungianFunctionTerm.SI,
            JungianFunctionTerm.NE, JungianFunctionTerm.NI);
}
```

- [ ] **Step 2: Run tests — verify fail**

- [ ] **Step 3: Create FunctionCategory and FunctionAttitude enums**

```java
package io.casehub.eidos.vocab;
public enum FunctionCategory { JUDGING, PERCEIVING }

package io.casehub.eidos.vocab;
public enum FunctionAttitude { INTROVERTED, EXTRAVERTED }
```

- [ ] **Step 4: Create JungianFunctionTerm enum**

Full enum with 8 constants, each implementing `axisExactMatch` per the spec §1.1 table.
Follow the `DiscTerm` pattern: anonymous subclass per constant for `axisExactMatch` overrides.

Include `shadow()`, `category()`, `attitude()`, `compatibleAuxiliaries()` methods.
Include weight tier constants as `public static final double`.

- [ ] **Step 5: Create JungianVocabRegistrar**

```java
@ApplicationScoped
public class JungianVocabRegistrar implements VocabularyRegistrar {
    @Override
    public Class<? extends Enum<? extends VocabularyTerm>> vocabulary() {
        return JungianFunctionTerm.class;
    }
}
```

- [ ] **Step 6: Run tests — verify pass**

- [ ] **Step 7: Commit**

```
git commit -m "feat(#108): JungianFunctionTerm vocabulary — 8 cognitive functions with cross-vocab mapping

Adds urn:casehub:vocab:jungian with Ti, Te, Fi, Fe, Si, Se, Ni, Ne.
Each function implements axisExactMatch to Conscientiousness and
Thomas-Kilmann per axis. Includes shadow(), category(), attitude(),
compatibleAuxiliaries() for Jungian structural rules.

Closes #108, Refs #109"
```

---

### Task 4: MbtiTypeTerm vocabulary (#110)

**Files:**
- Create: `vocab/src/main/java/io/casehub/eidos/vocab/MbtiTypeTerm.java`
- Create: `vocab/src/main/java/io/casehub/eidos/vocab/MbtiVocabRegistrar.java`
- Test: `vocab/src/test/java/io/casehub/eidos/vocab/MbtiTypeTermTest.java`

**Interfaces:**
- Consumes: `JungianFunctionTerm`, `DispositionValue`
- Produces: 16 MbtiTypeTerm constants with `specializes()` and `defaultProfile()`

- [ ] **Step 1: Write MbtiTypeTerm tests**

```java
@Test void intp_specializes_ti_and_ne() {
    assertThat(MbtiTypeTerm.INTP.specializes())
        .containsExactly(JungianFunctionTerm.TI, JungianFunctionTerm.NE);
}

@Test void default_profile_has_eight_functions() {
    var profile = MbtiTypeTerm.INTP.defaultProfile();
    assertThat(profile).hasSize(8);
}

@Test void default_profile_weights_sum_to_one() {
    var profile = MbtiTypeTerm.INTP.defaultProfile();
    double sum = profile.stream().mapToDouble(DispositionValue::weight).sum();
    assertThat(sum).isCloseTo(1.0, within(0.001));
}

@Test void default_profile_dominant_has_highest_weight() {
    var profile = MbtiTypeTerm.INTP.defaultProfile();
    var dominant = profile.stream()
        .max(Comparator.comparingDouble(DispositionValue::weight))
        .orElseThrow();
    assertThat(dominant.term()).isEqualTo("ti");
}
```

- [ ] **Step 2: Run tests — verify fail**

- [ ] **Step 3: Create MbtiTypeTerm enum**

16 constants. Each `specializes()` returns its dominant + auxiliary JungianFunctionTerms.
Each `defaultProfile()` returns 8 DispositionValues with JPAF weight distribution:
dominant ~0.35, auxiliary ~0.20, remaining 6 distributed across ~0.45.

- [ ] **Step 4: Create MbtiVocabRegistrar**

- [ ] **Step 5: Run tests — verify pass**

- [ ] **Step 6: Commit**

```
git commit -m "feat(#110): MbtiTypeTerm vocabulary — 16 MBTI types via Jungian specialization

Each type specializes its dominant + auxiliary JungianFunctionTerms.
defaultProfile() provides JPAF-parameterized 8-function weight
distributions for convenience.

Closes #110, Closes #109"
```

---

### Task 5: DispositionSignalStore SPI + implementations (#115)

**Files:**
- Create: `api/src/main/java/io/casehub/eidos/api/DispositionSignalStore.java`
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/health/NoOpDispositionSignalStore.java`
- Create: `persistence-memory/src/main/java/io/casehub/eidos/memory/InMemoryDispositionSignalStore.java`
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/health/JpaDispositionSignalStore.java`
- Test: `persistence-memory/src/test/java/io/casehub/eidos/memory/InMemoryDispositionSignalStoreTest.java`

**Interfaces:**
- Produces: `DispositionSignalStore` SPI with `recordActivation`, `activationCounts`, `decay`, `clear`

- [ ] **Step 1: Write InMemoryDispositionSignalStore tests**

```java
@Test void recordActivation_increments_count() {
    store.recordActivation("agent-1", "tenant-1", "ti");
    store.recordActivation("agent-1", "tenant-1", "ti");
    var counts = store.activationCounts("agent-1", "tenant-1");
    assertThat(counts).containsEntry("ti", 2);
}

@Test void decay_removes_twenty_percent() {
    for (int i = 0; i < 10; i++) store.recordActivation("a", "t", "ti");
    store.decay("a", "t", 0.20);
    assertThat(store.activationCounts("a", "t")).containsEntry("ti", 8);
}

@Test void decay_single_activation_drops_to_zero() {
    store.recordActivation("a", "t", "ne");
    store.decay("a", "t", 0.20);
    assertThat(store.activationCounts("a", "t").getOrDefault("ne", 0)).isEqualTo(0);
}

@Test void clear_removes_all() {
    store.recordActivation("a", "t", "ti");
    store.clear("a", "t");
    assertThat(store.activationCounts("a", "t")).isEmpty();
}

@Test void tenancy_isolation() {
    store.recordActivation("a", "t1", "ti");
    store.recordActivation("a", "t2", "fe");
    assertThat(store.activationCounts("a", "t1")).containsOnlyKeys("ti");
    assertThat(store.activationCounts("a", "t2")).containsOnlyKeys("fe");
}
```

- [ ] **Step 2: Create DispositionSignalStore SPI**

```java
package io.casehub.eidos.api;

import java.util.Map;

public interface DispositionSignalStore {
    void recordActivation(String agentId, String tenancyId, String functionTerm);
    Map<String, Integer> activationCounts(String agentId, String tenancyId);
    void decay(String agentId, String tenancyId, double decayFactor);
    void clear(String agentId, String tenancyId);
}
```

- [ ] **Step 3: Create NoOpDispositionSignalStore**

```java
@ApplicationScoped @DefaultBean
public class NoOpDispositionSignalStore implements DispositionSignalStore {
    @Override public void recordActivation(String a, String t, String f) {}
    @Override public Map<String, Integer> activationCounts(String a, String t) { return Map.of(); }
    @Override public void decay(String a, String t, double d) {}
    @Override public void clear(String a, String t) {}
}
```

- [ ] **Step 4: Create InMemoryDispositionSignalStore**

```java
@Alternative @Priority(1) @ApplicationScoped
public class InMemoryDispositionSignalStore implements DispositionSignalStore {
    private final Map<String, Map<String, Integer>> store = new ConcurrentHashMap<>();
    // key = agentId + ":" + tenancyId
    // ...
}
```

- [ ] **Step 5: Create JpaDispositionSignalStore**

Uses the `disposition_signal` table from V9 migration. `@IfBuildProperty` gated (blocking mode).

- [ ] **Step 6: Run tests — verify pass**

- [ ] **Step 7: Commit**

```
git commit -m "feat(#115): DispositionSignalStore SPI + NoOp, InMemory, JPA implementations

Purpose-built store for disposition function activation tracking.
Accumulative counts (no TTL) with explicit decay and clear operations.
Follows CDI displacement ladder: NoOp @DefaultBean, InMemory @Alternative,
JPA @IfBuildProperty.

Closes #115"
```

---

### Task 6: DispositionHealth + DispositionEvolution SPIs (#116)

**Files:**
- Create: `api/src/main/java/io/casehub/eidos/api/DispositionHealth.java`
- Create: `api/src/main/java/io/casehub/eidos/api/DispositionEvolution.java`
- Create: `api/src/main/java/io/casehub/eidos/api/EvolutionType.java`
- Create: `vocab/src/main/java/io/casehub/eidos/vocab/JungianEvolutionType.java`
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/health/NoOpDispositionHealth.java`
- Test: `api/src/test/java/io/casehub/eidos/api/DispositionHealthTest.java` (sealed interface tests)

**Interfaces:**
- Consumes: `AgentDescriptor`, `CapabilityHealth.ProbeContext`, `DispositionValue`
- Produces: `DispositionHealth.probe()` → `DispositionStatus` sealed interface
- Produces: `DispositionEvolution.evaluate()` → `EvolutionResult` sealed interface
- Produces: `EvolutionType` interface + `JungianEvolutionType` enum

- [ ] **Step 1: Create EvolutionType interface (api tier, framework-agnostic)**

```java
package io.casehub.eidos.api;
public interface EvolutionType { String name(); }
```

- [ ] **Step 2: Create JungianEvolutionType enum (vocab tier)**

```java
package io.casehub.eidos.vocab;
import io.casehub.eidos.api.EvolutionType;

public enum JungianEvolutionType implements EvolutionType {
    DOMINANT_AUXILIARY_SWAP,
    DOMINANT_REPLACEMENT,
    AUXILIARY_REPLACEMENT,
    STRUCTURAL_REORGANIZATION
}
```

- [ ] **Step 3: Create DispositionHealth SPI + DispositionStatus sealed interface**

Per spec §2.6: `Aligned`, `Drifted` (with L2 driftMagnitude), `EvolutionPending`.

- [ ] **Step 4: Create DispositionEvolution SPI + EvolutionResult sealed interface**

Per spec §2.7: `Evolved` (with framework-agnostic `previousTypeLabel`/`newTypeLabel`), `Dampened`.

- [ ] **Step 5: Create NoOpDispositionHealth**

Always returns `Aligned` with empty weights map.

- [ ] **Step 6: Write sealed interface tests**

Verify pattern matching works for all status variants. Verify record field access.

- [ ] **Step 7: Commit**

```
git commit -m "feat(#116): DispositionHealth + DispositionEvolution SPIs

DispositionHealth.probe() returns sealed Aligned/Drifted/EvolutionPending.
DispositionEvolution.evaluate() returns sealed Evolved/Dampened.
EvolutionType interface in api, JungianEvolutionType enum in vocab.
NoOp @DefaultBean DispositionHealth in runtime.

Refs #116"
```

---

### Task 7: DefaultDispositionHealth implementation (#116)

**Files:**
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/health/DefaultDispositionHealth.java`
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/preferences/DispositionPreferenceKeys.java`
- Test: `runtime/src/test/java/io/casehub/eidos/runtime/health/DefaultDispositionHealthTest.java`

**Interfaces:**
- Consumes: `DispositionSignalStore`, `VocabularyRegistry`, `PreferenceProvider`, `JungianFunctionTerm`
- Produces: Full probe implementation with JPAF threshold conditions

- [ ] **Step 1: Write probe tests**

Test all status outcomes:
- Aligned when no signals accumulated
- Drifted when signals accumulated but no threshold crossed
- EvolutionPending(DOMINANT_AUXILIARY_SWAP) when auxiliary effective ≥ dominant base
- EvolutionPending(DOMINANT_REPLACEMENT) when shadow of dominant ≥ dominant base
- EvolutionPending(AUXILIARY_REPLACEMENT) when shadow of auxiliary ≥ auxiliary base
- EvolutionPending(STRUCTURAL_REORGANIZATION) when unrelated ≥ dominant base
- driftMagnitude computed as L2 distance
- Returns Aligned when no dispositionProfile present

- [ ] **Step 2: Implement DefaultDispositionHealth**

```java
@ApplicationScoped
public class DefaultDispositionHealth implements DispositionHealth {
    @Inject DispositionSignalStore signalStore;
    @Inject VocabularyRegistry vocabRegistry;
    @Inject Instance<PreferenceProvider> preferenceProviders;

    @Override
    public DispositionStatus probe(AgentDescriptor descriptor, ProbeContext context) {
        var profile = descriptor.disposition().dispositionProfile();
        if (profile == null || profile.isEmpty()) {
            return new DispositionStatus.Aligned(Map.of());
        }
        // Compute effective weights = base + (activationCount × delta)
        // Check JPAF threshold conditions
        // Return appropriate status
    }
}
```

- [ ] **Step 3: Run tests — verify pass**

- [ ] **Step 4: Commit**

```
git commit -m "feat(#116): DefaultDispositionHealth — JPAF threshold probe

Implements all 4 evolution detection conditions: dominant-auxiliary swap,
dominant replacement (shadow), auxiliary replacement (shadow), structural
reorganization. L2 drift magnitude. Tenancy-scoped thresholds via
PreferenceProvider.

Closes #116"
```

---

### Task 8: Auto-derivation — profile → weighted axes (#111)

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/registrar/DescriptorCollector.java`
- Test: `runtime/src/test/java/io/casehub/eidos/runtime/registrar/DispositionProfileDerivationTest.java`

**Interfaces:**
- Consumes: `AgentDisposition.dispositionProfile`, `VocabularyRegistry`, `JungianFunctionTerm.axisExactMatch`
- Produces: Auto-populated axis fields + axisVocabularies map on the descriptor

- [ ] **Step 1: Write auto-derivation tests**

```java
@Test void intp_profile_derives_independent_dominant_on_social_orient() {
    var profile = MbtiTypeTerm.INTP.defaultProfile();
    var descriptor = AgentDescriptor.builder()
        .agentId("test").name("test").tenancyId("t")
        .dispositionVocabulary(JungianFunctionTerm.URI)
        .disposition(AgentDisposition.builder()
            .dispositionProfile(profile.toArray(DispositionValue[]::new))
            .build())
        .build();

    var derived = deriver.derive(descriptor, vocabRegistry);

    assertThat(derived.disposition().socialOrient())
        .extracting(DispositionValue::term)
        .contains("independent");
}

@Test void explicit_axis_values_take_precedence() {
    // When both profile and axis values are specified,
    // explicit axis values are NOT overwritten
}

@Test void axis_vocabularies_populated_for_derived_axes() {
    // axisVocabularies map should contain entries for all 5 axes
    // pointing to conscientiousness and thomas-kilmann URIs
}
```

- [ ] **Step 2: Implement derivation logic in DescriptorCollector**

Per spec §2.4: iterate profile functions, resolve axisExactMatch per axis, weight by function weight, aggregate, normalize per axis.

- [ ] **Step 3: Run tests — verify pass**

- [ ] **Step 4: Commit**

```
git commit -m "feat(#111): auto-derivation — disposition profile → weighted axes

When dispositionProfile is populated and axes are empty, projects
function weights onto 5 disposition axes via cross-vocabulary resolution.
Populates axisVocabularies for derived axes. Explicit axis values
take precedence over auto-derived.

Refs #111"
```

---

### Task 9: Weighted axes + cognitive profile rendering (#111)

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosRenderPipeline.java`
- Test: `runtime/src/test/java/io/casehub/eidos/runtime/renderer/EidosRenderPipelineTest.java`
- Test: `runtime/src/test/java/io/casehub/eidos/runtime/renderer/EidosSystemPromptRendererTest.java`

**Interfaces:**
- Consumes: `AgentDisposition` with weighted axes and dispositionProfile
- Produces: MARKDOWN/PROSE/A2A_CARD output with weighted values and JPAF-style cognitive prompting

- [ ] **Step 1: Write rendering tests**

```java
@Test void weighted_axis_renders_primarily_x_with_y_tendencies() {
    // socialOrient: [{independent, 0.7}, {collaborative, 0.3}]
    // → "Social orientation: primarily independent (0.7), with collaborative tendencies (0.3)"
}

@Test void single_axis_value_renders_without_weight() {
    // socialOrient: [{independent, 1.0}]
    // → "Social orientation: independent"  (unchanged from today)
}

@Test void jungian_profile_renders_cognitive_style_section() {
    // dispositionProfile with Jungian functions
    // → "## Cognitive Style" section with dominant/auxiliary
}

@Test void a2a_card_includes_disposition_profile() {
    // A2A_CARD format includes raw profile weights + derived MBTI type
}
```

- [ ] **Step 2: Update assembleMarkdownDisposition**

Currently iterates `DispositionAxis.values()` and calls `d.get(axis)` which returned `Optional<String>`. Now returns `List<DispositionValue>`. Update to render weighted values.

- [ ] **Step 3: Add cognitive profile rendering**

New method `assembleMarkdownCognitiveProfile` that renders the JPAF-style cognitive style section when `dispositionProfile` is populated and `dispositionVocabulary` is Jungian.

- [ ] **Step 4: Update A2A_CARD assembly**

Add `dispositionProfile` JSON block with vocabulary, functions (term + weight + derived role), and derivedMbtiType.

- [ ] **Step 5: Update buildDescriptorPayload**

The descriptor payload for LLM enrichment needs to include weighted axis values and cognitive profile.

- [ ] **Step 6: Run tests — verify pass**

- [ ] **Step 7: Commit**

```
git commit -m "feat(#111): weighted disposition + cognitive profile rendering

MARKDOWN renders weighted axes as 'primarily X, with Y tendencies'.
Jungian-profiled agents get a Cognitive Style section with dominant/
auxiliary function descriptions and compensation instructions.
A2A_CARD includes raw profile weights and derived MBTI type.

Refs #111"
```

---

### Task 10: DefaultDispositionEvolution + tests (#116)

**Files:**
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/health/DefaultDispositionEvolution.java`
- Test: `runtime/src/test/java/io/casehub/eidos/runtime/health/DefaultDispositionEvolutionTest.java`

**Interfaces:**
- Consumes: `DispositionHealth.EvolutionPending`, `ChatModel` (optional), `JungianFunctionTerm`, `MbtiTypeTerm`
- Produces: `EvolutionResult.Evolved` or `EvolutionResult.Dampened`

- [ ] **Step 1: Write evolution tests**

```java
@Test void rule_based_swap_when_no_chat_model() {
    // Auxiliary weight exceeds dominant → swap
    // Returns Evolved with swapped profile
}

@Test void dampened_when_reflection_says_no() {
    // LLM-adjudicated: model returns "no"
    // Returns Dampened(0.2)
}

@Test void evolved_profile_respects_weight_tiers() {
    // After evolution: dominant in [0.31, 1.0], auxiliary in [0.06, 0.30]
}

@Test void evolved_profile_sums_to_one() {
    // All weights in evolved profile sum to 1.0
}
```

- [ ] **Step 2: Implement DefaultDispositionEvolution**

Rule-based fallback: apply JPAF reflection rules deterministically.
LLM-adjudicated (when ChatModel available): present agent history and weight profile, ask for judgment.

- [ ] **Step 3: Run tests — verify pass**

- [ ] **Step 4: Commit**

```
git commit -m "feat(#116): DefaultDispositionEvolution — JPAF reflection rules

Implements all 4 evolution types: dominant-auxiliary swap, dominant
replacement, auxiliary replacement, structural reorganization.
Rule-based fallback when no ChatModel available. LLM-adjudicated
transitions match JPAF's approach.

Refs #116"
```

---

### Task 11: Update personality-frameworks.md (#114)

**Files:**
- Modify: `docs/personality-frameworks.md`

- [ ] **Step 1: Add §2.4 Jungian Cognitive Functions**

Per spec §6.1: 8 functions, descriptions, JPAF findings.

- [ ] **Step 2: Revise §2.3 MBTI**

Per spec §6.2: keep original rejection, add "Jungian rehabilitation" subsection.

- [ ] **Step 3: Update Anti-pattern 1**

Per spec §6.2: distinguish human-measured (inadvisable) from agent-specified (valid).

- [ ] **Step 4: Update cross-reference table (§5)**

Add Jungian column per spec §6.3.

- [ ] **Step 5: Add Jungian Profile combination pattern**

Per spec §6.4.

- [ ] **Step 6: Update framework compatibility (§6)**

Per spec §6.5: Jungian + Belbin = Additive, etc.

- [ ] **Step 7: Commit**

```
git commit -m "docs(#114): update personality-frameworks.md — Jungian rehabilitation

Adds §2.4 Jungian Cognitive Functions with JPAF findings. Revises §2.3
MBTI: keeps original rejection for human-measured, adds Jungian
rehabilitation for agent-specified. Updates cross-reference table,
combination patterns, and compatibility ratings.

Closes #114"
```

---

### Task 12: Eval judges — MbtiAlignment, FunctionActivation, PersonalityEvolution (#113, #117)

**Files:**
- Create: `eval/src/main/java/io/casehub/eidos/eval/MbtiAlignmentJudge.java`
- Create: `eval/src/main/java/io/casehub/eidos/eval/FunctionActivationJudge.java`
- Create: `eval/src/main/java/io/casehub/eidos/eval/PersonalityEvolutionJudge.java`
- Create: `eval/src/test/resources/jungian-profiles/` — YAML agent profiles for all 16 MBTI types
- Create: `eval/src/test/resources/function-scenarios/` — 8 scenario sets for TAA
- Test: `eval/src/test/java/io/casehub/eidos/eval/MbtiAlignmentJudgeTest.java`
- Test: `eval/src/test/java/io/casehub/eidos/eval/FunctionActivationJudgeTest.java`

**Interfaces:**
- Consumes: `SystemPromptRenderer`, `DispositionHealth`, `DispositionEvolution`, `DispositionSignalStore`
- Produces: MBTI alignment scores (per-dimension accuracy), TAA (activation accuracy), PSA (shift accuracy)

- [ ] **Step 1: Create YAML agent profiles for 16 MBTI types**

Each profile specifies a Jungian cognitive function dispositionProfile with the MBTI type's default weights. Minimum: 8 types (one per dominant function).

- [ ] **Step 2: Create function activation scenario corpus**

8 sets × 3 scenarios × 5 questions = 120 items. Software engineering contexts.

- [ ] **Step 3: Implement MbtiAlignmentJudge**

Per spec §4.1: render agent, administer MBTI-70 questionnaire items, score per-dimension accuracy.

- [ ] **Step 4: Implement FunctionActivationJudge**

Per spec §4.2: present scenarios, parse function identification from response, compute TAA.

- [ ] **Step 5: Implement PersonalityEvolutionJudge**

Per spec §4.3: run scenarios, record signals, invoke probe + evolution, validate transitions.

- [ ] **Step 6: Write unit tests**

Test output schema parsing, scoring logic, validation rules. LLM-dependent integration tests run via eval profile.

- [ ] **Step 7: Commit**

```
git commit -m "feat(#113,#117): eval judges — MbtiAlignment, FunctionActivation, PersonalityEvolution

MbtiAlignmentJudge: MBTI-70 questionnaire alignment per dimension.
FunctionActivationJudge: TAA — cognitive function activation accuracy.
PersonalityEvolutionJudge: PSA — personality shift structural validity.
16 YAML profiles + 120 function activation scenarios.

Closes #113, Closes #117"
```

---

## Dependency Graph

```
Task 1 (API evolution)
  ├── Task 2 (Schema + JPA + all module ripple)
  │     ├── Task 3 (JungianFunctionTerm)
  │     │     └── Task 4 (MbtiTypeTerm)
  │     ├── Task 5 (DispositionSignalStore)
  │     ├── Task 6 (Health + Evolution SPIs)
  │     │     ├── Task 7 (DefaultDispositionHealth)
  │     │     └── Task 10 (DefaultDispositionEvolution)
  │     ├── Task 8 (Auto-derivation) ← depends on Task 3
  │     ├── Task 9 (Rendering) ← depends on Task 3
  │     └── Task 12 (Eval) ← depends on Tasks 3,4,5,6,7,9,10
  └── Task 11 (Documentation) ← can run after Task 4
```

## Issue Closure Map

| Task | Closes |
|------|--------|
| 1 | — (foundation, refs #111, #112) |
| 2 | — (ripple, refs #111) |
| 3 | #108, refs #109 |
| 4 | #110, #109 |
| 5 | #115 |
| 6 | refs #116 |
| 7 | #116 |
| 8 | refs #111 |
| 9 | refs #111 |
| 10 | refs #116 |
| 11 | #114 |
| 12 | #113, #117 |
