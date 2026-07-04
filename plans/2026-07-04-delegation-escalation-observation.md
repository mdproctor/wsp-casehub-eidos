# Delegation and Escalation Observation Semantics — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use hortora:subagent-driven-development (recommended) or hortora:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add delegation and escalation compliance dimensions to the behavioral contracts framework — ComplianceDimension constants, VocabularyTerm.impliesSupervision(), BehavioralExpectations.escalationExpected(), and vocabulary overrides.

**Architecture:** Pure additions to existing framework. Two new ComplianceDimension constants, one new default method on VocabularyTerm, one new static method (two overloads) on BehavioralExpectations, and impliesSupervision() overrides on ConscientiousnessTerm and DiscTerm. No probe pipeline changes — Step 6 already handles all VIOLATED signals generically.

**Tech Stack:** Java 21, Quarkus 3.32.2, JUnit 5, AssertJ

## Global Constraints

- Java 21 source on Java 26 JVM: `JAVA_HOME=$(/usr/libexec/java_home -v 26)`
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api` / `-pl vocab` / `-pl runtime`
- Use `mvn` not `./mvnw`
- All commits reference issue: `Refs casehubio/eidos#87`
- eidos-api is Tier 1 pure Java — no CDI, no Quarkus dependencies
- VocabularyTerm is an interface in eidos-api — default methods only

---

### Task 1: ComplianceDimension Constants + VocabularyTerm.impliesSupervision()

**Files:**
- Modify: `api/src/main/java/io/casehub/eidos/api/ComplianceDimension.java`
- Modify: `api/src/main/java/io/casehub/eidos/api/VocabularyTerm.java`

**Interfaces:**
- Produces: `ComplianceDimension.DELEGATION` (String constant `"delegation"`), `ComplianceDimension.ESCALATION` (String constant `"escalation"`), `VocabularyTerm.impliesSupervision()` (default method returning `boolean`)

- [ ] **Step 1: Add DELEGATION and ESCALATION constants to ComplianceDimension**

In `api/src/main/java/io/casehub/eidos/api/ComplianceDimension.java`, add after the `ATTESTATION_RATE` field:

```java
public static final String DELEGATION = "delegation";
public static final String ESCALATION = "escalation";
```

- [ ] **Step 2: Add impliesSupervision() default method to VocabularyTerm**

In `api/src/main/java/io/casehub/eidos/api/VocabularyTerm.java`, add after the `axisExactMatch` method:

```java
/**
 * Returns {@code true} if this term indicates the entity operates under
 * supervision and should escalate to a supervisor when encountering
 * uncertain or high-stakes decisions.
 *
 * <p>Primarily meaningful for {@link DispositionAxis#AUTONOMY AUTONOMY} axis
 * terms. Slot terms, capability terms, and other axis terms inherit the
 * default {@code false} — semantically correct and harmless if called in a
 * non-autonomy context.
 *
 * <p>Used by {@link BehavioralExpectations#escalationExpected} to determine
 * whether escalation compliance should be monitored for an agent.
 */
default boolean impliesSupervision() { return false; }
```

- [ ] **Step 3: Build api module to verify compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api`
Expected: All existing tests pass (new method has no callers yet).

- [ ] **Step 4: Commit**

```
git add api/src/main/java/io/casehub/eidos/api/ComplianceDimension.java \
       api/src/main/java/io/casehub/eidos/api/VocabularyTerm.java
git commit -m "feat(eidos#87): add DELEGATION/ESCALATION constants and VocabularyTerm.impliesSupervision()"
```

---

### Task 2: BehavioralExpectations.escalationExpected() + Tests

**Files:**
- Modify: `api/src/main/java/io/casehub/eidos/api/BehavioralExpectations.java`
- Modify: `api/src/test/java/io/casehub/eidos/api/BehavioralExpectationsTest.java`

**Interfaces:**
- Consumes: `VocabularyTerm.impliesSupervision()` from Task 1, `VocabularyRegistry.resolve(String, String)` → `Optional<? extends VocabularyTerm>`, `AgentDescriptor.vocabUriForAxis(DispositionAxis)` → `Optional<String>`
- Produces: `BehavioralExpectations.escalationExpected(AgentDisposition, String, VocabularyRegistry)` → `boolean`, `BehavioralExpectations.escalationExpected(AgentDescriptor, VocabularyRegistry)` → `boolean`

- [ ] **Step 1: Write failing tests for 3-param escalationExpected()**

In `api/src/test/java/io/casehub/eidos/api/BehavioralExpectationsTest.java`, add a test vocabulary enum and test methods. The test needs a minimal VocabularyTerm implementation and a mock VocabularyRegistry. Since eidos-api is pure Java (no Mockito), use a simple inline implementation:

```java
// --- Inner enum for testing ---
@VocabularyMetadata(uri = "urn:test:autonomy", name = "Test Autonomy", version = "1.0")
enum TestAutonomyTerm implements VocabularyTerm {
    SUPERVISED("supervised", "Supervised", true),
    BOUNDED("bounded", "Bounded", true),
    SELF_GOVERNING("self-governing", "Self-Governing", false);

    private final String value, label;
    private final boolean supervised;

    TestAutonomyTerm(String value, String label, boolean supervised) {
        this.value = value;
        this.label = label;
        this.supervised = supervised;
    }

    @Override public String value() { return value; }
    @Override public String label() { return label; }
    @Override public boolean impliesSupervision() { return supervised; }
}
```

For VocabularyRegistry, create a minimal inline implementation that resolves terms from TestAutonomyTerm:

```java
private static VocabularyRegistry testRegistry() {
    return new VocabularyRegistry() {
        @Override public <T extends Enum<T> & VocabularyTerm> void register(Class<T> vocab) {}
        @Override public boolean isRegistered(String vocabUri) {
            return "urn:test:autonomy".equals(vocabUri);
        }
        @Override public Optional<? extends VocabularyTerm> resolve(String vocabUri, String value) {
            if (!"urn:test:autonomy".equals(vocabUri)) return Optional.empty();
            for (TestAutonomyTerm t : TestAutonomyTerm.values()) {
                if (t.value().equals(value)) return Optional.of(t);
            }
            return Optional.empty();
        }
        @Override public List<? extends VocabularyTerm> allTerms(String vocabUri) { return List.of(); }
        @Override public Optional<String> equivalentValues(String f, String v, String t) { return Optional.empty(); }
        @Override public Optional<String> equivalentValues(String f, String v, String t, DispositionAxis a) { return Optional.empty(); }
        @Override public <T extends Enum<T> & VocabularyTerm> Optional<T> resolve(Class<T> vocab, String value) { return Optional.empty(); }
        @Override public <S extends Enum<S> & VocabularyTerm, T extends Enum<T> & VocabularyTerm> Optional<T> equivalentValues(S from, Class<T> targetVocab) { return Optional.empty(); }
        @Override public <S extends Enum<S> & VocabularyTerm, T extends Enum<T> & VocabularyTerm> Optional<T> equivalentValues(S from, Class<T> targetVocab, DispositionAxis axis) { return Optional.empty(); }
        @Override public Optional<VocabularyMetadata> vocabularyMetadata(String uri) { return Optional.empty(); }
        @Override public boolean subsumes(String vocabUri, String generalValue, String specificValue) { return false; }
        @Override public MatchDegree match(String vocabUri, String declaredValue, String requestedValue) { return MatchDegree.NONE; }
        @Override public List<? extends VocabularyTerm> ancestors(String vocabUri, String value) { return List.of(); }
        @Override public List<? extends VocabularyTerm> descendants(String vocabUri, String value) { return List.of(); }
        @Override public java.util.Map<String, java.util.Set<String>> expandForMatchingByVocabulary(String value) { return java.util.Map.of(); }
    };
}
```

Then the test methods:

```java
private static final String TEST_VOCAB_URI = "urn:test:autonomy";

@Test
void escalationExpected_true_for_supervised_term() {
    var disp = AgentDisposition.builder().autonomy("supervised").build();
    assertThat(BehavioralExpectations.escalationExpected(disp, TEST_VOCAB_URI, testRegistry())).isTrue();
}

@Test
void escalationExpected_true_for_bounded_term() {
    var disp = AgentDisposition.builder().autonomy("bounded").build();
    assertThat(BehavioralExpectations.escalationExpected(disp, TEST_VOCAB_URI, testRegistry())).isTrue();
}

@Test
void escalationExpected_false_for_self_governing_term() {
    var disp = AgentDisposition.builder().autonomy("self-governing").build();
    assertThat(BehavioralExpectations.escalationExpected(disp, TEST_VOCAB_URI, testRegistry())).isFalse();
}

@Test
void escalationExpected_false_when_null_disposition() {
    assertThat(BehavioralExpectations.escalationExpected((AgentDisposition) null, TEST_VOCAB_URI, testRegistry())).isFalse();
}

@Test
void escalationExpected_false_when_null_autonomy() {
    var disp = AgentDisposition.builder().build();
    assertThat(BehavioralExpectations.escalationExpected(disp, TEST_VOCAB_URI, testRegistry())).isFalse();
}

@Test
void escalationExpected_false_when_null_vocabUri() {
    var disp = AgentDisposition.builder().autonomy("supervised").build();
    assertThat(BehavioralExpectations.escalationExpected(disp, null, testRegistry())).isFalse();
}

@Test
void escalationExpected_false_when_null_registry() {
    var disp = AgentDisposition.builder().autonomy("supervised").build();
    assertThat(BehavioralExpectations.escalationExpected(disp, TEST_VOCAB_URI, null)).isFalse();
}

@Test
void escalationExpected_false_when_value_unresolvable() {
    var disp = AgentDisposition.builder().autonomy("unknown-value").build();
    assertThat(BehavioralExpectations.escalationExpected(disp, TEST_VOCAB_URI, testRegistry())).isFalse();
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=BehavioralExpectationsTest`
Expected: FAIL — `escalationExpected` method does not exist.

- [ ] **Step 3: Implement escalationExpected() — 3-param overload**

In `api/src/main/java/io/casehub/eidos/api/BehavioralExpectations.java`, add:

```java
public static boolean escalationExpected(final AgentDisposition disposition,
                                         final String autonomyVocabUri,
                                         final VocabularyRegistry registry) {
    if (disposition == null || disposition.autonomy() == null) return false;
    if (autonomyVocabUri == null || registry == null) return false;

    return registry.resolve(autonomyVocabUri, disposition.autonomy())
            .map(VocabularyTerm::impliesSupervision)
            .orElse(false);
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=BehavioralExpectationsTest`
Expected: All tests PASS.

- [ ] **Step 5: Write failing tests for convenience overload**

Add tests for `escalationExpected(AgentDescriptor, VocabularyRegistry)`:

```java
@Test
void escalationExpected_convenience_null_descriptor() {
    assertThat(BehavioralExpectations.escalationExpected((AgentDescriptor) null, testRegistry())).isFalse();
}

@Test
void escalationExpected_convenience_null_disposition() {
    var desc = AgentDescriptor.builder()
            .agentId("a1").name("Agent").slot("worker").tenancyId("t1")
            .build();
    assertThat(BehavioralExpectations.escalationExpected(desc, testRegistry())).isFalse();
}

@Test
void escalationExpected_convenience_no_vocab_uri() {
    var desc = AgentDescriptor.builder()
            .agentId("a1").name("Agent").slot("worker").tenancyId("t1")
            .disposition(AgentDisposition.builder().autonomy("supervised").build())
            .build();
    assertThat(BehavioralExpectations.escalationExpected(desc, testRegistry())).isFalse();
}

@Test
void escalationExpected_convenience_with_domain_vocab() {
    var desc = AgentDescriptor.builder()
            .agentId("a1").name("Agent").slot("worker").tenancyId("t1")
            .domainVocabulary(TEST_VOCAB_URI)
            .disposition(AgentDisposition.builder().autonomy("supervised").build())
            .build();
    assertThat(BehavioralExpectations.escalationExpected(desc, testRegistry())).isTrue();
}

@Test
void escalationExpected_convenience_autonomous_with_domain_vocab() {
    var desc = AgentDescriptor.builder()
            .agentId("a1").name("Agent").slot("worker").tenancyId("t1")
            .domainVocabulary(TEST_VOCAB_URI)
            .disposition(AgentDisposition.builder().autonomy("self-governing").build())
            .build();
    assertThat(BehavioralExpectations.escalationExpected(desc, testRegistry())).isFalse();
}
```

- [ ] **Step 6: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=BehavioralExpectationsTest`
Expected: FAIL — convenience overload does not exist.

- [ ] **Step 7: Implement convenience overload**

In `api/src/main/java/io/casehub/eidos/api/BehavioralExpectations.java`, add:

```java
public static boolean escalationExpected(final AgentDescriptor descriptor,
                                         final VocabularyRegistry registry) {
    if (descriptor == null || descriptor.disposition() == null) return false;
    return descriptor.vocabUriForAxis(DispositionAxis.AUTONOMY)
            .map(uri -> escalationExpected(descriptor.disposition(), uri, registry))
            .orElse(false);
}
```

Add the import at the top if not already present:

```java
import java.util.Optional;
```

- [ ] **Step 8: Run all tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=BehavioralExpectationsTest`
Expected: All tests PASS.

- [ ] **Step 9: Commit**

```
git add api/src/main/java/io/casehub/eidos/api/BehavioralExpectations.java \
       api/src/test/java/io/casehub/eidos/api/BehavioralExpectationsTest.java
git commit -m "feat(eidos#87): BehavioralExpectations.escalationExpected() with vocabulary-aware supervision detection"
```

---

### Task 3: Vocabulary Overrides — ConscientiousnessTerm + DiscTerm + Tests

**Files:**
- Modify: `vocab/src/main/java/io/casehub/eidos/vocab/ConscientiousnessTerm.java`
- Modify: `vocab/src/main/java/io/casehub/eidos/vocab/DiscTerm.java`
- Create: `vocab/src/test/java/io/casehub/eidos/vocab/ImpliesSupervisionTest.java`

**Interfaces:**
- Consumes: `VocabularyTerm.impliesSupervision()` from Task 1

- [ ] **Step 1: Write failing tests for ConscientiousnessTerm.impliesSupervision()**

Create `vocab/src/test/java/io/casehub/eidos/vocab/ImpliesSupervisionTest.java`:

```java
package io.casehub.eidos.vocab;

import io.casehub.eidos.api.DispositionAxis;
import io.casehub.eidos.api.VocabularyTerm;
import org.junit.jupiter.api.Test;

import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;

class ImpliesSupervisionTest {

    // --- ConscientiousnessTerm: AUTONOMY axis ---

    @Test
    void conscientiousness_directed_implies_supervision() {
        assertThat(ConscientiousnessTerm.DIRECTED.impliesSupervision()).isTrue();
    }

    @Test
    void conscientiousness_semi_autonomous_implies_supervision() {
        assertThat(ConscientiousnessTerm.SEMI_AUTONOMOUS.impliesSupervision()).isTrue();
    }

    @Test
    void conscientiousness_autonomous_does_not_imply_supervision() {
        assertThat(ConscientiousnessTerm.AUTONOMOUS.impliesSupervision()).isFalse();
    }

    // --- ConscientiousnessTerm: non-AUTONOMY axes default to false ---

    @Test
    void conscientiousness_rule_following_terms_do_not_imply_supervision() {
        assertThat(ConscientiousnessTerm.STRICT.impliesSupervision()).isFalse();
        assertThat(ConscientiousnessTerm.PRINCIPLED.impliesSupervision()).isFalse();
        assertThat(ConscientiousnessTerm.FLEXIBLE.impliesSupervision()).isFalse();
    }

    @Test
    void conscientiousness_risk_appetite_terms_do_not_imply_supervision() {
        assertThat(ConscientiousnessTerm.CONSERVATIVE.impliesSupervision()).isFalse();
        assertThat(ConscientiousnessTerm.MEASURED.impliesSupervision()).isFalse();
        assertThat(ConscientiousnessTerm.BOLD.impliesSupervision()).isFalse();
    }

    @Test
    void conscientiousness_social_orientation_terms_do_not_imply_supervision() {
        assertThat(ConscientiousnessTerm.COLLABORATIVE.impliesSupervision()).isFalse();
        assertThat(ConscientiousnessTerm.INDEPENDENT.impliesSupervision()).isFalse();
        assertThat(ConscientiousnessTerm.FACILITATIVE.impliesSupervision()).isFalse();
    }

    // --- DiscTerm ---

    @Test
    void disc_steadiness_implies_supervision() {
        assertThat(DiscTerm.STEADINESS.impliesSupervision()).isTrue();
    }

    @Test
    void disc_influence_implies_supervision() {
        assertThat(DiscTerm.INFLUENCE.impliesSupervision()).isTrue();
    }

    @Test
    void disc_conscientiousness_disc_implies_supervision() {
        assertThat(DiscTerm.CONSCIENTIOUSNESS_DISC.impliesSupervision()).isTrue();
    }

    @Test
    void disc_dominance_does_not_imply_supervision() {
        assertThat(DiscTerm.DOMINANCE.impliesSupervision()).isFalse();
    }

    // --- Cross-vocabulary consistency ---

    @Test
    void disc_autonomy_equivalents_have_consistent_supervision() {
        for (DiscTerm disc : DiscTerm.values()) {
            Optional<VocabularyTerm> equivalent =
                    disc.axisExactMatch(ConscientiousnessTerm.class, DispositionAxis.AUTONOMY);
            if (equivalent.isPresent()) {
                assertThat(disc.impliesSupervision())
                        .as("%s.impliesSupervision() should match its ConscientiousnessTerm AUTONOMY equivalent %s",
                                disc, equivalent.get())
                        .isEqualTo(equivalent.get().impliesSupervision());
            }
        }
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl vocab -Dtest=ImpliesSupervisionTest`
Expected: FAIL — ConscientiousnessTerm terms return default `false` for DIRECTED and SEMI_AUTONOMOUS.

- [ ] **Step 3: Add impliesSupervision() overrides to ConscientiousnessTerm**

In `vocab/src/main/java/io/casehub/eidos/vocab/ConscientiousnessTerm.java`, modify DIRECTED and SEMI_AUTONOMOUS enum constants to override `impliesSupervision()`. Change these two constants from simple constructor calls to anonymous class bodies:

Replace the DIRECTED constant:
```java
DIRECTED    ("directed",       "Directed Autonomy",
             "Follows explicit instructions",       List.of("instruction-following")) {
    @Override public boolean impliesSupervision() { return true; }
},
```

Replace the SEMI_AUTONOMOUS constant:
```java
SEMI_AUTONOMOUS("semi-autonomous", "Semi-Autonomous",
             "Acts within defined boundaries",      List.of("bounded-autonomy")) {
    @Override public boolean impliesSupervision() { return true; }
},
```

- [ ] **Step 4: Add impliesSupervision() overrides to DiscTerm**

In `vocab/src/main/java/io/casehub/eidos/vocab/DiscTerm.java`, add `@Override public boolean impliesSupervision() { return true; }` inside the anonymous class body of STEADINESS, INFLUENCE, and CONSCIENTIOUSNESS_DISC. Each already has an anonymous class body for `axisExactMatch`, so add the override after the existing `axisExactMatch` method.

For STEADINESS, add before the closing `}` of the enum constant body:
```java
@Override public boolean impliesSupervision() { return true; }
```

Same for INFLUENCE and CONSCIENTIOUSNESS_DISC.

DOMINANCE returns `false` by default — no override needed.

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl vocab -Dtest=ImpliesSupervisionTest`
Expected: All tests PASS, including the cross-vocabulary consistency test.

- [ ] **Step 6: Commit**

```
git add vocab/src/main/java/io/casehub/eidos/vocab/ConscientiousnessTerm.java \
       vocab/src/main/java/io/casehub/eidos/vocab/DiscTerm.java \
       vocab/src/test/java/io/casehub/eidos/vocab/ImpliesSupervisionTest.java
git commit -m "feat(eidos#87): impliesSupervision() overrides on ConscientiousnessTerm and DiscTerm"
```

---

### Task 4: Integration Test — Probe Pipeline with DELEGATION/ESCALATION Dimensions

**Files:**
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/health/DefaultCapabilityHealthTest.java`

**Interfaces:**
- Consumes: `ComplianceDimension.DELEGATION`, `ComplianceDimension.ESCALATION` from Task 1, `InMemoryBehavioralSignalStore` from persistence-memory module, `DefaultCapabilityHealth.probe()` from runtime

- [ ] **Step 1: Write integration tests for probe with DELEGATION and ESCALATION violations**

In `runtime/src/test/java/io/casehub/eidos/runtime/health/DefaultCapabilityHealthTest.java`, add two tests. The test class already has a `health` field and helper methods for creating agents. Use the existing `InMemoryBehavioralSignalStore` that the test setup injects.

First check the test class setup — it uses `@Inject` for `CapabilityHealth` and `VocabularyRegistry`. The tests need `BehavioralSignalStore` injected too. Check if it's already available; if not, add `@Inject BehavioralSignalStore signalStore;`.

Add these tests:

```java
@Test
void probe_returns_behavioral_violation_for_delegation_dimension() {
    var cap = AgentCapability.builder().name("code-review").build();
    var desc = agent("agent-1", cap);

    // Record 3 VIOLATED signals for delegation dimension (meets default threshold of 3)
    signalStore.record("agent-1", "tenant-1", "code-review",
            ComplianceDimension.DELEGATION, BehavioralSignal.VIOLATED);
    signalStore.record("agent-1", "tenant-1", "code-review",
            ComplianceDimension.DELEGATION, BehavioralSignal.VIOLATED);
    signalStore.record("agent-1", "tenant-1", "code-review",
            ComplianceDimension.DELEGATION, BehavioralSignal.VIOLATED);

    var status = health.probe(desc, "code-review", ProbeContext.of(null));

    assertThat(status).isInstanceOf(CapabilityStatus.BehavioralViolation.class);
    var violation = (CapabilityStatus.BehavioralViolation) status;
    assertThat(violation.violations()).containsEntry(ComplianceDimension.DELEGATION, 3);
    assertThat(violation.kind())
            .isEqualTo(CapabilityStatus.BehavioralViolation.ViolationKind.PER_DIMENSION);
}

@Test
void probe_returns_behavioral_violation_for_escalation_dimension() {
    var cap = AgentCapability.builder().name("code-review").build();
    var desc = agent("agent-2", cap);

    signalStore.record("agent-2", "tenant-1", "code-review",
            ComplianceDimension.ESCALATION, BehavioralSignal.VIOLATED);
    signalStore.record("agent-2", "tenant-1", "code-review",
            ComplianceDimension.ESCALATION, BehavioralSignal.VIOLATED);
    signalStore.record("agent-2", "tenant-1", "code-review",
            ComplianceDimension.ESCALATION, BehavioralSignal.VIOLATED);

    var status = health.probe(desc, "code-review", ProbeContext.of(null));

    assertThat(status).isInstanceOf(CapabilityStatus.BehavioralViolation.class);
    var violation = (CapabilityStatus.BehavioralViolation) status;
    assertThat(violation.violations()).containsEntry(ComplianceDimension.ESCALATION, 3);
    assertThat(violation.kind())
            .isEqualTo(CapabilityStatus.BehavioralViolation.ViolationKind.PER_DIMENSION);
}
```

Note: The `agent()` helper uses `"tenant-1"` as tenancyId — verify this matches the existing helper method. Adjust if different.

- [ ] **Step 2: Run tests to verify they pass**

These tests should pass immediately — Step 6 already counts all VIOLATED signals regardless of dimension key. The new constants are just strings.

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=DefaultCapabilityHealthTest`
Expected: All tests PASS (both old and new).

- [ ] **Step 3: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
Expected: All tests PASS across all modules.

- [ ] **Step 4: Commit**

```
git add runtime/src/test/java/io/casehub/eidos/runtime/health/DefaultCapabilityHealthTest.java
git commit -m "test(eidos#87): integration tests — probe pipeline with DELEGATION and ESCALATION violated signals"
```

---

### Task 5: Cross-Vocabulary Escalation Test + CLAUDE.md + ARC42STORIES.MD Updates

**Files:**
- Modify: `api/src/test/java/io/casehub/eidos/api/BehavioralExpectationsTest.java`
- Modify: `CLAUDE.md`
- Modify: `ARC42STORIES.MD` (in workspace — check routing via `design-repo` in `.meta`)

**Interfaces:**
- Consumes: `BehavioralExpectations.escalationExpected()` from Task 2, `DiscTerm` from Task 3

- [ ] **Step 1: Add cross-vocabulary escalation test using DISC vocabulary**

The api module test can't import DiscTerm (different module). Instead, this test belongs in the vocab module alongside the ImpliesSupervisionTest. Add to `vocab/src/test/java/io/casehub/eidos/vocab/ImpliesSupervisionTest.java`:

```java
@Test
void escalationExpected_with_disc_steadiness() {
    var registry = new io.casehub.eidos.runtime.vocabulary.CdiVocabularyRegistry();
    // Register both vocabularies manually (no CDI in unit tests)
    // Use a simpler approach: inline registry that resolves DISC terms
    var disp = AgentDisposition.builder().autonomy("steadiness").build();
    assertThat(BehavioralExpectations.escalationExpected(disp, DiscTerm.URI, discRegistry())).isTrue();
}

@Test
void escalationExpected_with_disc_dominance() {
    var disp = AgentDisposition.builder().autonomy("dominance").build();
    assertThat(BehavioralExpectations.escalationExpected(disp, DiscTerm.URI, discRegistry())).isFalse();
}
```

Add a `discRegistry()` helper that resolves DiscTerm values:

```java
private static VocabularyRegistry discRegistry() {
    return new VocabularyRegistry() {
        @Override public <T extends Enum<T> & VocabularyTerm> void register(Class<T> vocab) {}
        @Override public boolean isRegistered(String vocabUri) {
            return DiscTerm.URI.equals(vocabUri);
        }
        @Override public Optional<? extends VocabularyTerm> resolve(String vocabUri, String value) {
            if (!DiscTerm.URI.equals(vocabUri)) return Optional.empty();
            for (DiscTerm t : DiscTerm.values()) {
                if (t.value().equals(value)) return Optional.of(t);
            }
            return Optional.empty();
        }
        @Override public List<? extends VocabularyTerm> allTerms(String vocabUri) { return List.of(); }
        @Override public Optional<String> equivalentValues(String f, String v, String t) { return Optional.empty(); }
        @Override public Optional<String> equivalentValues(String f, String v, String t, DispositionAxis a) { return Optional.empty(); }
        @Override public <T extends Enum<T> & VocabularyTerm> Optional<T> resolve(Class<T> vocab, String value) { return Optional.empty(); }
        @Override public <S extends Enum<S> & VocabularyTerm, T extends Enum<T> & VocabularyTerm> Optional<T> equivalentValues(S from, Class<T> targetVocab) { return Optional.empty(); }
        @Override public <S extends Enum<S> & VocabularyTerm, T extends Enum<T> & VocabularyTerm> Optional<T> equivalentValues(S from, Class<T> targetVocab, DispositionAxis axis) { return Optional.empty(); }
        @Override public Optional<VocabularyMetadata> vocabularyMetadata(String uri) { return Optional.empty(); }
        @Override public boolean subsumes(String vocabUri, String generalValue, String specificValue) { return false; }
        @Override public MatchDegree match(String vocabUri, String declaredValue, String requestedValue) { return MatchDegree.NONE; }
        @Override public List<? extends VocabularyTerm> ancestors(String vocabUri, String value) { return List.of(); }
        @Override public List<? extends VocabularyTerm> descendants(String vocabUri, String value) { return List.of(); }
        @Override public java.util.Map<String, java.util.Set<String>> expandForMatchingByVocabulary(String value) { return java.util.Map.of(); }
    };
}
```

Add required imports:

```java
import io.casehub.eidos.api.AgentDisposition;
import io.casehub.eidos.api.BehavioralExpectations;
import io.casehub.eidos.api.MatchDegree;
import io.casehub.eidos.api.VocabularyMetadata;
import io.casehub.eidos.api.VocabularyRegistry;
import java.util.List;
import java.util.Optional;
```

- [ ] **Step 2: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl vocab -Dtest=ImpliesSupervisionTest`
Expected: All tests PASS.

- [ ] **Step 3: Update CLAUDE.md**

Update the `ComplianceDimension` description in CLAUDE.md to include the new constants. Find the line referencing `ComplianceDimension.java` and add DELEGATION, ESCALATION. Find the `BehavioralExpectations` description and add `escalationExpected`. Find the `VocabularyTerm` description and add `impliesSupervision()`.

- [ ] **Step 4: Update ARC42STORIES.MD**

Update the relevant sections in ARC42STORIES.MD:
- §1 description — add DELEGATION, ESCALATION to ComplianceDimension constant list
- L1 and L4 layer entries — add `escalationExpected()` to BehavioralExpectations method list
- ComplianceDimension.java file entry — add DELEGATION, ESCALATION constants
- BehavioralExpectations.java file entry — add `escalationExpected` signatures
- VocabularyTerm.java file entry — add `impliesSupervision()` default method

- [ ] **Step 5: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
Expected: All tests PASS across all modules.

- [ ] **Step 6: Commit**

```
git add vocab/src/test/java/io/casehub/eidos/vocab/ImpliesSupervisionTest.java \
       CLAUDE.md ARC42STORIES.MD
git commit -m "feat(eidos#87): cross-vocabulary escalation tests, CLAUDE.md and ARC42STORIES.MD updates"
```
