# Capability Sub-Specialization Metadata Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `excludedDomains` to `AgentCapability`, introduce `CapabilitySpecializationStore` SPI for learned DECLINE-based exclusions, and wire both into `DefaultCapabilityHealth.probe()` as new steps 3 and 4.

**Architecture:** `AgentCapability` gains a declared `excludedDomains: Set<String>` field. A new `CapabilitySpecializationStore` SPI (in `casehub-eidos-api`) stores per-domain DECLINE counts with TTL; `InMemoryCapabilitySpecializationStore` is the `@Alternative @Priority(1)` implementation in `persistence-memory`. `DefaultCapabilityHealth` checks declared then learned exclusions before the existing epistemic-weakness check. `CapabilityStatus` gains an `Excluded(domain, ExclusionSource, declineCount)` sealed variant.

**Tech Stack:** Java 21, Quarkus 3.32.2, JPA/Hibernate, AssertJ + JUnit 5, Mockito, MicroProfile Config, `casehub-platform-api` (PreferenceProvider / PreferenceKey / SettingsScope / SingleValuePreference).

---

## File Map

### `api/` module — create or modify

| File | Action |
|------|--------|
| `api/src/main/java/io/casehub/eidos/api/AgentCapability.java` | Modify — add `excludedDomains` field, add `Builder` inner class, add overlap validation in compact constructor |
| `api/src/main/java/io/casehub/eidos/api/CapabilityHealth.java` | Modify — add `Excluded(domain, ExclusionSource, declineCount)` to sealed `CapabilityStatus`; add `ExclusionSource` enum |
| `api/src/main/java/io/casehub/eidos/api/CapabilitySpecializationStore.java` | Create — new SPI with `recordDecline`, `clearDeclines`, `learnedExclusions`, `declineCount` |
| `api/src/test/java/io/casehub/eidos/api/AgentCapabilityTest.java` | Modify — migrate `valid()` to builder; add `excludedDomains` validation tests; add overlap invariant test |

### `runtime/` module — create or modify

| File | Action |
|------|--------|
| `runtime/pom.xml` | Modify — add `casehub-platform-api` compile dependency |
| `runtime/src/main/java/io/casehub/eidos/runtime/preferences/EidosPreferenceKeys.java` | Create — `EXCLUDE_THRESHOLD` `PreferenceKey` constant |
| `runtime/src/main/java/io/casehub/eidos/runtime/preferences/ExcludeThresholdPreference.java` | Create — record implementing `SingleValuePreference` |
| `runtime/src/main/java/io/casehub/eidos/runtime/health/NoOpCapabilitySpecializationStore.java` | Create — `@DefaultBean @ApplicationScoped`, all methods no-op |
| `runtime/src/main/java/io/casehub/eidos/runtime/health/DefaultCapabilityHealth.java` | Modify — inject `CapabilitySpecializationStore` + `Instance<PreferenceProvider>`; add probe steps 3+4 |
| `runtime/src/main/resources/db/eidos/migration/V1__initial_schema.sql` | Modify — add `excluded_domains TEXT` column to `agent_capability` |
| `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentCapabilityEntity.java` | Modify — add `excludedDomains` field |
| `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentDescriptorMapper.java` | Modify — map `excludedDomains` in `toCapability` and `toCapabilityEntity` |
| `runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosRenderPipeline.java` | Modify — add `excludedDomains` array to A2A_CARD capability node |
| `runtime/src/test/java/io/casehub/eidos/runtime/health/DefaultCapabilityHealthTest.java` | Modify — migrate `capability()` factory to builder |
| `runtime/src/test/java/io/casehub/eidos/runtime/health/DefaultCapabilityHealthDegradedTest.java` | Modify — migrate `capability()` factory to builder; add `Excluded` priority test |
| `runtime/src/test/java/io/casehub/eidos/runtime/health/DefaultCapabilityHealthExclusionTest.java` | Create — new JUnit (non-Quarkus) tests for steps 3+4 with stubs/mocks |
| `runtime/src/test/java/io/casehub/eidos/runtime/renderer/EidosRenderPipelineTest.java` | Modify — add `excludedDomains` A2A tests; migrate `AgentCapability` constructor calls |

### Call sites to migrate (positional → builder) across all modules

Every `new AgentCapability(...)` positional call must move to `AgentCapability.builder()...build()`. Files:

- `api/src/test/java/io/casehub/eidos/api/AgentCapabilityTest.java`
- `api/src/test/java/io/casehub/eidos/api/AgentDescriptorTest.java`
- `runtime/src/test/java/io/casehub/eidos/runtime/health/DefaultCapabilityHealthTest.java`
- `runtime/src/test/java/io/casehub/eidos/runtime/health/DefaultCapabilityHealthDegradedTest.java`
- `runtime/src/test/java/io/casehub/eidos/runtime/health/DefaultReactiveCapabilityHealthTest.java`
- `runtime/src/test/java/io/casehub/eidos/runtime/health/DefaultReactiveCapabilityHealthDefaultProfileTest.java`
- `runtime/src/test/java/io/casehub/eidos/runtime/registry/JpaAgentRegistryTest.java`
- `runtime/src/test/java/io/casehub/eidos/runtime/registry/JpaReactiveAgentRegistryTest.java`
- `runtime/src/test/java/io/casehub/eidos/runtime/renderer/EidosRenderPipelineTest.java`
- `runtime/src/test/java/io/casehub/eidos/runtime/renderer/EidosSystemPromptRendererTest.java`
- `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentDescriptorMapper.java` (in `toCapability`)
- `persistence-memory/src/test/java/io/casehub/eidos/memory/InMemoryAgentRegistryTest.java`
- `persistence-memory/src/test/java/io/casehub/eidos/memory/InMemoryReactiveAgentRegistryTest.java`
- `eval/src/main/java/io/casehub/eidos/eval/EvalDataset.java`
- `eval/src/test/java/io/casehub/eidos/eval/PromptJudgeTest.java`
- `eval/src/test/java/io/casehub/eidos/eval/EvalResultCompletenessTest.java`
- `examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/*.java` (all example tests)

### `persistence-memory/` module — create

| File | Action |
|------|--------|
| `persistence-memory/src/main/java/io/casehub/eidos/memory/InMemoryCapabilitySpecializationStore.java` | Create |
| `persistence-memory/src/test/java/io/casehub/eidos/memory/InMemoryCapabilitySpecializationStoreTest.java` | Create |

---

## Task 1: `AgentCapability` — add `excludedDomains`, builder, and validation

**Files:**
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentCapability.java`
- Modify: `api/src/test/java/io/casehub/eidos/api/AgentCapabilityTest.java`
- Migrate: all positional `new AgentCapability(...)` call sites listed above

- [ ] **Step 1.1: Write failing tests for `excludedDomains` validation and overlap invariant**

Add to the bottom of `api/src/test/java/io/casehub/eidos/api/AgentCapabilityTest.java`, before the closing brace:

```java
    // ── excludedDomains ────────────────────────────────────────────────────────

    @Test
    void excluded_domains_null_is_allowed() {
        assertThatNoException().isThrownBy(() ->
            AgentCapability.builder().name("review").build());
    }

    @Test
    void excluded_domains_blank_entry_throws() {
        assertThatThrownBy(() ->
            AgentCapability.builder()
                .name("review")
                .excludedDomains(java.util.Set.of("rust", "  "))
                .build())
            .isInstanceOf(AgentValidationException.class);
    }

    @Test
    void excluded_domains_over_length_throws() {
        assertThatThrownBy(() ->
            AgentCapability.builder()
                .name("review")
                .excludedDomains(java.util.Set.of("d".repeat(201)))
                .build())
            .isInstanceOf(AgentValidationException.class);
    }

    @Test
    void excluded_domains_defensive_copy_prevents_external_mutation() {
        var domains = new java.util.HashSet<>(java.util.Set.of("rust"));
        var cap = AgentCapability.builder().name("review").excludedDomains(domains).build();
        domains.add("go");
        assertThat(cap.excludedDomains()).doesNotContain("go");
    }

    @Test
    void domain_in_both_excluded_and_epistemic_throws() {
        assertThatThrownBy(() ->
            AgentCapability.builder()
                .name("security-review")
                .epistemicDomains(Map.of("java", 0.95))
                .excludedDomains(java.util.Set.of("java"))
                .build())
            .isInstanceOf(AgentValidationException.class)
            .hasMessageContaining("java");
    }

    @Test
    void builder_round_trips_all_fields() {
        var cap = AgentCapability.builder()
            .name("code-review")
            .qualityHint(0.9)
            .latencyHintP50Ms(100L)
            .costHint("low")
            .inputTypes(List.of("pull-request"))
            .outputTypes(List.of("review-comment"))
            .tags(List.of("java"))
            .epistemicDomains(Map.of("java", 0.95))
            .excludedDomains(java.util.Set.of("rust"))
            .build();
        assertThat(cap.name()).isEqualTo("code-review");
        assertThat(cap.qualityHint()).isEqualTo(0.9);
        assertThat(cap.latencyHintP50Ms()).isEqualTo(100L);
        assertThat(cap.costHint()).isEqualTo("low");
        assertThat(cap.inputTypes()).containsExactly("pull-request");
        assertThat(cap.outputTypes()).containsExactly("review-comment");
        assertThat(cap.tags()).containsExactly("java");
        assertThat(cap.epistemicDomains()).containsEntry("java", 0.95);
        assertThat(cap.excludedDomains()).containsExactly("rust");
    }
```

- [ ] **Step 1.2: Run the new tests — confirm compile failure (field doesn't exist yet)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=AgentCapabilityTest 2>&1 | tail -20
```

Expected: `COMPILATION ERROR — cannot find symbol: method excludedDomains(java.util.Set<String>)` (builder doesn't exist yet).

- [ ] **Step 1.3: Implement `AgentCapability` with `excludedDomains` field and builder**

Replace `api/src/main/java/io/casehub/eidos/api/AgentCapability.java` entirely:

```java
package io.casehub.eidos.api;

import java.util.List;
import java.util.Map;
import java.util.Set;

/**
 * A declared capability of an agent with operational metadata and epistemic domain qualifications.
 * qualityHint is a self-declared prior — ActorTrustScore per CapabilityTag in casehub-ledger
 * is the evidence-backed replacement that accumulates over time.
 */
public record AgentCapability(
        String name,
        Double qualityHint,
        Long latencyHintP50Ms,
        String costHint,
        List<String> inputTypes,
        List<String> outputTypes,
        List<String> tags,
        Map<String, Double> epistemicDomains,
        Set<String> excludedDomains
) {
    public AgentCapability {
        AgentDescriptorValidator.validateRequired("capability.name", name,
            AgentDescriptorValidator.MAX_CAPABILITY_NAME);
        AgentDescriptorValidator.validateOptional("costHint", costHint,
            AgentDescriptorValidator.MAX_CAPABILITY_STRING);
        AgentDescriptorValidator.validateItems("inputTypes", inputTypes,
            AgentDescriptorValidator.MAX_CAPABILITY_STRING);
        AgentDescriptorValidator.validateItems("outputTypes", outputTypes,
            AgentDescriptorValidator.MAX_CAPABILITY_STRING);
        AgentDescriptorValidator.validateItems("tags", tags,
            AgentDescriptorValidator.MAX_CAPABILITY_STRING);
        if (epistemicDomains != null) {
            AgentDescriptorValidator.validateMapKeys("epistemicDomains",
                epistemicDomains.keySet(), AgentDescriptorValidator.MAX_CAPABILITY_STRING);
        }
        if (excludedDomains != null) {
            AgentDescriptorValidator.validateItems("excludedDomains",
                excludedDomains, AgentDescriptorValidator.MAX_CAPABILITY_STRING);
            if (epistemicDomains != null) {
                excludedDomains.stream()
                    .filter(epistemicDomains::containsKey)
                    .findFirst()
                    .ifPresent(d -> { throw new AgentValidationException("excludedDomains",
                        "domain '" + d + "' appears in both excludedDomains and epistemicDomains"); });
            }
            excludedDomains = Set.copyOf(excludedDomains);
        }
    }

    public static Builder builder() { return new Builder(); }

    public static final class Builder {
        private String name;
        private Double qualityHint;
        private Long latencyHintP50Ms;
        private String costHint;
        private List<String> inputTypes;
        private List<String> outputTypes;
        private List<String> tags;
        private Map<String, Double> epistemicDomains;
        private Set<String> excludedDomains;

        public Builder name(String v)                     { this.name = v; return this; }
        public Builder qualityHint(Double v)              { this.qualityHint = v; return this; }
        public Builder latencyHintP50Ms(Long v)           { this.latencyHintP50Ms = v; return this; }
        public Builder costHint(String v)                 { this.costHint = v; return this; }
        public Builder inputTypes(List<String> v)         { this.inputTypes = v; return this; }
        public Builder outputTypes(List<String> v)        { this.outputTypes = v; return this; }
        public Builder tags(List<String> v)               { this.tags = v; return this; }
        public Builder epistemicDomains(Map<String, Double> v) { this.epistemicDomains = v; return this; }
        public Builder excludedDomains(Set<String> v)     { this.excludedDomains = v; return this; }

        public AgentCapability build() {
            return new AgentCapability(name, qualityHint, latencyHintP50Ms, costHint,
                inputTypes, outputTypes, tags, epistemicDomains, excludedDomains);
        }
    }
}
```

- [ ] **Step 1.4: Run the new tests — confirm they now pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=AgentCapabilityTest 2>&1 | tail -20
```

Expected: The new tests pass; the existing tests FAIL to compile (they use 8-arg positional constructor).

- [ ] **Step 1.5: Migrate `AgentCapabilityTest` existing tests from positional constructor to builder**

Replace the `valid()` factory and update all 8-arg positional constructor calls in `AgentCapabilityTest.java`. The existing tests test field-level validation — use the builder with only the relevant field set:

```java
    static AgentCapability valid() {
        return AgentCapability.builder()
            .name("code-review").qualityHint(0.9).latencyHintP50Ms(100L).costHint("low")
            .inputTypes(List.of("pull-request")).outputTypes(List.of("review-comment"))
            .tags(List.of("java")).epistemicDomains(Map.of("java", 0.95))
            .build();
    }

    // name_null_throws
    assertThatThrownBy(() -> AgentCapability.builder().name(null).build())

    // name_blank_throws
    assertThatThrownBy(() -> AgentCapability.builder().name("  ").build())

    // name_exceeds_100_chars_throws
    assertThatThrownBy(() -> AgentCapability.builder().name("n".repeat(101)).build())

    // name_with_bidi_control_throws
    assertThatThrownBy(() -> AgentCapability.builder().name("code‮review").build())

    // name_at_exactly_100_chars_is_valid
    assertThatNoException().isThrownBy(() -> AgentCapability.builder().name("n".repeat(100)).build())

    // cost_hint_null_is_allowed
    assertThatNoException().isThrownBy(() -> AgentCapability.builder().name("review").build())

    // cost_hint_blank_throws
    assertThatThrownBy(() -> AgentCapability.builder().name("review").costHint("  ").build())

    // cost_hint_with_injection_char_throws  (U+200B ZERO WIDTH SPACE)
    assertThatThrownBy(() -> AgentCapability.builder().name("review").costHint("low​cost").build())

    // input_types_null_list_is_allowed
    assertThatNoException().isThrownBy(() -> AgentCapability.builder().name("review").inputTypes(null).build())

    // input_types_blank_item_throws_with_index
    assertThatThrownBy(() -> AgentCapability.builder().name("review")
        .inputTypes(List.of("pr", "")).build())

    // output_types_item_with_c0_control_char_throws  (U+0001)
    assertThatThrownBy(() -> AgentCapability.builder().name("review")
        .outputTypes(List.of("commentinject")).build())

    // tags_item_exceeds_200_chars_throws
    assertThatThrownBy(() -> AgentCapability.builder().name("review")
        .tags(List.of("t".repeat(201))).build())

    // epistemic_domain_key_with_alm_throws  (U+061C)
    assertThatThrownBy(() -> AgentCapability.builder().name("review")
        .epistemicDomains(Map.of("java؜domain", 0.9)).build())

    // epistemic_domain_null_map_is_allowed
    assertThatNoException().isThrownBy(() -> AgentCapability.builder().name("review")
        .epistemicDomains(null).build())
```

- [ ] **Step 1.6: Migrate all other call sites across all modules**

**Pattern:** Replace `new AgentCapability(n, q, l, c, inp, out, tags, epistemic)` with `AgentCapability.builder().name(n).qualityHint(q).latencyHintP50Ms(l).costHint(c).inputTypes(inp).outputTypes(out).tags(tags).epistemicDomains(epistemic).build()`. Omit null fields entirely — the builder defaults them to null.

**Shorthand for the most common form** (many call sites have mostly-null fields):
```java
// Before:
new AgentCapability("code-review", 0.9, null, null, List.of(), List.of(), List.of(), null)
// After:
AgentCapability.builder().name("code-review").qualityHint(0.9).build()

// Before (with epistemic):
new AgentCapability("code-review", 0.9, 150L, "low",
    List.of("pr"), List.of("comment"), List.of("java"), Map.of("java", 0.95))
// After:
AgentCapability.builder().name("code-review").qualityHint(0.9).latencyHintP50Ms(150L)
    .costHint("low").inputTypes(List.of("pr")).outputTypes(List.of("comment"))
    .tags(List.of("java")).epistemicDomains(Map.of("java", 0.95)).build()
```

Apply this pattern to every file in the call sites list. The `AgentDescriptorMapper.toCapability()` becomes:
```java
private AgentCapability toCapability(AgentCapabilityEntity c) {
    return AgentCapability.builder()
        .name(c.name)
        .qualityHint(c.qualityHint)
        .latencyHintP50Ms(c.latencyHintP50Ms)
        .costHint(c.costHint)
        .inputTypes(readJson(c.inputTypes, new TypeReference<List<String>>() {}))
        .outputTypes(readJson(c.outputTypes, new TypeReference<List<String>>() {}))
        .tags(readJson(c.tags, new TypeReference<List<String>>() {}))
        .epistemicDomains(readJson(c.epistemicDomains, new TypeReference<Map<String, Double>>() {}))
        .excludedDomains(readJson(c.excludedDomains, new TypeReference<Set<String>>() {}))
        .build();
}
```

The `capability()` factory in `DefaultCapabilityHealthTest`:
```java
static AgentCapability capability(String name, Map<String, Double> epistemicDomains) {
    return AgentCapability.builder().name(name).qualityHint(0.9)
        .epistemicDomains(epistemicDomains).build();
}
```

The `capability()` factory in `DefaultCapabilityHealthDegradedTest`:
```java
static AgentCapability capability(final String name, final Map<String, Double> epistemicDomains) {
    return AgentCapability.builder().name(name).qualityHint(0.9)
        .epistemicDomains(epistemicDomains).build();
}
```

- [ ] **Step 1.7: Run full build to confirm all modules compile and all existing tests pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -DskipTests 2>&1 | tail -10
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api 2>&1 | tail -20
```

Expected: `BUILD SUCCESS` on both.

- [ ] **Step 1.8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add -A
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#55): add excludedDomains field + builder to AgentCapability

Refs #55"
```

---

## Task 2: `CapabilityStatus.Excluded` + `CapabilitySpecializationStore` SPI + `NoOpCapabilitySpecializationStore`

**Files:**
- Modify: `api/src/main/java/io/casehub/eidos/api/CapabilityHealth.java`
- Create: `api/src/main/java/io/casehub/eidos/api/CapabilitySpecializationStore.java`
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/health/NoOpCapabilitySpecializationStore.java`

- [ ] **Step 2.1: Add `Excluded` variant to `CapabilityStatus` in `CapabilityHealth.java`**

Replace the `sealed interface CapabilityStatus` block:

```java
    sealed interface CapabilityStatus permits
            CapabilityStatus.Ready,
            CapabilityStatus.EpistemicallyWeak,
            CapabilityStatus.Excluded,
            CapabilityStatus.Degraded,
            CapabilityStatus.Unavailable {

        record Ready() implements CapabilityStatus {}
        record Degraded(DegradationReason reason, String detail) implements CapabilityStatus {}
        record Unavailable(String reason) implements CapabilityStatus {}
        record EpistemicallyWeak(String domain, double confidence) implements CapabilityStatus {}
        record Excluded(String domain, ExclusionSource source, int declineCount) implements CapabilityStatus {}

        enum ExclusionSource { DECLARED, LEARNED }
    }
```

- [ ] **Step 2.2: Create `CapabilitySpecializationStore.java`**

```java
package io.casehub.eidos.api;

import java.util.Map;

public interface CapabilitySpecializationStore {

    /**
     * Records one DECLINE for the given agent, capability, and domain.
     * Called by casehub-ledger/CBR per qualifying DECLINE attestation.
     * TTL is owned by the store implementation.
     */
    void recordDecline(String agentId, String tenancyId, String capabilityName, String domain);

    /**
     * Retracts all learned data for a (agentId, tenancyId, capabilityName) triple.
     * Clears all domain entries regardless of TTL. Emergency override.
     */
    void clearDeclines(String agentId, String tenancyId, String capabilityName);

    /**
     * Returns domain → count of unexpired DECLINE records for all domains with at least
     * 1 unexpired recorded decline. Empty map when none. Never null.
     */
    Map<String, Integer> learnedExclusions(String agentId, String tenancyId, String capabilityName);

    /**
     * Returns the count of unexpired DECLINE records for the given domain.
     * 0 when no unexpired records exist. Never negative.
     */
    int declineCount(String agentId, String tenancyId, String capabilityName, String domain);
}
```

- [ ] **Step 2.3: Create `NoOpCapabilitySpecializationStore.java`**

```java
package io.casehub.eidos.runtime.health;

import io.casehub.eidos.api.CapabilitySpecializationStore;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;

import java.util.Map;

@DefaultBean
@ApplicationScoped
public class NoOpCapabilitySpecializationStore implements CapabilitySpecializationStore {

    @Override
    public void recordDecline(String agentId, String tenancyId,
                               String capabilityName, String domain) {}

    @Override
    public void clearDeclines(String agentId, String tenancyId, String capabilityName) {}

    @Override
    public Map<String, Integer> learnedExclusions(String agentId, String tenancyId,
                                                   String capabilityName) {
        return Map.of();
    }

    @Override
    public int declineCount(String agentId, String tenancyId,
                             String capabilityName, String domain) {
        return 0;
    }
}
```

- [ ] **Step 2.4: Compile to confirm no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -DskipTests 2>&1 | tail -10
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 2.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add -A
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#55): add CapabilitySpecializationStore SPI and Excluded status variant

Refs #55"
```

---

## Task 3: Preferences infrastructure + update `DefaultCapabilityHealth`

**Files:**
- Modify: `runtime/pom.xml`
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/preferences/EidosPreferenceKeys.java`
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/preferences/ExcludeThresholdPreference.java`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/health/DefaultCapabilityHealth.java`
- Create: `runtime/src/test/java/io/casehub/eidos/runtime/health/DefaultCapabilityHealthExclusionTest.java`
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/health/DefaultCapabilityHealthDegradedTest.java`

- [ ] **Step 3.1: Write failing tests for exclusion probe steps (3 and 4)**

Create `runtime/src/test/java/io/casehub/eidos/runtime/health/DefaultCapabilityHealthExclusionTest.java`:

```java
package io.casehub.eidos.runtime.health;

import io.casehub.eidos.api.*;
import io.casehub.eidos.api.CapabilityHealth.CapabilityStatus;
import io.casehub.eidos.api.CapabilityHealth.ProbeContext;
import io.casehub.platform.api.preferences.PreferenceProvider;
import io.casehub.platform.api.preferences.Preferences;
import io.casehub.platform.api.preferences.SettingsScope;
import io.casehub.eidos.runtime.preferences.EidosPreferenceKeys;
import io.casehub.eidos.runtime.preferences.ExcludeThresholdPreference;
import jakarta.enterprise.inject.Instance;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.time.Instant;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class DefaultCapabilityHealthExclusionTest {

    // Stub state store (no operational degradation by default)
    static class StubStateStore implements AgentStateStore {
        @Override public void record(String a, String t, DegradationReason r, Instant e) {}
        @Override public Optional<DegradationReason> query(String a, String t) { return Optional.empty(); }
        @Override public void clear(String a, String t) {}
    }

    // Stub specialization store with explicit count control
    static class StubSpecializationStore implements CapabilitySpecializationStore {
        private final Map<String, Integer> counts = new HashMap<>();

        void setCount(String agentId, String tenancyId, String capability, String domain, int count) {
            counts.put(agentId + "|" + tenancyId + "|" + capability + "|" + domain, count);
        }

        @Override public void recordDecline(String a, String t, String c, String d) {}
        @Override public void clearDeclines(String a, String t, String c) {}
        @Override public Map<String, Integer> learnedExclusions(String a, String t, String c) { return Map.of(); }

        @Override
        public int declineCount(String agentId, String tenancyId, String capabilityName, String domain) {
            return counts.getOrDefault(agentId + "|" + tenancyId + "|" + capabilityName + "|" + domain, 0);
        }
    }

    @Mock
    @SuppressWarnings("unchecked")
    Instance<PreferenceProvider> preferenceProviderInstance;

    StubStateStore stateStore;
    StubSpecializationStore specializationStore;
    DefaultCapabilityHealth health;

    @BeforeEach
    void setUp() {
        stateStore = new StubStateStore();
        specializationStore = new StubSpecializationStore();
        // Default: no PreferenceProvider → isUnsatisfied() → threshold = 3 (key defaultValue)
        when(preferenceProviderInstance.isUnsatisfied()).thenReturn(true);
        health = new DefaultCapabilityHealth(0.3, stateStore, specializationStore, preferenceProviderInstance);
    }

    static AgentDescriptor agent(String agentId, AgentCapability... capabilities) {
        return AgentDescriptor.builder()
            .agentId(agentId).name("Agent").version("1.0").provider("anthropic")
            .modelFamily("claude").modelVersion("claude-3-7").slot("reviewer")
            .capabilities(List.of(capabilities)).tenancyId("default").build();
    }

    static AgentCapability capabilityWithExclusions(String name, Set<String> excludedDomains) {
        return AgentCapability.builder().name(name).qualityHint(0.9)
            .excludedDomains(excludedDomains).build();
    }

    static AgentCapability capability(String name) {
        return AgentCapability.builder().name(name).qualityHint(0.9).build();
    }

    // ── Step 3: declared exclusion ─────────────────────────────────────────────

    @Test
    void declared_excluded_domain_returns_excluded_status() {
        var descriptor = agent("agent1", capabilityWithExclusions("security-review", Set.of("rust", "go")));
        var status = health.probe(descriptor, "security-review", ProbeContext.of("rust"));
        assertThat(status).isInstanceOf(CapabilityStatus.Excluded.class);
        var excluded = (CapabilityStatus.Excluded) status;
        assertThat(excluded.domain()).isEqualTo("rust");
        assertThat(excluded.source()).isEqualTo(CapabilityStatus.ExclusionSource.DECLARED);
        assertThat(excluded.declineCount()).isEqualTo(0);
    }

    @Test
    void null_excluded_domains_does_not_throw_and_skips_step3() {
        var descriptor = agent("agent2", capability("code-review"));
        var status = health.probe(descriptor, "code-review", ProbeContext.of("rust"));
        assertThat(status).isInstanceOf(CapabilityStatus.Ready.class);
    }

    @Test
    void excluded_domain_not_in_set_returns_ready() {
        var descriptor = agent("agent3", capabilityWithExclusions("code-review", Set.of("rust")));
        var status = health.probe(descriptor, "code-review", ProbeContext.of("java"));
        assertThat(status).isInstanceOf(CapabilityStatus.Ready.class);
    }

    @Test
    void declared_exclusion_does_not_consult_specialization_store() {
        var descriptor = agent("agent4", capabilityWithExclusions("code-review", Set.of("rust")));
        health.probe(descriptor, "code-review", ProbeContext.of("rust"));
        // Store has no data for this agent — if store were consulted it'd return 0 and not Excluded
        // The fact it returns Excluded confirms store was NOT consulted (declared short-circuits)
        var status = health.probe(descriptor, "code-review", ProbeContext.of("rust"));
        assertThat(status).isInstanceOf(CapabilityStatus.Excluded.class);
        assertThat(((CapabilityStatus.Excluded) status).source())
            .isEqualTo(CapabilityStatus.ExclusionSource.DECLARED);
    }

    // ── Step 4: learned exclusion ──────────────────────────────────────────────

    @Test
    void learned_exclusion_at_default_threshold_returns_excluded() {
        specializationStore.setCount("agent5", "default", "code-review", "rust", 3);
        var descriptor = agent("agent5", capability("code-review"));
        var status = health.probe(descriptor, "code-review", ProbeContext.of("rust"));
        assertThat(status).isInstanceOf(CapabilityStatus.Excluded.class);
        var excluded = (CapabilityStatus.Excluded) status;
        assertThat(excluded.domain()).isEqualTo("rust");
        assertThat(excluded.source()).isEqualTo(CapabilityStatus.ExclusionSource.LEARNED);
        assertThat(excluded.declineCount()).isEqualTo(3);
    }

    @Test
    void learned_exclusion_below_threshold_continues_to_ready() {
        specializationStore.setCount("agent6", "default", "code-review", "rust", 2);
        var descriptor = agent("agent6", capability("code-review"));
        var status = health.probe(descriptor, "code-review", ProbeContext.of("rust"));
        assertThat(status).isInstanceOf(CapabilityStatus.Ready.class);
    }

    @Test
    void count_captured_in_single_call_matches_excluded_record() {
        specializationStore.setCount("agent7", "default", "code-review", "rust", 5);
        var descriptor = agent("agent7", capability("code-review"));
        var excluded = (CapabilityStatus.Excluded) health.probe(descriptor, "code-review", ProbeContext.of("rust"));
        assertThat(excluded.declineCount()).isEqualTo(5);
    }

    @Test
    void null_task_domain_skips_both_exclusion_checks() {
        var descriptor = agent("agent8", capabilityWithExclusions("code-review", Set.of("rust")));
        specializationStore.setCount("agent8", "default", "code-review", "rust", 10);
        var status = health.probe(descriptor, "code-review", ProbeContext.of(null));
        assertThat(status).isInstanceOf(CapabilityStatus.Ready.class);
    }

    @Test
    void threshold_resolved_from_preference_provider_when_satisfied() {
        var mockPreferences = mock(Preferences.class);
        var mockProvider = mock(PreferenceProvider.class);
        when(preferenceProviderInstance.isUnsatisfied()).thenReturn(false);
        when(preferenceProviderInstance.get()).thenReturn(mockProvider);
        when(mockProvider.resolve(any(SettingsScope.class))).thenReturn(mockPreferences);
        when(mockPreferences.getOrDefault(EidosPreferenceKeys.EXCLUDE_THRESHOLD))
            .thenReturn(new ExcludeThresholdPreference(2));

        specializationStore.setCount("agent9", "default", "code-review", "rust", 2);
        var descriptor = agent("agent9", capability("code-review"));
        var status = health.probe(descriptor, "code-review", ProbeContext.of("rust"));

        assertThat(status).isInstanceOf(CapabilityStatus.Excluded.class);
    }

    @Test
    void no_preference_provider_falls_back_to_default_threshold_of_3() {
        // preferenceProviderInstance.isUnsatisfied() returns true (setUp default)
        specializationStore.setCount("agent10", "default", "code-review", "rust", 2);
        var descriptor = agent("agent10", capability("code-review"));
        // 2 < 3 (default) → NOT excluded
        assertThat(health.probe(descriptor, "code-review", ProbeContext.of("rust")))
            .isInstanceOf(CapabilityStatus.Ready.class);

        specializationStore.setCount("agent10", "default", "code-review", "rust", 3);
        // 3 >= 3 (default) → excluded
        assertThat(health.probe(descriptor, "code-review", ProbeContext.of("rust")))
            .isInstanceOf(CapabilityStatus.Excluded.class);
    }
}
```

- [ ] **Step 3.2: Run the new tests — confirm failure (fields/types don't exist yet)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=DefaultCapabilityHealthExclusionTest 2>&1 | tail -20
```

Expected: `COMPILATION ERROR` — `CapabilitySpecializationStore`, `EidosPreferenceKeys`, `ExcludeThresholdPreference` not found; `DefaultCapabilityHealth` lacks the required constructor.

- [ ] **Step 3.3: Add `casehub-platform-api` to `runtime/pom.xml`**

In `runtime/pom.xml`, after the `casehub-ledger-api` dependency block, add:

```xml
    <!-- Platform preferences SPI — PreferenceProvider, PreferenceKey, SettingsScope -->
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-platform-api</artifactId>
    </dependency>
```

- [ ] **Step 3.4: Create `ExcludeThresholdPreference.java`**

```java
package io.casehub.eidos.runtime.preferences;

import io.casehub.platform.api.preferences.SingleValuePreference;

public record ExcludeThresholdPreference(int value) implements SingleValuePreference {}
```

- [ ] **Step 3.5: Create `EidosPreferenceKeys.java`**

```java
package io.casehub.eidos.runtime.preferences;

import io.casehub.platform.api.preferences.PreferenceKey;

public final class EidosPreferenceKeys {

    private EidosPreferenceKeys() {}

    /**
     * Minimum number of unexpired DECLINE records for a domain before eidos treats it as a
     * learned exclusion. Resolved per-tenancy via PreferenceProvider ancestor-chain walk.
     * Default: 3. Revisit once casehub-ledger/CBR integration produces real DECLINE data.
     */
    public static final PreferenceKey<ExcludeThresholdPreference> EXCLUDE_THRESHOLD =
        new PreferenceKey<>("casehub.eidos", "specialization.exclude-threshold",
                            new ExcludeThresholdPreference(3),
                            s -> new ExcludeThresholdPreference(Integer.parseInt(s)));
}
```

- [ ] **Step 3.6: Update `DefaultCapabilityHealth.java`**

Replace `runtime/src/main/java/io/casehub/eidos/runtime/health/DefaultCapabilityHealth.java` entirely:

```java
package io.casehub.eidos.runtime.health;

import io.casehub.eidos.api.*;
import io.casehub.eidos.api.CapabilityHealth.CapabilityStatus.ExclusionSource;
import io.casehub.eidos.runtime.preferences.EidosPreferenceKeys;
import io.casehub.platform.api.preferences.PreferenceProvider;
import io.casehub.platform.api.preferences.SettingsScope;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import org.eclipse.microprofile.config.inject.ConfigProperty;

@DefaultBean
@ApplicationScoped
public class DefaultCapabilityHealth implements CapabilityHealth {

    private final double weakThreshold;
    private final AgentStateStore stateStore;
    private final CapabilitySpecializationStore specializationStore;
    private final Instance<PreferenceProvider> preferenceProviderInstance;

    @Inject
    public DefaultCapabilityHealth(
            @ConfigProperty(name = "casehub.eidos.epistemic.weak-threshold", defaultValue = "0.3")
            final double weakThreshold,
            final AgentStateStore stateStore,
            final CapabilitySpecializationStore specializationStore,
            final Instance<PreferenceProvider> preferenceProviderInstance) {
        this.weakThreshold = weakThreshold;
        this.stateStore = stateStore;
        this.specializationStore = specializationStore;
        this.preferenceProviderInstance = preferenceProviderInstance;
    }

    @Override
    public CapabilityStatus probe(final AgentDescriptor descriptor, final String capabilityTag,
                                  final ProbeContext context) {
        // Step 1: operational degradation takes priority
        final var degraded = stateStore.query(descriptor.agentId(), descriptor.tenancyId());
        if (degraded.isPresent()) {
            return new CapabilityStatus.Degraded(degraded.get(), "recorded at dispatch time");
        }

        // Step 2: capability not declared → unavailable
        if (descriptor.capabilities() == null || descriptor.capabilities().isEmpty()) {
            return new CapabilityStatus.Unavailable("Capability '" + capabilityTag + "' not declared");
        }

        final var capability = descriptor.capabilities().stream()
                .filter(c -> c.name().equals(capabilityTag))
                .findFirst()
                .orElse(null);

        if (capability == null) {
            return new CapabilityStatus.Unavailable("Capability '" + capabilityTag + "' not declared");
        }

        // Step 3: declared exclusion
        if (context.taskDomain() != null
                && capability.excludedDomains() != null
                && capability.excludedDomains().contains(context.taskDomain())) {
            return new CapabilityStatus.Excluded(context.taskDomain(), ExclusionSource.DECLARED, 0);
        }

        // Step 4: learned exclusion
        if (context.taskDomain() != null) {
            final int count = specializationStore.declineCount(
                descriptor.agentId(), descriptor.tenancyId(), capabilityTag, context.taskDomain());
            if (count >= excludeThreshold(descriptor.tenancyId())) {
                return new CapabilityStatus.Excluded(context.taskDomain(), ExclusionSource.LEARNED, count);
            }
        }

        // Step 5: epistemic weakness
        if (context.taskDomain() != null && capability.epistemicDomains() != null) {
            final Double confidence = capability.epistemicDomains().get(context.taskDomain());
            if (confidence != null && confidence < weakThreshold) {
                return new CapabilityStatus.EpistemicallyWeak(context.taskDomain(), confidence);
            }
        }

        return new CapabilityStatus.Ready();
    }

    private int excludeThreshold(final String tenancyId) {
        if (preferenceProviderInstance.isUnsatisfied()) {
            return EidosPreferenceKeys.EXCLUDE_THRESHOLD.defaultValue().value();
        }
        return preferenceProviderInstance.get()
            .resolve(SettingsScope.of(tenancyId))
            .getOrDefault(EidosPreferenceKeys.EXCLUDE_THRESHOLD).value();
    }
}
```

- [ ] **Step 3.7: Add `Excluded` priority tests to `DefaultCapabilityHealthDegradedTest`**

Add to `DefaultCapabilityHealthDegradedTest.java` (the `setUp` needs updating because the constructor now has extra params — update it):

```java
    // Add import at top:
    // import io.casehub.eidos.api.CapabilitySpecializationStore;
    // import io.casehub.platform.api.preferences.PreferenceProvider;
    // import jakarta.enterprise.inject.Instance;

    static class NoOpSpecializationStore implements CapabilitySpecializationStore {
        @Override public void recordDecline(String a, String t, String c, String d) {}
        @Override public void clearDeclines(String a, String t, String c) {}
        @Override public java.util.Map<String, Integer> learnedExclusions(String a, String t, String c) { return java.util.Map.of(); }
        @Override public int declineCount(String a, String t, String c, String d) { return 0; }
    }

    @SuppressWarnings("unchecked")
    static Instance<PreferenceProvider> unsatisfiedProvider() {
        Instance<PreferenceProvider> mock = org.mockito.Mockito.mock(Instance.class);
        org.mockito.Mockito.when(mock.isUnsatisfied()).thenReturn(true);
        return mock;
    }

    // Update setUp():
    @BeforeEach
    void setUp() {
        stateStore = new StubStateStore();
        health = new DefaultCapabilityHealth(0.3, stateStore, new NoOpSpecializationStore(), unsatisfiedProvider());
    }

    // Add new test:
    @Test
    void degraded_state_takes_precedence_over_declared_exclusion() {
        stateStore.record("agent-x", "default", DegradationReason.RATE_LIMITED, Instant.now().plusSeconds(60));
        final var descriptor = agent("agent-x",
            AgentCapability.builder().name("code-review").qualityHint(0.9)
                .excludedDomains(java.util.Set.of("rust")).build());
        final var status = health.probe(descriptor, "code-review", ProbeContext.of("rust"));
        assertThat(status).isInstanceOf(CapabilityStatus.Degraded.class);
    }
```

Note: `DefaultCapabilityHealthDegradedTest` uses `@ExtendWith(MockitoExtension.class)` — add this annotation to the class.

- [ ] **Step 3.8: Run exclusion tests — confirm they now pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest="DefaultCapabilityHealthExclusionTest,DefaultCapabilityHealthDegradedTest,DefaultCapabilityHealthTest" \
  2>&1 | tail -30
```

Expected: All tests pass.

- [ ] **Step 3.9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add -A
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#55): wire exclusion probe steps 3+4 into DefaultCapabilityHealth

Refs #55"
```

---

## Task 4: `InMemoryCapabilitySpecializationStore`

**Files:**
- Create: `persistence-memory/src/main/java/io/casehub/eidos/memory/InMemoryCapabilitySpecializationStore.java`
- Create: `persistence-memory/src/test/java/io/casehub/eidos/memory/InMemoryCapabilitySpecializationStoreTest.java`

- [ ] **Step 4.1: Write failing tests**

Create `persistence-memory/src/test/java/io/casehub/eidos/memory/InMemoryCapabilitySpecializationStoreTest.java`:

```java
package io.casehub.eidos.memory;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.lang.reflect.Field;

import static org.assertj.core.api.Assertions.assertThat;

class InMemoryCapabilitySpecializationStoreTest {

    InMemoryCapabilitySpecializationStore store;

    @BeforeEach
    void setUp() throws Exception {
        store = new InMemoryCapabilitySpecializationStore();
        setTtl(store, 30);
    }

    static void setTtl(InMemoryCapabilitySpecializationStore s, int days) throws Exception {
        Field f = InMemoryCapabilitySpecializationStore.class.getDeclaredField("declineTtlDays");
        f.setAccessible(true);
        f.setInt(s, days);
    }

    @Test
    void single_decline_counted() {
        store.recordDecline("a1", "t1", "code-review", "rust");
        assertThat(store.declineCount("a1", "t1", "code-review", "rust")).isEqualTo(1);
    }

    @Test
    void multiple_declines_accumulate() {
        store.recordDecline("a1", "t1", "code-review", "rust");
        store.recordDecline("a1", "t1", "code-review", "rust");
        store.recordDecline("a1", "t1", "code-review", "rust");
        assertThat(store.declineCount("a1", "t1", "code-review", "rust")).isEqualTo(3);
    }

    @Test
    void learned_exclusions_returns_populated_domains_with_counts() {
        store.recordDecline("a1", "t1", "code-review", "rust");
        store.recordDecline("a1", "t1", "code-review", "rust");
        store.recordDecline("a1", "t1", "code-review", "go");

        var exclusions = store.learnedExclusions("a1", "t1", "code-review");
        assertThat(exclusions).containsEntry("rust", 2).containsEntry("go", 1);
    }

    @Test
    void empty_map_when_no_declines() {
        assertThat(store.learnedExclusions("a1", "t1", "code-review")).isEmpty();
    }

    @Test
    void zero_count_for_unknown_triple() {
        assertThat(store.declineCount("unknown", "t1", "code-review", "rust")).isEqualTo(0);
    }

    @Test
    void zero_count_for_unknown_domain() {
        store.recordDecline("a1", "t1", "code-review", "rust");
        assertThat(store.declineCount("a1", "t1", "code-review", "go")).isEqualTo(0);
    }

    @Test
    void clear_declines_removes_all_domains() {
        store.recordDecline("a1", "t1", "code-review", "rust");
        store.recordDecline("a1", "t1", "code-review", "go");
        store.clearDeclines("a1", "t1", "code-review");
        assertThat(store.declineCount("a1", "t1", "code-review", "rust")).isEqualTo(0);
        assertThat(store.declineCount("a1", "t1", "code-review", "go")).isEqualTo(0);
    }

    @Test
    void clear_code_review_does_not_remove_code_review_enhanced() {
        store.recordDecline("a1", "t1", "code-review", "rust");
        store.recordDecline("a1", "t1", "code-review-enhanced", "rust");
        store.clearDeclines("a1", "t1", "code-review");
        assertThat(store.declineCount("a1", "t1", "code-review", "rust")).isEqualTo(0);
        assertThat(store.declineCount("a1", "t1", "code-review-enhanced", "rust")).isEqualTo(1);
    }

    @Test
    void isolation_by_agent_id() {
        store.recordDecline("a1", "t1", "code-review", "rust");
        assertThat(store.declineCount("a2", "t1", "code-review", "rust")).isEqualTo(0);
    }

    @Test
    void isolation_by_tenancy_id() {
        store.recordDecline("a1", "t1", "code-review", "rust");
        assertThat(store.declineCount("a1", "t2", "code-review", "rust")).isEqualTo(0);
    }

    @Test
    void expired_entries_not_counted() throws Exception {
        setTtl(store, -1);   // TTL = -1 day → expires 1 day in the past → immediately stale
        store.recordDecline("a1", "t1", "code-review", "rust");
        assertThat(store.declineCount("a1", "t1", "code-review", "rust")).isEqualTo(0);
        assertThat(store.learnedExclusions("a1", "t1", "code-review")).isEmpty();
    }

    @Test
    void record_decline_purges_expired_entries_before_inserting() throws Exception {
        setTtl(store, -1);
        store.recordDecline("a1", "t1", "code-review", "rust");  // expires immediately
        store.recordDecline("a1", "t1", "code-review", "rust");  // expires immediately
        setTtl(store, 30);
        store.recordDecline("a1", "t1", "code-review", "rust");  // purges the two expired, adds one fresh
        assertThat(store.declineCount("a1", "t1", "code-review", "rust")).isEqualTo(1);
    }

    @Test
    void concurrent_record_decline_is_thread_safe() throws Exception {
        final int threads = 20;
        final Thread[] ts = new Thread[threads];
        for (int i = 0; i < threads; i++) {
            ts[i] = new Thread(() -> store.recordDecline("a1", "t1", "code-review", "rust"));
        }
        for (Thread t : ts) t.start();
        for (Thread t : ts) t.join();
        assertThat(store.declineCount("a1", "t1", "code-review", "rust")).isEqualTo(threads);
    }
}
```

- [ ] **Step 4.2: Run tests — confirm compile failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory \
  -Dtest=InMemoryCapabilitySpecializationStoreTest 2>&1 | tail -10
```

Expected: `COMPILATION ERROR — cannot find symbol: class InMemoryCapabilitySpecializationStore`.

- [ ] **Step 4.3: Implement `InMemoryCapabilitySpecializationStore.java`**

```java
package io.casehub.eidos.memory;

import io.casehub.eidos.api.CapabilitySpecializationStore;
import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import org.eclipse.microprofile.config.inject.ConfigProperty;

import java.time.Instant;
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentLinkedQueue;

@Alternative
@Priority(1)
@ApplicationScoped
public class InMemoryCapabilitySpecializationStore implements CapabilitySpecializationStore {

    @ConfigProperty(name = "casehub.eidos.specialization.decline-ttl-days", defaultValue = "30")
    int declineTtlDays;

    private final ConcurrentHashMap<AgentCapKey, ConcurrentHashMap<String, ConcurrentLinkedQueue<Instant>>>
        store = new ConcurrentHashMap<>();

    @Override
    public void recordDecline(final String agentId, final String tenancyId,
                               final String capabilityName, final String domain) {
        final var key = new AgentCapKey(agentId, tenancyId, capabilityName);
        final var domainQueues = store.computeIfAbsent(key, k -> new ConcurrentHashMap<>());
        final var queue = domainQueues.computeIfAbsent(domain, d -> new ConcurrentLinkedQueue<>());
        final Instant now = Instant.now();
        queue.removeIf(ts -> !now.isBefore(ts));
        queue.offer(now.plusDays(declineTtlDays));
    }

    @Override
    public void clearDeclines(final String agentId, final String tenancyId,
                               final String capabilityName) {
        store.remove(new AgentCapKey(agentId, tenancyId, capabilityName));
    }

    @Override
    public Map<String, Integer> learnedExclusions(final String agentId, final String tenancyId,
                                                   final String capabilityName) {
        final var domainQueues = store.get(new AgentCapKey(agentId, tenancyId, capabilityName));
        if (domainQueues == null) return Map.of();
        final Instant now = Instant.now();
        final var result = new HashMap<String, Integer>();
        domainQueues.forEach((domain, queue) -> {
            final int count = (int) queue.stream().filter(ts -> now.isBefore(ts)).count();
            if (count > 0) result.put(domain, count);
        });
        return Map.copyOf(result);
    }

    @Override
    public int declineCount(final String agentId, final String tenancyId,
                             final String capabilityName, final String domain) {
        final var domainQueues = store.get(new AgentCapKey(agentId, tenancyId, capabilityName));
        if (domainQueues == null) return 0;
        final var queue = domainQueues.get(domain);
        if (queue == null) return 0;
        final Instant now = Instant.now();
        return (int) queue.stream().filter(ts -> now.isBefore(ts)).count();
    }

    private record AgentCapKey(String agentId, String tenancyId, String capabilityName) {}
}
```

- [ ] **Step 4.4: Run all tests — confirm they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory 2>&1 | tail -20
```

Expected: All tests pass.

- [ ] **Step 4.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add -A
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#55): implement InMemoryCapabilitySpecializationStore with TTL

Refs #55"
```

---

## Task 5: Persistence — schema, entity, mapper

**Files:**
- Modify: `runtime/src/main/resources/db/eidos/migration/V1__initial_schema.sql`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentCapabilityEntity.java`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentDescriptorMapper.java`

- [ ] **Step 5.1: Add `excluded_domains` to `V1__initial_schema.sql`**

In the `CREATE TABLE agent_capability` block, add after `epistemic_domains TEXT`:

```sql
    excluded_domains   TEXT
```

The full `agent_capability` table now ends:
```sql
    epistemic_domains  TEXT,
    excluded_domains   TEXT
);
```

- [ ] **Step 5.2: Add `excludedDomains` field to `AgentCapabilityEntity.java`**

Add after the `epistemicDomains` field:

```java
    @Column(name = "excluded_domains", columnDefinition = "TEXT") String excludedDomains;
```

- [ ] **Step 5.3: Update `AgentDescriptorMapper.java`**

Update `toCapability()` to add the new field (using the builder from Task 1):

```java
private AgentCapability toCapability(AgentCapabilityEntity c) {
    return AgentCapability.builder()
        .name(c.name)
        .qualityHint(c.qualityHint)
        .latencyHintP50Ms(c.latencyHintP50Ms)
        .costHint(c.costHint)
        .inputTypes(readJson(c.inputTypes, new TypeReference<List<String>>() {}))
        .outputTypes(readJson(c.outputTypes, new TypeReference<List<String>>() {}))
        .tags(readJson(c.tags, new TypeReference<List<String>>() {}))
        .epistemicDomains(readJson(c.epistemicDomains, new TypeReference<Map<String, Double>>() {}))
        .excludedDomains(readJson(c.excludedDomains, new TypeReference<Set<String>>() {}))
        .build();
}
```

Update `toCapabilityEntity()` to persist the new field — add after `e.epistemicDomains = writeJson(c.epistemicDomains())`:

```java
        e.excludedDomains = writeJson(c.excludedDomains());
```

Add the `Set` import to `AgentDescriptorMapper.java`:

```java
import java.util.Set;
```

- [ ] **Step 5.4: Run JPA registry tests to confirm round-trip**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest="JpaAgentRegistryTest,JpaReactiveAgentRegistryTest" 2>&1 | tail -20
```

Expected: All tests pass (schema change applies cleanly; `excludedDomains = null` round-trips as null).

- [ ] **Step 5.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add -A
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#55): persist excludedDomains on agent_capability

Refs #55"
```

---

## Task 6: A2A_CARD renderer — add `excludedDomains`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosRenderPipeline.java`
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/renderer/EidosRenderPipelineTest.java`

- [ ] **Step 6.1: Write failing renderer tests for `excludedDomains`**

In `EidosRenderPipelineTest.java`, add after the existing `a2a_card_capability_numeric_fields_absent_when_null` test:

```java
    @Test
    void a2a_card_capability_includes_excluded_domains_when_populated() throws Exception {
        var desc = AgentDescriptor.builder()
            .agentId("renderer-excl").name("Agent").slot("reviewer").tenancyId("t1")
            .capabilities(List.of(AgentCapability.builder()
                .name("security-review")
                .excludedDomains(Set.of("rust", "go"))
                .build()))
            .build();
        var rendered = renderer.render(desc, AgentPromptContext.builder()
            .renderFormat(RenderFormat.A2A_CARD).build());
        var card = mapper.readTree(rendered.content());
        var caps = card.get("capabilities");
        assertThat(caps).isNotNull();
        var capNode = caps.get(0);
        assertThat(capNode.get("excludedDomains")).isNotNull();
        var excluded = new java.util.HashSet<String>();
        capNode.get("excludedDomains").forEach(n -> excluded.add(n.asText()));
        assertThat(excluded).containsExactlyInAnyOrder("rust", "go");
    }

    @Test
    void a2a_card_capability_excludes_domains_absent_when_null() throws Exception {
        var desc = AgentDescriptor.builder()
            .agentId("renderer-no-excl").name("Agent").slot("reviewer").tenancyId("t1")
            .capabilities(List.of(AgentCapability.builder().name("code-review").build()))
            .build();
        var rendered = renderer.render(desc, AgentPromptContext.builder()
            .renderFormat(RenderFormat.A2A_CARD).build());
        var card = mapper.readTree(rendered.content());
        var capNode = card.get("capabilities").get(0);
        assertThat(capNode.has("excludedDomains")).isFalse();
    }

    @Test
    void markdown_render_does_not_include_excluded_domains() throws Exception {
        var desc = AgentDescriptor.builder()
            .agentId("renderer-md-excl").name("Agent").slot("reviewer").tenancyId("t1")
            .capabilities(List.of(AgentCapability.builder()
                .name("code-review")
                .excludedDomains(Set.of("rust"))
                .build()))
            .build();
        var rendered = renderer.render(desc, AgentPromptContext.builder()
            .renderFormat(RenderFormat.MARKDOWN).build());
        assertThat(rendered.content()).doesNotContain("excludedDomains");
        assertThat(rendered.content()).doesNotContain("rust");
    }
```

- [ ] **Step 6.2: Run the new renderer tests — confirm failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=EidosRenderPipelineTest#a2a_card_capability_includes_excluded_domains_when_populated \
  2>&1 | tail -20
```

Expected: Test fails — `excludedDomains` field not in the JSON output yet.

- [ ] **Step 6.3: Add `excludedDomains` to the A2A_CARD capability node in `EidosRenderPipeline.java`**

In the capability loop (around line 626, after the `outputTypes` block), add:

```java
                if (cap.excludedDomains() != null && !cap.excludedDomains().isEmpty()) {
                    final ArrayNode arr = capNode.putArray("excludedDomains");
                    cap.excludedDomains().forEach(arr::add);
                }
```

Place this immediately after the `outputTypes` block and before the `description` block:

```java
                if (cap.outputTypes() != null && !cap.outputTypes().isEmpty()) {
                    final ArrayNode arr = capNode.putArray("outputTypes");
                    cap.outputTypes().forEach(arr::add);
                }
                // ← add here
                if (cap.excludedDomains() != null && !cap.excludedDomains().isEmpty()) {
                    final ArrayNode arr = capNode.putArray("excludedDomains");
                    cap.excludedDomains().forEach(arr::add);
                }
                final String desc = descriptionByName.get(cap.name());
```

- [ ] **Step 6.4: Run all renderer tests — confirm they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=EidosRenderPipelineTest 2>&1 | tail -20
```

Expected: All tests pass including the three new ones.

- [ ] **Step 6.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add -A
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#55): add excludedDomains to A2A_CARD capability rendering

Refs #55"
```

---

## Task 7: Full suite green + close issue

- [ ] **Step 7.1: Run the full test suite across all modules**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean test 2>&1 | tail -30
```

Expected: `BUILD SUCCESS`, all tests pass across api, runtime, persistence-memory, eval, examples.

- [ ] **Step 7.2: Commit if any last fixes were needed**

If any tests failed and required fixes, commit those now with `fix(eidos#55): <reason>`.

- [ ] **Step 7.3: Final commit — close issue reference**

```bash
git -C /Users/mdproctor/claude/casehub/eidos commit --allow-empty -m "chore(eidos#55): capability specialization metadata complete

Closes #55"
```

---

## Self-Review

**Spec coverage check:**

| Spec requirement | Covered by task |
|---|---|
| `AgentCapability.excludedDomains: Set<String>` | Task 1 |
| Builder for `AgentCapability` | Task 1 |
| Overlap validation (declared + epistemic) | Task 1 |
| Null guard on `excludedDomains` in probe step 3 | Task 3 (DefaultCapabilityHealth step 3 guard) |
| `CapabilityStatus.Excluded(domain, ExclusionSource, declineCount)` | Task 2 |
| `ExclusionSource { DECLARED, LEARNED }` | Task 2 |
| `CapabilitySpecializationStore` SPI (all 4 methods) | Task 2 |
| `NoOpCapabilitySpecializationStore @DefaultBean` | Task 2 |
| `InMemoryCapabilitySpecializationStore @Alternative @Priority(1)` | Task 4 |
| Store-owned TTL via `@ConfigProperty` (30-day default) | Task 4 |
| `AgentCapKey` compound record key (no string concatenation) | Task 4 |
| `ConcurrentLinkedQueue<Instant>` per domain | Task 4 |
| Purge on `recordDecline` | Task 4 |
| `clearDeclines` is O(1) | Task 4 |
| `EidosPreferenceKeys` + `ExcludeThresholdPreference` in `runtime.preferences` | Task 3 |
| `Instance<PreferenceProvider>` injection with `isUnsatisfied()` fallback | Task 3 |
| No `NoOpPreferenceProvider` | Task 3 (not created) |
| `casehub-platform-api` in runtime pom | Task 3 |
| V1 schema `excluded_domains TEXT` | Task 5 |
| `AgentCapabilityEntity.excludedDomains` | Task 5 |
| Mapper round-trip for `excludedDomains` | Task 5 |
| `excludedDomains` in A2A_CARD, absent from MARKDOWN/PROSE | Task 6 |
| All call sites migrated to builder | Task 1 |
| `DefaultReactiveCapabilityHealth` — no change needed (delegates to blocking impl) | — |
| Deferred: `ReactiveCapabilitySpecializationStore`, JPA store, `AgentQuery` domain filter | Filed as eidos#61, eidos#62 |

**Placeholder scan:** No TBD, TODO, or vague steps found.

**Type consistency:** `CapabilityStatus.Excluded` used consistently throughout. `ExclusionSource.DECLARED` and `ExclusionSource.LEARNED` consistent. `EidosPreferenceKeys.EXCLUDE_THRESHOLD` matches usage in `DefaultCapabilityHealth.excludeThreshold()`. `AgentCapKey` record used in both implementation and tests via `setTtl` reflection.

**Note on `DefaultReactiveCapabilityHealth`:** It delegates entirely to `DefaultCapabilityHealth` via `delegate.probe(...)` — no changes needed. The new probe steps execute through the blocking delegate.
