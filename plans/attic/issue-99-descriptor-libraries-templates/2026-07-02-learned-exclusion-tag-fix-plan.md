# Learned Exclusion Tag Fix — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix the learned exclusion tag mismatch in `DefaultCapabilityHealth.probe()` and extract subsumption resolution to a shared `CapabilityResolver` utility.

**Architecture:** Extract the private `findCapability()` logic from `DefaultCapabilityHealth` into a public `CapabilityResolver` utility in the api module. Fix probe() to use the declared capability name for store lookups. Clarify the store SPI contract via Javadoc. Update ARC42STORIES L4 to reflect the current probe order.

**Tech Stack:** Java 21, Quarkus 3.32.2, JUnit 5, AssertJ

## Global Constraints

- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- Test single module: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl <module>`
- Use `mvn` not `./mvnw`
- All commits reference issue: `Refs eidos#76` or `Closes eidos#76`
- Root Java package: `io.casehub.eidos.api` (api module), `io.casehub.eidos.runtime` (runtime module)
- IntelliJ MCP: use `mcp__intellij-index__*` for all code navigation, never `mcp__intellij__*`

---

### Task 1: CapabilityResolver utility with tests (api module)

**Files:**
- Create: `api/src/main/java/io/casehub/eidos/api/CapabilityResolver.java`
- Create: `api/src/test/java/io/casehub/eidos/api/CapabilityResolverTest.java`
- Test: `api/src/test/java/io/casehub/eidos/api/CapabilityResolverTest.java`

**Interfaces:**
- Consumes: `VocabularyRegistry.match(String vocabUri, String declaredValue, String requestedValue)` → `MatchDegree`; `AgentCapability.name()`, `AgentCapability.capabilityVocabulary()`; `MatchDegree.Exact`, `MatchDegree.Plugin(depth)`, `MatchDegree.Specialization(depth)`, `MatchDegree.None`
- Produces: `CapabilityResolver.match(AgentCapability, String, VocabularyRegistry)` → `MatchDegree`; `CapabilityResolver.resolve(List<AgentCapability>, String, VocabularyRegistry)` → `AgentCapability` (or null)

- [ ] **Step 1: Write the failing tests**

The api module has no Mockito — use a minimal stub VocabularyRegistry. The existing `TestCapabilityVocab` enum in `api/src/test/java/io/casehub/eidos/api/TestCapabilityVocab.java` provides a three-level hierarchy: `review` → `code-review` → `security-review`, plus `testing` → `unit-testing` / `integration-testing`.

Create `api/src/test/java/io/casehub/eidos/api/CapabilityResolverTest.java`:

```java
package io.casehub.eidos.api;

import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class CapabilityResolverTest {

    static final String VOCAB_URI = "urn:test:capabilities";
    static VocabularyRegistry registry;

    @BeforeAll
    static void setUp() {
        // Use CdiVocabularyRegistry directly — it's a plain Java class despite the CDI name.
        // The api test-jar already depends on it transitively for other tests.
        // If not available, we'll use a stub — see step 1 note.
        registry = createTestRegistry();
    }

    static AgentCapability grounded(String name) {
        return AgentCapability.builder().name(name)
            .capabilityVocabulary(VOCAB_URI).build();
    }

    static AgentCapability ungrounded(String name) {
        return AgentCapability.builder().name(name).build();
    }

    // --- match() tests ---

    @Test
    void match_exact_name_returns_exact() {
        var cap = grounded("code-review");
        var result = CapabilityResolver.match(cap, "code-review", registry);
        assertThat(result).isInstanceOf(MatchDegree.Exact.class);
    }

    @Test
    void match_ungrounded_non_exact_returns_none() {
        var cap = ungrounded("code-review");
        var result = CapabilityResolver.match(cap, "security-review", registry);
        assertThat(result).isInstanceOf(MatchDegree.None.class);
    }

    @Test
    void match_grounded_parent_returns_plugin() {
        // code-review is parent of security-review → Plugin
        var cap = grounded("code-review");
        var result = CapabilityResolver.match(cap, "security-review", registry);
        assertThat(result).isInstanceOf(MatchDegree.Plugin.class);
        assertThat(((MatchDegree.Plugin) result).depth()).isEqualTo(1);
    }

    @Test
    void match_grounded_child_returns_specialization() {
        // security-review is child of code-review → Specialization
        var cap = grounded("security-review");
        var result = CapabilityResolver.match(cap, "code-review", registry);
        assertThat(result).isInstanceOf(MatchDegree.Specialization.class);
        assertThat(((MatchDegree.Specialization) result).depth()).isEqualTo(1);
    }

    @Test
    void match_grounded_unrelated_returns_none() {
        var cap = grounded("code-review");
        var result = CapabilityResolver.match(cap, "testing", registry);
        assertThat(result).isInstanceOf(MatchDegree.None.class);
    }

    // --- resolve() tests ---

    @Test
    void resolve_exact_match_preferred_over_subsumption() {
        var caps = List.of(grounded("code-review"), grounded("security-review"));
        var result = CapabilityResolver.resolve(caps, "security-review", registry);
        assertThat(result).isNotNull();
        assertThat(result.name()).isEqualTo("security-review");
    }

    @Test
    void resolve_closest_depth_wins() {
        // sast-review: code-review is depth 2, security-review is depth 1
        // security-review should win
        var caps = List.of(grounded("code-review"), grounded("security-review"));
        // Need a sast-review term — TestCapabilityVocab doesn't have one.
        // Use "security-review" queried from "review" perspective instead:
        // review → code-review (depth 1) vs review → security-review (depth 2 via code-review)
        // Actually: query for "unit-testing", list has "testing" (depth 1) and "review" (no match)
        var caps2 = List.of(grounded("review"), grounded("testing"));
        var result = CapabilityResolver.resolve(caps2, "unit-testing", registry);
        assertThat(result).isNotNull();
        assertThat(result.name()).isEqualTo("testing"); // depth 1 beats review (no match)
    }

    @Test
    void resolve_ungrounded_exact_only() {
        var caps = List.of(ungrounded("code-review"));
        assertThat(CapabilityResolver.resolve(caps, "security-review", registry)).isNull();
        assertThat(CapabilityResolver.resolve(caps, "code-review", registry)).isNotNull();
    }

    @Test
    void resolve_null_capabilities_returns_null() {
        assertThat(CapabilityResolver.resolve(null, "code-review", registry)).isNull();
    }

    @Test
    void resolve_empty_capabilities_returns_null() {
        assertThat(CapabilityResolver.resolve(List.of(), "code-review", registry)).isNull();
    }

    @Test
    void resolve_first_in_list_wins_at_equal_depth() {
        // Both are depth 1 from "review": code-review and design-review
        var caps = List.of(grounded("code-review"), grounded("design-review"));
        // Query for "review" — both are Specialization(1)
        var result = CapabilityResolver.resolve(caps, "review", registry);
        assertThat(result).isNotNull();
        assertThat(result.name()).isEqualTo("code-review"); // first in list
    }

    /**
     * Creates a test VocabularyRegistry with TestCapabilityVocab registered.
     * Uses CdiVocabularyRegistry if available, otherwise a minimal stub.
     */
    private static VocabularyRegistry createTestRegistry() {
        try {
            var clazz = Class.forName("io.casehub.eidos.runtime.vocabulary.CdiVocabularyRegistry");
            var registry = (VocabularyRegistry) clazz.getDeclaredConstructor().newInstance();
            registry.register(TestCapabilityVocab.class);
            return registry;
        } catch (Exception e) {
            throw new RuntimeException("CdiVocabularyRegistry not on test classpath — "
                + "add runtime test-jar dependency to api module or use a stub", e);
        }
    }
}
```

**Note:** The `createTestRegistry()` method uses reflection to instantiate `CdiVocabularyRegistry` since the api module doesn't depend on runtime. If this fails at test time, add a `<dependency>` on `casehub-eidos:test-jar:test` to api's pom.xml, OR write a minimal stub that delegates to `VocabularyTerm.specializes()` for the hierarchy walk. The existing `DefaultCapabilityHealthTest` (a `@QuarkusTest` in runtime) handles this via CDI injection — but api tests are plain JUnit.

**The practical path:** Since `CdiVocabularyRegistry` has CDI annotations but its `register()` and `match()` methods are pure Java logic, instantiating it directly works. Check whether the runtime module's test-jar is already on api's test classpath. If not, add it.

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=CapabilityResolverTest`
Expected: Compilation failure — `CapabilityResolver` class does not exist yet.

- [ ] **Step 3: Write CapabilityResolver implementation**

Create `api/src/main/java/io/casehub/eidos/api/CapabilityResolver.java`:

```java
package io.casehub.eidos.api;

import java.util.List;

public final class CapabilityResolver {

    private CapabilityResolver() {}

    public static MatchDegree match(final AgentCapability capability,
                                     final String capabilityTag,
                                     final VocabularyRegistry registry) {
        if (capability.name().equals(capabilityTag)) {
            return new MatchDegree.Exact();
        }

        if (capability.capabilityVocabulary() == null
                || capability.capabilityVocabulary().isBlank()) {
            return new MatchDegree.None();
        }

        return registry.match(
            capability.capabilityVocabulary(),
            capability.name(),
            capabilityTag
        );
    }

    public static AgentCapability resolve(final List<AgentCapability> capabilities,
                                           final String capabilityTag,
                                           final VocabularyRegistry registry) {
        if (capabilities == null || capabilities.isEmpty()) {
            return null;
        }

        AgentCapability bestMatch = null;
        int bestDepth = Integer.MAX_VALUE;

        for (final var capability : capabilities) {
            final MatchDegree degree = match(capability, capabilityTag, registry);

            if (degree instanceof MatchDegree.Exact) {
                return capability;
            } else if (degree instanceof MatchDegree.Plugin plugin) {
                if (plugin.depth() < bestDepth) {
                    bestMatch = capability;
                    bestDepth = plugin.depth();
                }
            } else if (degree instanceof MatchDegree.Specialization specialization) {
                if (specialization.depth() < bestDepth) {
                    bestMatch = capability;
                    bestDepth = specialization.depth();
                }
            }
        }

        return bestMatch;
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=CapabilityResolverTest`
Expected: All tests PASS.

If `createTestRegistry()` fails because `CdiVocabularyRegistry` is not on the api test classpath, resolve by adding the runtime test-jar dependency to `api/pom.xml`:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-eidos</artifactId>
    <version>${project.version}</version>
    <type>test-jar</type>
    <scope>test</scope>
</dependency>
```

OR (preferred if the above creates a circular dependency): instantiate `CdiVocabularyRegistry` via a stub. The stub only needs `match()` — delegate to the vocabulary term hierarchy by resolving both terms and walking `specializes()`.

- [ ] **Step 5: Commit**

```bash
git add api/src/main/java/io/casehub/eidos/api/CapabilityResolver.java api/src/test/java/io/casehub/eidos/api/CapabilityResolverTest.java
git commit -m "feat(eidos#76): add CapabilityResolver utility — shared subsumption resolution"
```

---

### Task 2: Fix probe() and wire CapabilityResolver into DefaultCapabilityHealth

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/health/DefaultCapabilityHealth.java`
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/health/DefaultCapabilityHealthExclusionTest.java` (add subsumption-aware learned exclusion tests)
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/health/DefaultCapabilityHealthTest.java` (verify existing subsumption tests still pass)

**Interfaces:**
- Consumes: `CapabilityResolver.resolve(List<AgentCapability>, String, VocabularyRegistry)` → `AgentCapability` (from Task 1)
- Produces: Fixed `probe()` method — store lookups use `capability.name()` instead of `capabilityTag`

- [ ] **Step 1: Write the failing tests**

Add four tests to `DefaultCapabilityHealthExclusionTest.java`. These tests use the existing `StubSpecializationStore` and mock `VocabularyRegistry`.

```java
@Test
void learned_exclusion_uses_declared_name_under_subsumption() {
    // Agent declares "security-code-review" grounded. Probe for "code-review" (parent).
    // DECLINE signals recorded against "security-code-review" (declared name).
    // Mock vocab: code-review → security-code-review is Specialization(1)
    var cap = AgentCapability.builder().name("security-code-review")
        .capabilityVocabulary("urn:test:cap").qualityHint(0.9).build();
    var descriptor = agent("agent-sub1", cap);

    // No exact match for "code-review", so findCapability will try subsumption
    when(mockVocabRegistry.match("urn:test:cap", "security-code-review", "code-review"))
        .thenReturn(new MatchDegree.Specialization(1));

    // Record declines against declared name "security-code-review"
    specializationStore.setCount("agent-sub1", "default", "security-code-review",
        "rust", SpecializationSignal.DECLINE, 3);

    var status = health.probe(descriptor, "code-review", ProbeContext.of("rust"));
    assertThat(status).isInstanceOf(CapabilityStatus.Excluded.class);
    var excluded = (CapabilityStatus.Excluded) status;
    assertThat(excluded.source()).isEqualTo(CapabilityStatus.ExclusionSource.LEARNED);
    assertThat(excluded.declineCount()).isEqualTo(3);
}

@Test
void learned_exclusion_invisible_under_wrong_key() {
    // Same setup, but DECLINE recorded against "code-review" (query tag, wrong key)
    var cap = AgentCapability.builder().name("security-code-review")
        .capabilityVocabulary("urn:test:cap").qualityHint(0.9).build();
    var descriptor = agent("agent-sub2", cap);

    when(mockVocabRegistry.match("urn:test:cap", "security-code-review", "code-review"))
        .thenReturn(new MatchDegree.Specialization(1));

    // Record against query tag — wrong key
    specializationStore.setCount("agent-sub2", "default", "code-review",
        "rust", SpecializationSignal.DECLINE, 3);

    var status = health.probe(descriptor, "code-review", ProbeContext.of("rust"));
    // Should be Ready — declines under wrong key are invisible
    assertThat(status).isInstanceOf(CapabilityStatus.Ready.class);
}

@Test
void learned_exclusion_under_plugin_direction() {
    // Agent declares "code-review" (general, grounded). Probe for "security-code-review" (child).
    var cap = AgentCapability.builder().name("code-review")
        .capabilityVocabulary("urn:test:cap").qualityHint(0.9).build();
    var descriptor = agent("agent-sub3", cap);

    when(mockVocabRegistry.match("urn:test:cap", "code-review", "security-code-review"))
        .thenReturn(new MatchDegree.Plugin(1));

    // Record declines against declared name "code-review"
    specializationStore.setCount("agent-sub3", "default", "code-review",
        "rust", SpecializationSignal.DECLINE, 3);

    var status = health.probe(descriptor, "security-code-review", ProbeContext.of("rust"));
    assertThat(status).isInstanceOf(CapabilityStatus.Excluded.class);
    var excluded = (CapabilityStatus.Excluded) status;
    assertThat(excluded.source()).isEqualTo(CapabilityStatus.ExclusionSource.LEARNED);
}

@Test
void learned_exclusion_exact_match_regression() {
    // Exact match: declared and query tag are the same — regression guard
    var descriptor = agent("agent-sub4", capability("code-review"));

    specializationStore.setCount("agent-sub4", "default", "code-review",
        "rust", SpecializationSignal.DECLINE, 3);

    var status = health.probe(descriptor, "code-review", ProbeContext.of("rust"));
    assertThat(status).isInstanceOf(CapabilityStatus.Excluded.class);
    assertThat(((CapabilityStatus.Excluded) status).declineCount()).isEqualTo(3);
}
```

- [ ] **Step 2: Run tests to verify the subsumption tests fail (exact match test should pass)**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=DefaultCapabilityHealthExclusionTest`

Expected:
- `learned_exclusion_uses_declared_name_under_subsumption` — FAIL (probe uses `capabilityTag` not `capability.name()`)
- `learned_exclusion_invisible_under_wrong_key` — passes already (wrong key returns 0, which is < threshold)
- `learned_exclusion_under_plugin_direction` — FAIL (same bug, Plugin direction)
- `learned_exclusion_exact_match_regression` — PASS (exact match means `capabilityTag == capability.name()`)

- [ ] **Step 3: Fix DefaultCapabilityHealth — replace findCapability with CapabilityResolver and fix line 72**

In `DefaultCapabilityHealth.java`:

1. Replace the private `findCapability()` method with a call to `CapabilityResolver.resolve()`.
2. Change line 72 from `capabilityTag` to `capability.name()`.

The `probe()` method becomes:

```java
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

    final var capability = CapabilityResolver.resolve(
        descriptor.capabilities(), capabilityTag, vocabularyRegistry);

    if (capability == null) {
        return new CapabilityStatus.Unavailable("Capability '" + capabilityTag + "' not declared");
    }

    // Step 3: declared exclusion
    if (context.taskDomain() != null
            && capability.excludedDomains() != null
            && capability.excludedDomains().contains(context.taskDomain())) {
        return new CapabilityStatus.Excluded(context.taskDomain(), ExclusionSource.DECLARED, 0);
    }

    // Step 4: learned exclusion — use declared capability name, not query tag
    if (context.taskDomain() != null) {
        final int count = specializationStore.count(
            descriptor.agentId(), descriptor.tenancyId(), capability.name(),
            context.taskDomain(), SpecializationSignal.DECLINE);
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
```

Delete the private `findCapability()` method entirely. Add `import io.casehub.eidos.api.CapabilityResolver;` at the top.

- [ ] **Step 4: Run all exclusion tests and existing health tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="DefaultCapabilityHealthExclusionTest,DefaultCapabilityHealthTest,DefaultCapabilityHealthDegradedTest"`
Expected: All PASS.

- [ ] **Step 5: Run full module test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: All PASS — no regressions.

- [ ] **Step 6: Commit**

```bash
git add runtime/src/main/java/io/casehub/eidos/runtime/health/DefaultCapabilityHealth.java runtime/src/test/java/io/casehub/eidos/runtime/health/DefaultCapabilityHealthExclusionTest.java
git commit -m "fix(eidos#76): probe() uses declared capability name for learned exclusion lookups"
```

---

### Task 3: Clarify store SPI Javadoc + update ARC42STORIES L4 + integration test

**Files:**
- Modify: `api/src/main/java/io/casehub/eidos/api/CapabilitySpecializationStore.java` (Javadoc updates)
- Modify: `ARC42STORIES.MD` (L4 probe order)
- Create: `examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/LearnedExclusionSubsumptionTest.java`

**Interfaces:**
- Consumes: `CapabilityResolver.resolve()` (from Task 1), `CapabilityHealth.probe()` (from Task 2), `CapabilitySpecializationStore.record()`, `VocabularyRegistry`, `AgentRegistry`, `CasehubCapabilityTerm` vocab
- Produces: Updated SPI contract, updated architecture docs, end-to-end integration test

- [ ] **Step 1: Update CapabilitySpecializationStore Javadoc**

In `api/src/main/java/io/casehub/eidos/api/CapabilitySpecializationStore.java`, update all four method Javadocs:

```java
public interface CapabilitySpecializationStore {

    /**
     * Records one signal event for the given agent, capability, and domain.
     * TTL is owned by the store implementation — per-signal TTL is supported.
     *
     * <p>{@code capabilityName} must be the agent's declared capability name
     * (as returned by {@link AgentCapability#name()}), not a query/lookup term.
     * When the caller has a query tag instead, use
     * {@link CapabilityResolver#resolve(java.util.List, String, VocabularyRegistry)}
     * to obtain the declared capability first.
     */
    void record(String agentId, String tenancyId, String capabilityName,
                String domain, SpecializationSignal signal);

    /**
     * Retracts all learned data of the given signal type for an
     * (agentId, tenancyId, capabilityName) triple.
     * Clears all domain entries regardless of TTL.
     *
     * <p>{@code capabilityName} must be the agent's declared capability name —
     * see {@link #record} for details.
     */
    void clear(String agentId, String tenancyId, String capabilityName,
               SpecializationSignal signal);

    /**
     * Returns domain to count of unexpired records for the given signal type,
     * for all domains with at least one unexpired record.
     * Empty map when none. Never null.
     *
     * <p>{@code capabilityName} must be the agent's declared capability name —
     * see {@link #record} for details.
     */
    Map<String, Integer> learned(String agentId, String tenancyId,
                                 String capabilityName, SpecializationSignal signal);

    /**
     * Returns the count of unexpired records for the given signal type and domain.
     * 0 when no unexpired records exist. Never negative.
     *
     * <p>{@code capabilityName} must be the agent's declared capability name —
     * see {@link #record} for details.
     */
    int count(String agentId, String tenancyId, String capabilityName,
              String domain, SpecializationSignal signal);
}
```

- [ ] **Step 2: Update ARC42STORIES L4**

In `ARC42STORIES.MD`, find the L4 section (around line 838). Update the "What it adds" content:

**Line 852** — change the "After:" line to include all five status variants:
```
**After:** `DefaultCapabilityHealth` — fail-fast probe (Degraded → Unavailable → Excluded(DECLARED) → Excluded(LEARNED) → EpistemicallyWeak → Ready); `AgentStateStore` SPI records runtime degradation with TTL; `NoOpAgentStateStore @DefaultBean` preserves backward behaviour.
```

**Lines 856-859** — replace the bullet list with the full probe order:
```
- **Fail-fast probe order** — `AgentStateStore.query()` runs first (Degraded); capability resolution via `CapabilityResolver.resolve()` second (Unavailable if no match); declared `excludedDomains` third (Excluded/DECLARED); learned exclusion via `CapabilitySpecializationStore.count()` fourth (Excluded/LEARNED); epistemic domain confidence fifth (EpistemicallyWeak); else Ready
```

**Line 859** — change "Four `CapabilityStatus` variants" to:
```
- **Five `CapabilityStatus` variants** — `Ready`, `Unavailable`, `Degraded(reason, message)`, `EpistemicallyWeak(domain, confidence)`, `Excluded(domain, ExclusionSource, declineCount)`; sealed hierarchy; exhaustive switch at call sites
```

**Key files section** (around line 872) — add CapabilityResolver:
```
- `api/src/main/java/io/casehub/eidos/api/CapabilityResolver.java` — static utility; shared subsumption resolution for probe and recording paths; `resolve()` returns best-matching `AgentCapability` from a list; `match()` returns `MatchDegree` for a single capability
```

- [ ] **Step 3: Write the integration test**

Create `examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/LearnedExclusionSubsumptionTest.java`:

```java
package io.casehub.eidos.examples;

import io.casehub.eidos.api.*;
import io.casehub.eidos.api.CapabilityHealth.CapabilityStatus;
import io.casehub.eidos.api.CapabilityHealth.ProbeContext;
import io.casehub.eidos.vocab.CasehubCapabilityTerm;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class LearnedExclusionSubsumptionTest {

    @Inject AgentRegistry registry;
    @Inject CapabilityHealth capabilityHealth;
    @Inject CapabilitySpecializationStore specializationStore;
    @Inject VocabularyRegistry vocabularyRegistry;

    static AgentDescriptor securityReviewer(String tenancyId) {
        return AgentDescriptor.builder()
            .agentId("security-reviewer-learned")
            .tenancyId(tenancyId)
            .name("Security Code Reviewer")
            .slot("reviewer")
            .capabilities(List.of(
                AgentCapability.builder()
                    .name("security-code-review")
                    .capabilityVocabulary(CasehubCapabilityTerm.URI)
                    .build()))
            .disposition(AgentDisposition.builder().build())
            .build();
    }

    @Test
    void probe_subsumption_with_learned_exclusion_uses_declared_name() {
        var tenancyId = "tenant-learned-exclusion-sub";
        var agent = securityReviewer(tenancyId);
        registry.register(agent);

        // Resolve declared name using CapabilityResolver (as engine would)
        var resolved = CapabilityResolver.resolve(
            agent.capabilities(), "code-review", vocabularyRegistry);
        assertThat(resolved).isNotNull();
        assertThat(resolved.name()).isEqualTo("security-code-review");

        // Record 3 DECLINE signals against the declared capability name
        for (int i = 0; i < 3; i++) {
            specializationStore.record(
                agent.agentId(), tenancyId,
                resolved.name(), "rust",
                SpecializationSignal.DECLINE);
        }

        // Probe via subsumption query (code-review → matches security-code-review)
        var status = capabilityHealth.probe(
            agent, "code-review", ProbeContext.of("rust"));

        // Should be Excluded(LEARNED) — signals recorded under declared name are found
        assertThat(status).isInstanceOf(CapabilityStatus.Excluded.class);
        var excluded = (CapabilityStatus.Excluded) status;
        assertThat(excluded.source()).isEqualTo(CapabilityStatus.ExclusionSource.LEARNED);
        assertThat(excluded.declineCount()).isGreaterThanOrEqualTo(3);
    }

    @Test
    void probe_exact_match_with_learned_exclusion_still_works() {
        var tenancyId = "tenant-learned-exclusion-exact";
        var agent = securityReviewer(tenancyId);
        registry.register(agent);

        // Record 3 DECLINEs directly against declared name
        for (int i = 0; i < 3; i++) {
            specializationStore.record(
                agent.agentId(), tenancyId,
                "security-code-review", "java",
                SpecializationSignal.DECLINE);
        }

        // Probe with exact name
        var status = capabilityHealth.probe(
            agent, "security-code-review", ProbeContext.of("java"));

        assertThat(status).isInstanceOf(CapabilityStatus.Excluded.class);
        assertThat(((CapabilityStatus.Excluded) status).source())
            .isEqualTo(CapabilityStatus.ExclusionSource.LEARNED);
    }
}
```

- [ ] **Step 4: Run integration test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl examples/agent-scenarios -Dtest=LearnedExclusionSubsumptionTest`
Expected: All PASS.

- [ ] **Step 5: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS across all modules.

- [ ] **Step 6: Commit**

```bash
git add api/src/main/java/io/casehub/eidos/api/CapabilitySpecializationStore.java ARC42STORIES.MD examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/LearnedExclusionSubsumptionTest.java
git commit -m "docs(eidos#76): clarify store SPI contract, update L4 probe order, add integration test

Closes eidos#76"
```
