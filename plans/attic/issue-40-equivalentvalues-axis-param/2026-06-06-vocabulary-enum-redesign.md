# Vocabulary Enum Redesign + DispositionAxis Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the stringly-typed `Vocabulary`/`VocabularyTerm` records with an enum-based `VocabularyTerm` interface + `DispositionAxis` enum, giving cross-vocabulary mappings compile-time type safety and axis-aware resolution.

**Architecture:** New API types (`DispositionAxis`, `@VocabularyMetadata`, `VocabularyTerm` interface, `VocabularyRegistrar` SPI, updated `VocabularyRegistry`) are created first, then `CdiVocabularyRegistry` is fully rewritten to implement the new SPI, then the three vocab producers are replaced by enum classes. Old `Vocabulary`/`VocabularyTerm` records and `@Produces Vocabulary` producer beans are deleted entirely.

**Tech Stack:** Java 21, Quarkus 3.32.2, JUnit 5, AssertJ, `@QuarkusTest`. Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`.

---

## File Map

### api module (`api/src/main/java/io/casehub/eidos/api/`)

| File | Action |
|------|--------|
| `DispositionAxis.java` | **CREATE** — enum of four axis names |
| `VocabularyMetadata.java` | **CREATE** — annotation carrying uri/name/version |
| `VocabularyTerm.java` | **DELETE record; CREATE interface** (same name, same package) |
| `Vocabulary.java` | **DELETE** |
| `VocabularyRegistry.java` | **REWRITE** — remove old methods; add typed + string-based API |
| `AgentDisposition.java` | **MODIFY** — add `get(DispositionAxis)` method |
| `api/spi/VocabularyRegistrar.java` | **CREATE** — CDI SPI for vocabulary registration |

### api test module (`api/src/test/java/io/casehub/eidos/api/`)

| File | Action |
|------|--------|
| `AgentDispositionTest.java` | **MODIFY** — add `get()` tests (file already exists) |

### runtime module (`runtime/src/main/java/io/casehub/eidos/runtime/vocabulary/`)

| File | Action |
|------|--------|
| `CdiVocabularyRegistry.java` | **REWRITE** — inject `Instance<VocabularyRegistrar>`, three-map internal state |

### runtime test module (`runtime/src/test/java/io/casehub/eidos/runtime/vocabulary/`)

| File | Action |
|------|--------|
| `CdiVocabularyRegistryTest.java` | **REWRITE** — full new test suite |

### vocab module (`vocab/src/main/java/io/casehub/eidos/vocab/`)

| File | Action |
|------|--------|
| `ConscientiousnessVocabularyProducer.java` | **DELETE** |
| `SvoVocabularyProducer.java` | **DELETE** |
| `CasehubSlotVocabularyProducer.java` | **DELETE** |
| `ConscientiousnessTerm.java` | **CREATE** — enum with 12 terms |
| `ConsciousnessVocabRegistrar.java` | **CREATE** — `@ApplicationScoped VocabularyRegistrar` |
| `SvoTerm.java` | **CREATE** — enum with 3 terms + exactMatch to CasehubSlot |
| `SvoVocabRegistrar.java` | **CREATE** |
| `CasehubSlotTerm.java` | **CREATE** — enum with 4 terms + exactMatch back to SVO |
| `CasehubSlotVocabRegistrar.java` | **CREATE** |

### vocab test module (`vocab/src/test/java/io/casehub/eidos/vocab/`)

| File | Action |
|------|--------|
| `ConscientiousnessVocabularyTest.java` | **REWRITE** |
| `SvoVocabularyTest.java` | **REWRITE** |
| `CasehubSlotVocabularyTest.java` | **REWRITE** |

### examples module

| File | Action |
|------|--------|
| `examples/.../CrossVocabularyDiscoveryTest.java` | **UPDATE** — `find(URI)` → `isRegistered(URI)`, typed assertions |
| `examples/.../DispositionVocabularyTest.java` | **UPDATE** — typed resolve + allTerms |

---

## Task 1: `DispositionAxis` enum

**Files:**
- Create: `api/src/main/java/io/casehub/eidos/api/DispositionAxis.java`

- [ ] **Step 1.1: Create `DispositionAxis.java`**

```java
package io.casehub.eidos.api;

public enum DispositionAxis {
    SOCIAL_ORIENTATION,
    RULE_FOLLOWING,
    RISK_APPETITE,
    AUTONOMY;
}
```

- [ ] **Step 1.2: Compile to verify**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api -q
```
Expected: BUILD SUCCESS

- [ ] **Step 1.3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add api/src/main/java/io/casehub/eidos/api/DispositionAxis.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#40): add DispositionAxis enum"
```

---

## Task 2: `@VocabularyMetadata` annotation

**Files:**
- Create: `api/src/main/java/io/casehub/eidos/api/VocabularyMetadata.java`

- [ ] **Step 2.1: Create `VocabularyMetadata.java`**

```java
package io.casehub.eidos.api;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Vocabulary-level metadata for an enum implementing {@link VocabularyTerm}.
 * {@code name()} and {@code version()} default to {@code ""} meaning "not provided";
 * callers should treat {@code name().isEmpty()} as absent.
 */
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface VocabularyMetadata {
    String uri();
    String name()    default "";
    String version() default "";
}
```

- [ ] **Step 2.2: Compile to verify**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api -q
```
Expected: BUILD SUCCESS

- [ ] **Step 2.3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add api/src/main/java/io/casehub/eidos/api/VocabularyMetadata.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#40): add @VocabularyMetadata annotation"
```

---

## Task 3: `AgentDisposition.get(DispositionAxis)` with tests

`AgentDispositionTest.java` already exists. We ADD two new test methods, then add the method.

**Files:**
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentDisposition.java`
- Modify: `api/src/test/java/io/casehub/eidos/api/AgentDispositionTest.java`

- [ ] **Step 3.1: Add failing tests to `AgentDispositionTest.java`**

Add these two methods inside the existing class (after the last `@Test` method):

```java
@Test
void get_returns_value_for_each_axis() {
    var d = new AgentDisposition("collaborative", "strict", "conservative", "directed", false);
    assertThat(d.get(DispositionAxis.SOCIAL_ORIENTATION)).contains("collaborative");
    assertThat(d.get(DispositionAxis.RULE_FOLLOWING)).contains("strict");
    assertThat(d.get(DispositionAxis.RISK_APPETITE)).contains("conservative");
    assertThat(d.get(DispositionAxis.AUTONOMY)).contains("directed");
}

@Test
void get_returns_empty_for_null_axis_field() {
    var d = new AgentDisposition(null, null, null, null, false);
    assertThat(d.get(DispositionAxis.SOCIAL_ORIENTATION)).isEmpty();
    assertThat(d.get(DispositionAxis.RULE_FOLLOWING)).isEmpty();
    assertThat(d.get(DispositionAxis.RISK_APPETITE)).isEmpty();
    assertThat(d.get(DispositionAxis.AUTONOMY)).isEmpty();
}
```

- [ ] **Step 3.2: Run to confirm failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=AgentDispositionTest -q 2>&1 | tail -5
```
Expected: COMPILATION ERROR — `cannot find symbol: method get(DispositionAxis)`

- [ ] **Step 3.3: Add `get(DispositionAxis)` to `AgentDisposition.java`**

Add `import java.util.Optional;` at the top, then add this method inside the record body (after the compact constructor closing brace):

```java
    public Optional<String> get(DispositionAxis axis) {
        return switch (axis) {
            case SOCIAL_ORIENTATION -> Optional.ofNullable(socialOrient);
            case RULE_FOLLOWING     -> Optional.ofNullable(ruleFollowing);
            case RISK_APPETITE      -> Optional.ofNullable(riskAppetite);
            case AUTONOMY           -> Optional.ofNullable(autonomy);
        };
    }
```

- [ ] **Step 3.4: Run tests to confirm pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=AgentDispositionTest
```
Expected: BUILD SUCCESS, all tests pass

- [ ] **Step 3.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add api/src/main/java/io/casehub/eidos/api/AgentDisposition.java api/src/test/java/io/casehub/eidos/api/AgentDispositionTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#40): AgentDisposition.get(DispositionAxis) — exhaustive switch anchors compile-time axis completeness Refs #40"
```

---

## Task 4: Core Registry Rewrite (atomic)

This task deletes the old `Vocabulary` record, replaces the `VocabularyTerm` record with an interface of the same name, rewrites `VocabularyRegistry`, adds `VocabularyRegistrar` SPI, deletes old vocab producers, rewrites `CdiVocabularyRegistry`, and rewrites `CdiVocabularyRegistryTest`. All steps must complete before the build is green.

**Files:**
- Delete: `api/src/main/java/io/casehub/eidos/api/Vocabulary.java`
- Delete + recreate: `api/src/main/java/io/casehub/eidos/api/VocabularyTerm.java` (record → interface)
- Create: `api/src/main/java/io/casehub/eidos/api/spi/VocabularyRegistrar.java`
- Rewrite: `api/src/main/java/io/casehub/eidos/api/VocabularyRegistry.java`
- Delete: `vocab/src/main/java/io/casehub/eidos/vocab/ConscientiousnessVocabularyProducer.java`
- Delete: `vocab/src/main/java/io/casehub/eidos/vocab/SvoVocabularyProducer.java`
- Delete: `vocab/src/main/java/io/casehub/eidos/vocab/CasehubSlotVocabularyProducer.java`
- Rewrite: `runtime/src/main/java/io/casehub/eidos/runtime/vocabulary/CdiVocabularyRegistry.java`
- Rewrite: `runtime/src/test/java/io/casehub/eidos/runtime/vocabulary/CdiVocabularyRegistryTest.java`

- [ ] **Step 4.1: Write new `CdiVocabularyRegistryTest.java`** (will not compile yet — that's expected)

```java
package io.casehub.eidos.runtime.vocabulary;

import io.casehub.eidos.api.*;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Optional;

import static org.assertj.core.api.Assertions.*;

@QuarkusTest
class CdiVocabularyRegistryTest {

    @Inject VocabularyRegistry registry;

    // --- Test vocabulary enums defined inline ---
    // These are registered programmatically in each test using unique URIs to avoid
    // cross-test interference in the shared CDI context.

    @VocabularyMetadata(uri = "urn:test:source", name = "Source Vocab", version = "1.0")
    enum SourceTerm implements VocabularyTerm {
        ALPHA("alpha", "Alpha", "First term", List.of("a", "one")) {
            @Override public Optional<VocabularyTerm> exactMatch(Class<?> t) {
                // Class identity is correct — Class instances are singletons per class loader
                return t == TargetTerm.class ? Optional.of(TargetTerm.PRIMARY) : Optional.empty();
            }
        },
        BETA("beta", "Beta", "Second term", List.of("b"));

        final String value, label, description;
        final List<String> aliases;
        SourceTerm(String v, String l, String d, List<String> a) {
            value = v; label = l; description = d; aliases = a;
        }
        @Override public String value()         { return value; }
        @Override public String label()         { return label; }
        @Override public String description()   { return description; }
        @Override public List<String> aliases() { return aliases; }
    }

    @VocabularyMetadata(uri = "urn:test:target")
    enum TargetTerm implements VocabularyTerm {
        PRIMARY("primary", "Primary", List.of()),
        SECONDARY("secondary", "Secondary", List.of());

        final String value, label;
        final List<String> aliases;
        TargetTerm(String v, String l, List<String> a) {
            value = v; label = l; aliases = a;
        }
        @Override public String value()         { return value; }
        @Override public String label()         { return label; }
        @Override public List<String> aliases() { return aliases; }
    }

    @VocabularyMetadata(uri = "urn:test:axis-source")
    enum AxisSourceTerm implements VocabularyTerm {
        DTERM("d-type", "D Type", List.of()) {
            @Override public Optional<VocabularyTerm> axisExactMatch(Class<?> t, DispositionAxis axis) {
                if (t != AxisTargetTerm.class) return Optional.empty();
                return switch (axis) {
                    case RISK_APPETITE      -> Optional.of(AxisTargetTerm.HIGH_RISK);
                    case SOCIAL_ORIENTATION -> Optional.empty();
                    case RULE_FOLLOWING     -> Optional.empty();
                    case AUTONOMY           -> Optional.empty();
                };
            }
        };
        final String value, label;
        final List<String> aliases;
        AxisSourceTerm(String v, String l, List<String> a) { value = v; label = l; aliases = a; }
        @Override public String value()         { return value; }
        @Override public String label()         { return label; }
        @Override public List<String> aliases() { return aliases; }
    }

    @VocabularyMetadata(uri = "urn:test:axis-target")
    enum AxisTargetTerm implements VocabularyTerm {
        HIGH_RISK("high-risk", "High Risk", List.of()),
        LOW_RISK("low-risk",  "Low Risk",  List.of());
        final String value, label;
        final List<String> aliases;
        AxisTargetTerm(String v, String l, List<String> a) { value = v; label = l; aliases = a; }
        @Override public String value()         { return value; }
        @Override public String label()         { return label; }
        @Override public List<String> aliases() { return aliases; }
    }

    // Dedicated enums for the typed-bypass test — never registered anywhere in this class
    @VocabularyMetadata(uri = "urn:test:bypass-source")
    enum BypassSource implements VocabularyTerm {
        TERM("term", "Term", List.of()) {
            @Override public Optional<VocabularyTerm> exactMatch(Class<?> t) {
                return t == BypassTarget.class ? Optional.of(BypassTarget.MAPPED) : Optional.empty();
            }
        };
        final String value, label;
        final List<String> aliases;
        BypassSource(String v, String l, List<String> a) { value = v; label = l; aliases = a; }
        @Override public String value()         { return value; }
        @Override public String label()         { return label; }
        @Override public List<String> aliases() { return aliases; }
    }

    @VocabularyMetadata(uri = "urn:test:bypass-target")
    enum BypassTarget implements VocabularyTerm {
        MAPPED("mapped", "Mapped", List.of());
        final String value, label;
        final List<String> aliases;
        BypassTarget(String v, String l, List<String> a) { value = v; label = l; aliases = a; }
        @Override public String value()         { return value; }
        @Override public String label()         { return label; }
        @Override public List<String> aliases() { return aliases; }
    }

    // --- Registration tests ---

    @Test
    void programmatic_register_and_isRegistered() {
        registry.register(SourceTerm.class);
        assertThat(registry.isRegistered("urn:test:source")).isTrue();
    }

    @Test
    void isRegistered_false_for_unknown_uri() {
        assertThat(registry.isRegistered("urn:does-not-exist")).isFalse();
    }

    @Test
    void register_missing_annotation_throws() {
        @VocabularyMetadata(uri = "")
        enum BadEnum implements VocabularyTerm {
            TERM("t", "T", List.of());
            final String value, label; final List<String> aliases;
            BadEnum(String v, String l, List<String> a) { value=v; label=l; aliases=a; }
            @Override public String value()         { return value; }
            @Override public String label()         { return label; }
            @Override public List<String> aliases() { return aliases; }
        }
        // Use an enum with no annotation instead
        assertThatThrownBy(() -> registry.register(DispositionAxis.class))
            .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void register_duplicate_primary_value_throws() {
        @VocabularyMetadata(uri = "urn:test:dup-primary")
        enum DupPrimary implements VocabularyTerm {
            A("same", "A", List.of()), B("same", "B", List.of());
            final String value, label; final List<String> aliases;
            DupPrimary(String v, String l, List<String> a) { value=v; label=l; aliases=a; }
            @Override public String value()         { return value; }
            @Override public String label()         { return label; }
            @Override public List<String> aliases() { return aliases; }
        }
        assertThatThrownBy(() -> registry.register(DupPrimary.class))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("Duplicate primary value");
    }

    @Test
    void register_alias_duplicates_primary_throws() {
        @VocabularyMetadata(uri = "urn:test:alias-collision")
        enum AliasCollision implements VocabularyTerm {
            A("alpha", "A", List.of()),
            B("beta",  "B", List.of("alpha"));  // alias collides with A's primary value
            final String value, label; final List<String> aliases;
            AliasCollision(String v, String l, List<String> a) { value=v; label=l; aliases=a; }
            @Override public String value()         { return value; }
            @Override public String label()         { return label; }
            @Override public List<String> aliases() { return aliases; }
        }
        assertThatThrownBy(() -> registry.register(AliasCollision.class))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("conflicts");
    }

    @Test
    void register_duplicate_uri_different_class_throws() {
        registry.register(SourceTerm.class);  // registers "urn:test:source"
        @VocabularyMetadata(uri = "urn:test:source")
        enum OtherSource implements VocabularyTerm {
            X("x", "X", List.of());
            final String value, label; final List<String> aliases;
            OtherSource(String v, String l, List<String> a) { value=v; label=l; aliases=a; }
            @Override public String value()         { return value; }
            @Override public String label()         { return label; }
            @Override public List<String> aliases() { return aliases; }
        }
        assertThatThrownBy(() -> registry.register(OtherSource.class))
            .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void register_same_class_again_is_idempotent() {
        registry.register(SourceTerm.class);
        assertThatNoException().isThrownBy(() -> registry.register(SourceTerm.class));
    }

    // --- allTerms() tests ---

    @Test
    void allTerms_returns_distinct_constants_in_declaration_order() {
        registry.register(SourceTerm.class);
        var terms = registry.allTerms("urn:test:source");
        assertThat(terms).hasSize(2);  // ALPHA and BETA — not duplicated by alias
        assertThat(terms.get(0).value()).isEqualTo("alpha");
        assertThat(terms.get(1).value()).isEqualTo("beta");
    }

    @Test
    void allTerms_returns_empty_list_for_unknown_uri() {
        assertThat(registry.allTerms("urn:unknown")).isEmpty();
    }

    // --- resolve() tests ---

    @Test
    void typed_resolve_by_primary_value() {
        registry.register(SourceTerm.class);
        assertThat(registry.resolve(SourceTerm.class, "alpha")).contains(SourceTerm.ALPHA);
    }

    @Test
    void typed_resolve_by_alias() {
        registry.register(SourceTerm.class);
        assertThat(registry.resolve(SourceTerm.class, "one")).contains(SourceTerm.ALPHA);
    }

    @Test
    void typed_resolve_missing_value_returns_empty() {
        registry.register(SourceTerm.class);
        assertThat(registry.resolve(SourceTerm.class, "gamma")).isEmpty();
    }

    @Test
    void typed_resolve_unregistered_vocab_returns_empty() {
        assertThat(registry.resolve(TargetTerm.class, "primary")).isEmpty();
    }

    @Test
    void string_resolve_by_primary_value() {
        registry.register(SourceTerm.class);
        var term = registry.resolve("urn:test:source", "alpha");
        assertThat(term).isPresent();
        assertThat(term.get().value()).isEqualTo("alpha");
    }

    @Test
    void string_resolve_by_alias() {
        registry.register(SourceTerm.class);
        var term = registry.resolve("urn:test:source", "a");
        assertThat(term).isPresent().map(VocabularyTerm::value).contains("alpha");
    }

    @Test
    void string_resolve_unknown_uri_returns_empty() {
        assertThat(registry.resolve("urn:unknown", "alpha")).isEmpty();
    }

    // --- axis-unaware equivalentValues() ---

    @Test
    void typed_equivalentValues_axis_unaware_returns_correct_constant() {
        assertThat(registry.equivalentValues(SourceTerm.ALPHA, TargetTerm.class))
            .contains(TargetTerm.PRIMARY);
    }

    @Test
    void typed_equivalentValues_axis_unaware_not_covered_returns_empty() {
        assertThat(registry.equivalentValues(SourceTerm.BETA, TargetTerm.class)).isEmpty();
    }

    @Test
    void string_equivalentValues_axis_unaware_returns_value() {
        registry.register(SourceTerm.class);
        registry.register(TargetTerm.class);
        assertThat(registry.equivalentValues("urn:test:source", "alpha", "urn:test:target"))
            .contains("primary");
    }

    @Test
    void string_equivalentValues_alias_resolves_source() {
        registry.register(SourceTerm.class);
        registry.register(TargetTerm.class);
        assertThat(registry.equivalentValues("urn:test:source", "one", "urn:test:target"))
            .contains("primary");
    }

    @Test
    void string_equivalentValues_unknown_source_uri_returns_empty() {
        assertThat(registry.equivalentValues("urn:unknown", "alpha", "urn:test:target")).isEmpty();
    }

    @Test
    void string_equivalentValues_unknown_target_uri_returns_empty() {
        registry.register(SourceTerm.class);
        assertThat(registry.equivalentValues("urn:test:source", "alpha", "urn:unknown")).isEmpty();
    }

    @Test
    void string_equivalentValues_unknown_value_returns_empty() {
        registry.register(SourceTerm.class);
        registry.register(TargetTerm.class);
        assertThat(registry.equivalentValues("urn:test:source", "gamma", "urn:test:target")).isEmpty();
    }

    // --- axis-aware equivalentValues() ---

    @Test
    void typed_equivalentValues_axis_aware_returns_correct_constant() {
        assertThat(registry.equivalentValues(
                AxisSourceTerm.DTERM, AxisTargetTerm.class, DispositionAxis.RISK_APPETITE))
            .contains(AxisTargetTerm.HIGH_RISK);
    }

    @Test
    void typed_equivalentValues_sparse_axis_returns_empty() {
        assertThat(registry.equivalentValues(
                AxisSourceTerm.DTERM, AxisTargetTerm.class, DispositionAxis.SOCIAL_ORIENTATION))
            .isEmpty();
    }

    @Test
    void string_equivalentValues_axis_aware_returns_value() {
        registry.register(AxisSourceTerm.class);
        registry.register(AxisTargetTerm.class);
        assertThat(registry.equivalentValues(
                "urn:test:axis-source", "d-type", "urn:test:axis-target",
                DispositionAxis.RISK_APPETITE))
            .contains("high-risk");
    }

    @Test
    void axis_unaware_overload_against_axis_only_term_returns_empty() {
        // AxisSourceTerm only implements axisExactMatch, not exactMatch
        assertThat(registry.equivalentValues(AxisSourceTerm.DTERM, AxisTargetTerm.class)).isEmpty();
    }

    // --- Typed path bypasses registration ---

    @Test
    void typed_equivalentValues_bypasses_registration() {
        // BypassSource and BypassTarget are never registered in this test class
        assertThat(registry.isRegistered("urn:test:bypass-source")).isFalse();
        assertThat(registry.equivalentValues(BypassSource.TERM, BypassTarget.class))
            .contains(BypassTarget.MAPPED);
    }
}
```

- [ ] **Step 4.2: Delete `Vocabulary.java`**

```bash
rm /Users/mdproctor/claude/casehub/eidos/api/src/main/java/io/casehub/eidos/api/Vocabulary.java
```

- [ ] **Step 4.3: Delete old `VocabularyTerm.java` record**

```bash
rm /Users/mdproctor/claude/casehub/eidos/api/src/main/java/io/casehub/eidos/api/VocabularyTerm.java
```

- [ ] **Step 4.4: Create new `VocabularyTerm.java` interface**

```java
package io.casehub.eidos.api;

import java.util.List;
import java.util.Optional;

/**
 * A term within a vocabulary. Implemented by enum constants.
 *
 * <p>{@link #exactMatch} and {@link #axisExactMatch} are independent. A term may implement
 * either, both, or neither. The registry routes axis-aware and axis-unaware lookups to the
 * appropriate method independently — calling the axis-unaware overload against a DISC term
 * (which only implements axisExactMatch) returns {@code Optional.empty()}, which is correct.
 */
public interface VocabularyTerm {
    String value();
    String label();
    default String description()   { return ""; }
    default List<String> aliases() { return List.of(); }

    /**
     * Axis-unaware cross-vocabulary equivalence.
     * Returns the equivalent constant in {@code targetVocab}, or empty if none.
     * The registry's typed overload calls {@code targetVocab.cast()} on the result.
     */
    default Optional<VocabularyTerm> exactMatch(Class<?> targetVocab) {
        return Optional.empty();
    }

    /**
     * Axis-aware cross-vocabulary equivalence.
     *
     * <p>Implementations covering a given {@code targetVocab} MUST use an exhaustive switch
     * on {@code axis} with no default branch — adding a new {@link DispositionAxis} value
     * then causes a compile error, forcing explicit coverage of the new axis.
     * {@code Optional.empty()} is a valid branch for axes with no meaningful mapping.
     * Do NOT wrap the switch in {@code Optional.of()} — that forbids gaps.
     *
     * <p>The exhaustive switch enforces completeness for the axis dimension only. Adding a
     * new target vocabulary requires a new {@code if (targetVocab == ...)} branch;
     * no compile-time enforcement exists for the target-vocabulary dimension.
     */
    default Optional<VocabularyTerm> axisExactMatch(Class<?> targetVocab, DispositionAxis axis) {
        return Optional.empty();
    }
}
```

- [ ] **Step 4.5: Create `api/spi/` package directory and `VocabularyRegistrar.java`**

```bash
mkdir -p /Users/mdproctor/claude/casehub/eidos/api/src/main/java/io/casehub/eidos/api/spi
```

```java
package io.casehub.eidos.api.spi;

import io.casehub.eidos.api.VocabularyTerm;

/**
 * CDI SPI for vocabulary registration. Implement as an {@code @ApplicationScoped} bean
 * to auto-register a vocabulary enum with {@link io.casehub.eidos.api.VocabularyRegistry}
 * at startup. The enum class must carry {@link io.casehub.eidos.api.VocabularyMetadata}.
 */
@FunctionalInterface
public interface VocabularyRegistrar {
    Class<? extends Enum<? extends VocabularyTerm>> vocabulary();
}
```

- [ ] **Step 4.6: Rewrite `VocabularyRegistry.java`**

```java
package io.casehub.eidos.api;

import java.util.List;
import java.util.Optional;

public interface VocabularyRegistry {

    // --- Registration ---

    /** Registers a vocabulary enum. The class must carry {@link VocabularyMetadata}. */
    <T extends Enum<T> & VocabularyTerm> void register(Class<T> vocab);

    boolean isRegistered(String vocabUri);

    // --- String-based resolution (runtime values / unknown vocab class at compile time) ---

    Optional<? extends VocabularyTerm> resolve(String vocabUri, String value);

    /** Returns terms in enum declaration order. Empty list if URI not registered. */
    List<? extends VocabularyTerm>     allTerms(String vocabUri);

    Optional<String> equivalentValues(String fromUri, String value, String toUri);
    Optional<String> equivalentValues(String fromUri, String value, String toUri, DispositionAxis axis);

    // --- Typed resolution (compile-time-known vocab class) ---

    /**
     * Resolves {@code value} (primary or alias) to a typed constant.
     * REQUIRES the vocabulary to be registered — uses the internal byClass index.
     */
    <T extends Enum<T> & VocabularyTerm>
        Optional<T> resolve(Class<T> vocab, String value);

    /**
     * Returns the equivalent constant in {@code targetVocab} via {@link VocabularyTerm#exactMatch}.
     * Does NOT require registration — delegates directly to the source constant's method.
     */
    <S extends Enum<S> & VocabularyTerm, T extends Enum<T> & VocabularyTerm>
        Optional<T> equivalentValues(S from, Class<T> targetVocab);

    /**
     * Returns the axis-scoped equivalent constant via {@link VocabularyTerm#axisExactMatch}.
     * Does NOT require registration — delegates directly to the source constant's method.
     */
    <S extends Enum<S> & VocabularyTerm, T extends Enum<T> & VocabularyTerm>
        Optional<T> equivalentValues(S from, Class<T> targetVocab, DispositionAxis axis);
}
```

- [ ] **Step 4.7: Delete old vocab producers**

```bash
rm /Users/mdproctor/claude/casehub/eidos/vocab/src/main/java/io/casehub/eidos/vocab/ConscientiousnessVocabularyProducer.java
rm /Users/mdproctor/claude/casehub/eidos/vocab/src/main/java/io/casehub/eidos/vocab/SvoVocabularyProducer.java
rm /Users/mdproctor/claude/casehub/eidos/vocab/src/main/java/io/casehub/eidos/vocab/CasehubSlotVocabularyProducer.java
```

- [ ] **Step 4.8: Rewrite `CdiVocabularyRegistry.java`**

```java
package io.casehub.eidos.runtime.vocabulary;

import io.casehub.eidos.api.DispositionAxis;
import io.casehub.eidos.api.VocabularyMetadata;
import io.casehub.eidos.api.VocabularyRegistry;
import io.casehub.eidos.api.VocabularyTerm;
import io.casehub.eidos.api.spi.VocabularyRegistrar;
import io.quarkus.arc.DefaultBean;
import jakarta.annotation.PostConstruct;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Any;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;

import java.util.Arrays;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;

/**
 * CDI-discovered vocabulary registry.
 *
 * <p>Thread safety: {@code register()} is single-threaded — designed for {@code @PostConstruct}
 * initialization. The internal maps are {@code ConcurrentHashMap} for safe concurrent reads
 * after {@code @PostConstruct} completes.
 */
@DefaultBean
@ApplicationScoped
public class CdiVocabularyRegistry implements VocabularyRegistry {

    @Inject @Any Instance<VocabularyRegistrar> registrars;

    // vocabUri → enum class
    private final ConcurrentHashMap<String, Class<? extends Enum<?>>> byUri =
        new ConcurrentHashMap<>();
    // class → (value + aliases) → constant  (lookup index)
    private final ConcurrentHashMap<Class<?>, Map<String, VocabularyTerm>> byClass =
        new ConcurrentHashMap<>();
    // class → constants in declaration order  (for allTerms — stored immutably)
    private final ConcurrentHashMap<Class<?>, List<? extends VocabularyTerm>> byClassOrdered =
        new ConcurrentHashMap<>();

    @PostConstruct
    void init() {
        for (VocabularyRegistrar r : registrars) {
            register(r.vocabulary());  // validated path — not an internal shortcut
        }
    }

    @Override
    @SuppressWarnings("unchecked")
    public <T extends Enum<T> & VocabularyTerm> void register(Class<T> vocab) {
        var meta = vocab.getAnnotation(VocabularyMetadata.class);
        if (meta == null) {
            throw new IllegalArgumentException(
                "Vocabulary enum " + vocab.getName() + " is missing @VocabularyMetadata");
        }
        var constants = vocab.getEnumConstants();
        if (constants == null || constants.length == 0) {
            throw new IllegalArgumentException(
                "Vocabulary enum " + vocab.getName() + " has no constants");
        }
        var uri = meta.uri();

        // Validate URI before any map writes (fast-fail)
        var existing = byUri.get(uri);
        if (existing != null && existing != vocab) {
            throw new IllegalArgumentException("URI " + uri + " already registered by "
                + existing.getName() + "; cannot register " + vocab.getName());
        }

        // Build ordered list locally — List.copyOf is immutable so allTerms() returns directly
        var orderedList = List.copyOf(Arrays.asList(constants));

        // Build lookup map locally (validates duplicates before writing)
        var lookupMap = new LinkedHashMap<String, VocabularyTerm>();
        for (var constant : constants) {
            if (lookupMap.containsKey(constant.value())) {
                throw new IllegalArgumentException(
                    "Duplicate primary value '" + constant.value() + "' in " + vocab.getName());
            }
            lookupMap.put(constant.value(), constant);
            for (var alias : constant.aliases()) {
                if (lookupMap.containsKey(alias)) {
                    throw new IllegalArgumentException(
                        "Alias '" + alias + "' conflicts with an existing value or alias in "
                            + vocab.getName());
                }
                lookupMap.put(alias, constant);
            }
        }

        // Write all three maps
        byClassOrdered.put(vocab, orderedList);
        byClass.put(vocab, Map.copyOf(lookupMap));
        byUri.put(uri, (Class<? extends Enum<?>>) vocab);
    }

    @Override
    public boolean isRegistered(String vocabUri) {
        return byUri.containsKey(vocabUri);
    }

    @Override
    public Optional<? extends VocabularyTerm> resolve(String vocabUri, String value) {
        var clazz = byUri.get(vocabUri);
        if (clazz == null) return Optional.empty();
        return Optional.ofNullable(byClass.get(clazz).get(value));
    }

    @Override
    public List<? extends VocabularyTerm> allTerms(String vocabUri) {
        var clazz = byUri.get(vocabUri);
        return clazz == null ? List.of() : byClassOrdered.get(clazz);
    }

    @Override
    public Optional<String> equivalentValues(String fromUri, String value, String toUri) {
        var sourceClass = byUri.get(fromUri);
        if (sourceClass == null) return Optional.empty();
        var term = byClass.get(sourceClass).get(value);
        if (term == null) return Optional.empty();
        var targetClass = byUri.get(toUri);
        if (targetClass == null) return Optional.empty();
        return term.exactMatch(targetClass).map(VocabularyTerm::value);
    }

    @Override
    public Optional<String> equivalentValues(String fromUri, String value, String toUri,
                                              DispositionAxis axis) {
        var sourceClass = byUri.get(fromUri);
        if (sourceClass == null) return Optional.empty();
        var term = byClass.get(sourceClass).get(value);
        if (term == null) return Optional.empty();
        var targetClass = byUri.get(toUri);
        if (targetClass == null) return Optional.empty();
        return term.axisExactMatch(targetClass, axis).map(VocabularyTerm::value);
    }

    @Override
    public <T extends Enum<T> & VocabularyTerm> Optional<T> resolve(Class<T> vocab, String value) {
        var lookup = byClass.get(vocab);
        return lookup == null
            ? Optional.empty()
            : Optional.ofNullable(vocab.cast(lookup.get(value)));
    }

    @Override
    public <S extends Enum<S> & VocabularyTerm, T extends Enum<T> & VocabularyTerm>
    Optional<T> equivalentValues(S from, Class<T> targetVocab) {
        return from.exactMatch(targetVocab).map(targetVocab::cast);
    }

    @Override
    public <S extends Enum<S> & VocabularyTerm, T extends Enum<T> & VocabularyTerm>
    Optional<T> equivalentValues(S from, Class<T> targetVocab, DispositionAxis axis) {
        return from.axisExactMatch(targetVocab, axis).map(targetVocab::cast);
    }
}
```

- [ ] **Step 4.9: Run runtime module tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime
```

Expected: BUILD SUCCESS. The `@QuarkusTest` starts with no `VocabularyRegistrar` beans
(all old producers were deleted; new vocab enums don't exist yet). All tests pass because they
register vocabs programmatically.

If there are compile errors related to the old vocab module (it references deleted producers
and the deleted record types), also run:

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime --fail-at-end 2>&1 | grep -E "ERROR|FAIL|Tests run"
```

The vocab module will fail to compile until Task 5 and 6 replace the producers — that is expected at this point.

- [ ] **Step 4.10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos \
  add api/src/main/java/io/casehub/eidos/api/VocabularyTerm.java \
      api/src/main/java/io/casehub/eidos/api/VocabularyRegistry.java \
      api/src/main/java/io/casehub/eidos/api/spi/VocabularyRegistrar.java \
      runtime/src/main/java/io/casehub/eidos/runtime/vocabulary/CdiVocabularyRegistry.java \
      runtime/src/test/java/io/casehub/eidos/runtime/vocabulary/CdiVocabularyRegistryTest.java
git -C /Users/mdproctor/claude/casehub/eidos rm \
  api/src/main/java/io/casehub/eidos/api/Vocabulary.java \
  vocab/src/main/java/io/casehub/eidos/vocab/ConscientiousnessVocabularyProducer.java \
  vocab/src/main/java/io/casehub/eidos/vocab/SvoVocabularyProducer.java \
  vocab/src/main/java/io/casehub/eidos/vocab/CasehubSlotVocabularyProducer.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#40): core registry rewrite — VocabularyTerm interface, VocabularyRegistrar SPI, CdiVocabularyRegistry Refs #40"
```

---

## Task 5: `ConscientiousnessTerm` enum + registrar + test

**Files:**
- Create: `vocab/src/main/java/io/casehub/eidos/vocab/ConscientiousnessTerm.java`
- Create: `vocab/src/main/java/io/casehub/eidos/vocab/ConsciousnessVocabRegistrar.java`
- Rewrite: `vocab/src/test/java/io/casehub/eidos/vocab/ConscientiousnessVocabularyTest.java`

- [ ] **Step 5.1: Write failing test `ConscientiousnessVocabularyTest.java`**

```java
package io.casehub.eidos.vocab;

import io.casehub.eidos.api.VocabularyTerm;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.*;

class ConscientiousnessVocabularyTest {

    @Test
    void uri_constant_is_correct() {
        assertThat(ConscientiousnessTerm.URI).isEqualTo("urn:casehub:vocab:conscientiousness");
    }

    @Test
    void covers_all_twelve_terms() {
        assertThat(ConscientiousnessTerm.values()).hasSize(12);
    }

    @Test
    void rule_following_terms_present() {
        assertThat(ConscientiousnessTerm.STRICT.value()).isEqualTo("strict");
        assertThat(ConscientiousnessTerm.PRINCIPLED.value()).isEqualTo("principled");
        assertThat(ConscientiousnessTerm.FLEXIBLE.value()).isEqualTo("flexible");
    }

    @Test
    void risk_appetite_terms_present() {
        assertThat(ConscientiousnessTerm.CONSERVATIVE.value()).isEqualTo("conservative");
        assertThat(ConscientiousnessTerm.MEASURED.value()).isEqualTo("measured");
        assertThat(ConscientiousnessTerm.BOLD.value()).isEqualTo("bold");
    }

    @Test
    void social_orientation_terms_present() {
        assertThat(ConscientiousnessTerm.COLLABORATIVE.value()).isEqualTo("collaborative");
        assertThat(ConscientiousnessTerm.INDEPENDENT.value()).isEqualTo("independent");
        assertThat(ConscientiousnessTerm.FACILITATIVE.value()).isEqualTo("facilitative");
    }

    @Test
    void autonomy_terms_present() {
        assertThat(ConscientiousnessTerm.DIRECTED.value()).isEqualTo("directed");
        assertThat(ConscientiousnessTerm.SEMI_AUTONOMOUS.value()).isEqualTo("semi-autonomous");
        assertThat(ConscientiousnessTerm.AUTONOMOUS.value()).isEqualTo("autonomous");
    }

    @Test
    void strict_has_aliases() {
        assertThat(ConscientiousnessTerm.STRICT.aliases()).contains("rule-bound", "compliant");
    }

    @Test
    void bold_description_is_non_empty() {
        assertThat(ConscientiousnessTerm.BOLD.description()).isNotEmpty();
    }

    @Test
    void vocabUri_from_annotation_matches_uri_constant() {
        var meta = ConscientiousnessTerm.class.getAnnotation(
            io.casehub.eidos.api.VocabularyMetadata.class);
        assertThat(meta.uri()).isEqualTo(ConscientiousnessTerm.URI);
    }

    @Test
    void exactMatch_returns_empty_for_all_terms() {
        for (ConscientiousnessTerm t : ConscientiousnessTerm.values()) {
            assertThat(t.exactMatch(Object.class)).isEmpty();
        }
    }
}
```

- [ ] **Step 5.2: Verify test fails (won't compile — `ConscientiousnessTerm` doesn't exist yet)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -pl vocab 2>&1 | grep "ERROR" | head -5
```
Expected: compile error about `ConscientiousnessTerm`

- [ ] **Step 5.3: Create `ConscientiousnessTerm.java`**

```java
package io.casehub.eidos.vocab;

import io.casehub.eidos.api.VocabularyMetadata;
import io.casehub.eidos.api.VocabularyTerm;

import java.util.List;

@VocabularyMetadata(uri = "urn:casehub:vocab:conscientiousness",
                    name = "Conscientiousness Disposition Axes", version = "1.0")
public enum ConscientiousnessTerm implements VocabularyTerm {

    // RULE_FOLLOWING axis
    STRICT      ("strict",         "Strict Rule Following",
                 "Follows rules rigidly",              List.of("rule-bound", "compliant")),
    PRINCIPLED  ("principled",     "Principled",
                 "Follows intent of rules",             List.of("values-based")),
    FLEXIBLE    ("flexible",       "Flexible",
                 "Adapts rules to context",             List.of("adaptive", "pragmatic")),

    // RISK_APPETITE axis
    CONSERVATIVE("conservative",   "Conservative Risk",
                 "Avoids uncertainty",                  List.of("risk-averse", "cautious")),
    MEASURED    ("measured",       "Measured Risk",
                 "Balances risk and reward",            List.of("balanced")),
    BOLD        ("bold",           "Bold Risk",
                 "Accepts uncertainty for reward",      List.of("risk-tolerant", "adventurous")),

    // SOCIAL_ORIENTATION axis
    COLLABORATIVE("collaborative", "Collaborative",
                 "Works with others by default",        List.of("team-oriented", "cooperative")),
    INDEPENDENT ("independent",    "Independent",
                 "Works alone by preference",           List.of("autonomous-social", "self-directed")),
    FACILITATIVE("facilitative",   "Facilitative",
                 "Enables others to work",              List.of("supportive", "enabling")),

    // AUTONOMY axis
    DIRECTED    ("directed",       "Directed Autonomy",
                 "Follows explicit instructions",       List.of("instruction-following")),
    SEMI_AUTONOMOUS("semi-autonomous", "Semi-Autonomous",
                 "Acts within defined boundaries",      List.of("bounded-autonomy")),
    AUTONOMOUS  ("autonomous",     "Autonomous",
                 "Acts on own judgment",                List.of("self-governing", "agentic"));

    public static final String URI = "urn:casehub:vocab:conscientiousness";

    private final String value, label, description;
    private final java.util.List<String> aliases;

    ConscientiousnessTerm(String value, String label, String description,
                           java.util.List<String> aliases) {
        this.value = value;
        this.label = label;
        this.description = description;
        this.aliases = aliases;
    }

    @Override public String value()                   { return value; }
    @Override public String label()                   { return label; }
    @Override public String description()             { return description; }
    @Override public java.util.List<String> aliases() { return aliases; }
}
```

- [ ] **Step 5.4: Create `ConsciousnessVocabRegistrar.java`**

```java
package io.casehub.eidos.vocab;

import io.casehub.eidos.api.VocabularyTerm;
import io.casehub.eidos.api.spi.VocabularyRegistrar;
import jakarta.enterprise.context.ApplicationScoped;

@ApplicationScoped
public class ConsciousnessVocabRegistrar implements VocabularyRegistrar {
    @Override
    public Class<ConscientiousnessTerm> vocabulary() {
        return ConscientiousnessTerm.class;
    }
}
```

- [ ] **Step 5.5: Run vocab module tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl vocab -Dtest=ConscientiousnessVocabularyTest
```
Expected: BUILD SUCCESS (SvoVocabularyTest and CasehubSlotVocabularyTest still fail — that's expected until Task 6)

- [ ] **Step 5.6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos \
  add vocab/src/main/java/io/casehub/eidos/vocab/ConscientiousnessTerm.java \
      vocab/src/main/java/io/casehub/eidos/vocab/ConsciousnessVocabRegistrar.java \
      vocab/src/test/java/io/casehub/eidos/vocab/ConscientiousnessVocabularyTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#40): ConscientiousnessTerm enum + registrar Refs #40"
```

---

## Task 6: `SvoTerm` + `CasehubSlotTerm` enums + registrars + tests

These two enums must be created in the same task because they have bidirectional `exactMatch()` references to each other.

**Files:**
- Create: `vocab/src/main/java/io/casehub/eidos/vocab/SvoTerm.java`
- Create: `vocab/src/main/java/io/casehub/eidos/vocab/SvoVocabRegistrar.java`
- Create: `vocab/src/main/java/io/casehub/eidos/vocab/CasehubSlotTerm.java`
- Create: `vocab/src/main/java/io/casehub/eidos/vocab/CasehubSlotVocabRegistrar.java`
- Rewrite: `vocab/src/test/java/io/casehub/eidos/vocab/SvoVocabularyTest.java`
- Rewrite: `vocab/src/test/java/io/casehub/eidos/vocab/CasehubSlotVocabularyTest.java`

- [ ] **Step 6.1: Write failing tests**

**`SvoVocabularyTest.java`:**
```java
package io.casehub.eidos.vocab;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.*;

class SvoVocabularyTest {

    @Test
    void uri_constant_is_correct() {
        assertThat(SvoTerm.URI).isEqualTo("urn:casehub:vocab:svo");
    }

    @Test
    void has_three_terms() {
        assertThat(SvoTerm.values()).hasSize(3);
    }

    @Test
    void performer_aliases_preserved() {
        assertThat(SvoTerm.PERFORMER.aliases()).containsExactlyInAnyOrder("actor", "executor");
    }

    @Test
    void evaluator_aliases_preserved() {
        assertThat(SvoTerm.EVALUATOR.aliases()).containsExactlyInAnyOrder("reviewer", "judge");
    }

    @Test
    void coordinator_aliases_preserved() {
        assertThat(SvoTerm.COORDINATOR.aliases()).containsExactlyInAnyOrder("planner", "orchestrator");
    }

    @Test
    void evaluator_maps_to_casehub_slot_reviewer() {
        var match = SvoTerm.EVALUATOR.exactMatch(CasehubSlotTerm.class);
        assertThat(match).contains(CasehubSlotTerm.REVIEWER);
    }

    @Test
    void coordinator_maps_to_casehub_slot_planner() {
        assertThat(SvoTerm.COORDINATOR.exactMatch(CasehubSlotTerm.class))
            .contains(CasehubSlotTerm.PLANNER);
    }

    @Test
    void performer_maps_to_casehub_slot_executor() {
        assertThat(SvoTerm.PERFORMER.exactMatch(CasehubSlotTerm.class))
            .contains(CasehubSlotTerm.EXECUTOR);
    }

    @Test
    void unknown_target_vocab_returns_empty() {
        assertThat(SvoTerm.EVALUATOR.exactMatch(ConscientiousnessTerm.class)).isEmpty();
    }

    @Test
    void descriptions_preserved() {
        assertThat(SvoTerm.PERFORMER.description()).isEqualTo("Executes the assigned work");
        assertThat(SvoTerm.EVALUATOR.description()).isEqualTo("Assesses quality of work");
        assertThat(SvoTerm.COORDINATOR.description()).isEqualTo("Orchestrates other agents");
    }
}
```

**`CasehubSlotVocabularyTest.java`:**
```java
package io.casehub.eidos.vocab;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.*;

class CasehubSlotVocabularyTest {

    @Test
    void uri_constant_is_correct() {
        assertThat(CasehubSlotTerm.URI).isEqualTo("urn:casehub:vocab:casehub-slot");
    }

    @Test
    void has_four_terms() {
        assertThat(CasehubSlotTerm.values()).hasSize(4);
    }

    @Test
    void reviewer_maps_to_svo_evaluator() {
        assertThat(CasehubSlotTerm.REVIEWER.exactMatch(SvoTerm.class))
            .contains(SvoTerm.EVALUATOR);
    }

    @Test
    void planner_maps_to_svo_coordinator() {
        assertThat(CasehubSlotTerm.PLANNER.exactMatch(SvoTerm.class))
            .contains(SvoTerm.COORDINATOR);
    }

    @Test
    void executor_maps_to_svo_performer() {
        assertThat(CasehubSlotTerm.EXECUTOR.exactMatch(SvoTerm.class))
            .contains(SvoTerm.PERFORMER);
    }

    @Test
    void supervisor_has_no_svo_match() {
        assertThat(CasehubSlotTerm.SUPERVISOR.exactMatch(SvoTerm.class)).isEmpty();
    }

    @Test
    void aliases_preserved() {
        assertThat(CasehubSlotTerm.PLANNER.aliases()).contains("orchestrator");
        assertThat(CasehubSlotTerm.REVIEWER.aliases()).containsExactlyInAnyOrder("evaluator", "judge");
        assertThat(CasehubSlotTerm.EXECUTOR.aliases()).contains("performer");
        assertThat(CasehubSlotTerm.SUPERVISOR.aliases()).contains("overseer");
    }

    @Test
    void bidirectional_consistency() {
        // SVO evaluator → CasehubSlot reviewer → SVO evaluator
        var reviewerMatch = CasehubSlotTerm.REVIEWER.exactMatch(SvoTerm.class);
        assertThat(reviewerMatch).isPresent();
        var backMatch = reviewerMatch.get().exactMatch(CasehubSlotTerm.class);
        assertThat(backMatch).contains(CasehubSlotTerm.REVIEWER);
    }
}
```

- [ ] **Step 6.2: Verify tests fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -pl vocab 2>&1 | grep "ERROR" | head -5
```
Expected: compile errors about `SvoTerm` and `CasehubSlotTerm`

- [ ] **Step 6.3: Create `SvoTerm.java`**

```java
package io.casehub.eidos.vocab;

import io.casehub.eidos.api.VocabularyMetadata;
import io.casehub.eidos.api.VocabularyTerm;

import java.util.List;
import java.util.Optional;

@VocabularyMetadata(uri = "urn:casehub:vocab:svo", name = "SVO Roles", version = "1.0")
public enum SvoTerm implements VocabularyTerm {

    COORDINATOR("coordinator", "Coordinator", "Orchestrates other agents",
                List.of("planner", "orchestrator")) {
        @Override public Optional<VocabularyTerm> exactMatch(Class<?> t) {
            // Class identity is correct — Class instances are singletons per class loader
            return t == CasehubSlotTerm.class
                ? Optional.of(CasehubSlotTerm.PLANNER)
                : Optional.empty();
        }
    },

    PERFORMER("performer", "Performer", "Executes the assigned work",
              List.of("actor", "executor")) {
        @Override public Optional<VocabularyTerm> exactMatch(Class<?> t) {
            return t == CasehubSlotTerm.class
                ? Optional.of(CasehubSlotTerm.EXECUTOR)
                : Optional.empty();
        }
    },

    EVALUATOR("evaluator", "Evaluator", "Assesses quality of work",
              List.of("reviewer", "judge")) {
        @Override public Optional<VocabularyTerm> exactMatch(Class<?> t) {
            return t == CasehubSlotTerm.class
                ? Optional.of(CasehubSlotTerm.REVIEWER)
                : Optional.empty();
        }
    };

    public static final String URI = "urn:casehub:vocab:svo";

    private final String value, label, description;
    private final List<String> aliases;

    SvoTerm(String value, String label, String description, List<String> aliases) {
        this.value = value;
        this.label = label;
        this.description = description;
        this.aliases = aliases;
    }

    @Override public String value()         { return value; }
    @Override public String label()         { return label; }
    @Override public String description()   { return description; }
    @Override public List<String> aliases() { return aliases; }
}
```

- [ ] **Step 6.4: Create `CasehubSlotTerm.java`**

```java
package io.casehub.eidos.vocab;

import io.casehub.eidos.api.VocabularyMetadata;
import io.casehub.eidos.api.VocabularyTerm;

import java.util.List;
import java.util.Optional;

@VocabularyMetadata(uri = "urn:casehub:vocab:casehub-slot",
                    name = "CaseHub Slot Roles", version = "1.0")
public enum CasehubSlotTerm implements VocabularyTerm {

    PLANNER("planner", "Planner", "Plans and coordinates case execution",
            List.of("orchestrator")) {
        @Override public Optional<VocabularyTerm> exactMatch(Class<?> t) {
            // Class identity is correct — Class instances are singletons per class loader
            return t == SvoTerm.class ? Optional.of(SvoTerm.COORDINATOR) : Optional.empty();
        }
    },

    REVIEWER("reviewer", "Reviewer", "Evaluates outputs for quality",
             List.of("evaluator", "judge")) {
        @Override public Optional<VocabularyTerm> exactMatch(Class<?> t) {
            return t == SvoTerm.class ? Optional.of(SvoTerm.EVALUATOR) : Optional.empty();
        }
    },

    EXECUTOR("executor", "Executor", "Executes assigned tasks",
             List.of("performer")) {
        @Override public Optional<VocabularyTerm> exactMatch(Class<?> t) {
            return t == SvoTerm.class ? Optional.of(SvoTerm.PERFORMER) : Optional.empty();
        }
    },

    SUPERVISOR("supervisor", "Supervisor", "Oversees and governs agent behaviour",
               List.of("overseer"));

    public static final String URI = "urn:casehub:vocab:casehub-slot";

    private final String value, label, description;
    private final List<String> aliases;

    CasehubSlotTerm(String value, String label, String description, List<String> aliases) {
        this.value = value;
        this.label = label;
        this.description = description;
        this.aliases = aliases;
    }

    @Override public String value()         { return value; }
    @Override public String label()         { return label; }
    @Override public String description()   { return description; }
    @Override public List<String> aliases() { return aliases; }
}
```

- [ ] **Step 6.5: Create `SvoVocabRegistrar.java`**

```java
package io.casehub.eidos.vocab;

import io.casehub.eidos.api.VocabularyTerm;
import io.casehub.eidos.api.spi.VocabularyRegistrar;
import jakarta.enterprise.context.ApplicationScoped;

@ApplicationScoped
public class SvoVocabRegistrar implements VocabularyRegistrar {
    @Override public Class<SvoTerm> vocabulary() { return SvoTerm.class; }
}
```

- [ ] **Step 6.6: Create `CasehubSlotVocabRegistrar.java`**

```java
package io.casehub.eidos.vocab;

import io.casehub.eidos.api.VocabularyTerm;
import io.casehub.eidos.api.spi.VocabularyRegistrar;
import jakarta.enterprise.context.ApplicationScoped;

@ApplicationScoped
public class CasehubSlotVocabRegistrar implements VocabularyRegistrar {
    @Override public Class<CasehubSlotTerm> vocabulary() { return CasehubSlotTerm.class; }
}
```

- [ ] **Step 6.7: Run vocab module tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl vocab
```
Expected: BUILD SUCCESS — all three vocab tests pass

- [ ] **Step 6.8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos \
  add vocab/src/main/java/io/casehub/eidos/vocab/SvoTerm.java \
      vocab/src/main/java/io/casehub/eidos/vocab/SvoVocabRegistrar.java \
      vocab/src/main/java/io/casehub/eidos/vocab/CasehubSlotTerm.java \
      vocab/src/main/java/io/casehub/eidos/vocab/CasehubSlotVocabRegistrar.java \
      vocab/src/test/java/io/casehub/eidos/vocab/SvoVocabularyTest.java \
      vocab/src/test/java/io/casehub/eidos/vocab/CasehubSlotVocabularyTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#40): SvoTerm + CasehubSlotTerm enums with bidirectional exactMatch Refs #40"
```

---

## Task 7: Update example integration tests

**Files:**
- Rewrite: `examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/CrossVocabularyDiscoveryTest.java`
- Rewrite: `examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/DispositionVocabularyTest.java`

- [ ] **Step 7.1: Rewrite `CrossVocabularyDiscoveryTest.java`**

```java
package io.casehub.eidos.examples;

import io.casehub.eidos.api.VocabularyRegistry;
import io.casehub.eidos.vocab.CasehubSlotTerm;
import io.casehub.eidos.vocab.SvoTerm;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.*;

@QuarkusTest
class CrossVocabularyDiscoveryTest {

    @Inject VocabularyRegistry vocabRegistry;

    @Test
    void svo_and_casehub_slot_vocabularies_are_registered() {
        assertThat(vocabRegistry.isRegistered(SvoTerm.URI)).isTrue();
        assertThat(vocabRegistry.isRegistered(CasehubSlotTerm.URI)).isTrue();
    }

    @Test
    void typed_svo_evaluator_maps_to_casehub_slot_reviewer() {
        var result = vocabRegistry.equivalentValues(SvoTerm.EVALUATOR, CasehubSlotTerm.class);
        assertThat(result).contains(CasehubSlotTerm.REVIEWER);
    }

    @Test
    void typed_casehub_slot_reviewer_maps_to_svo_evaluator() {
        var result = vocabRegistry.equivalentValues(CasehubSlotTerm.REVIEWER, SvoTerm.class);
        assertThat(result).contains(SvoTerm.EVALUATOR);
    }

    @Test
    void string_svo_evaluator_maps_to_casehub_slot_reviewer() {
        var result = vocabRegistry.equivalentValues(SvoTerm.URI, "evaluator", CasehubSlotTerm.URI);
        assertThat(result).contains("reviewer");
    }

    @Test
    void string_casehub_slot_reviewer_maps_to_svo_evaluator() {
        var result = vocabRegistry.equivalentValues(CasehubSlotTerm.URI, "reviewer", SvoTerm.URI);
        assertThat(result).contains("evaluator");
    }

    @Test
    void cross_reference_is_bidirectional_for_all_pairs() {
        assertThat(vocabRegistry.equivalentValues(SvoTerm.COORDINATOR, CasehubSlotTerm.class))
            .contains(CasehubSlotTerm.PLANNER);
        assertThat(vocabRegistry.equivalentValues(CasehubSlotTerm.PLANNER, SvoTerm.class))
            .contains(SvoTerm.COORDINATOR);
        assertThat(vocabRegistry.equivalentValues(SvoTerm.PERFORMER, CasehubSlotTerm.class))
            .contains(CasehubSlotTerm.EXECUTOR);
        assertThat(vocabRegistry.equivalentValues(CasehubSlotTerm.EXECUTOR, SvoTerm.class))
            .contains(SvoTerm.PERFORMER);
    }

    @Test
    void resolve_term_by_alias_via_registry() {
        // "reviewer" is an alias for SvoTerm.EVALUATOR
        var term = vocabRegistry.resolve(SvoTerm.URI, "reviewer");
        assertThat(term).isPresent();
        assertThat(term.get().value()).isEqualTo("evaluator");
    }

    @Test
    void supervisor_has_no_svo_equivalent() {
        var result = vocabRegistry.equivalentValues(
            CasehubSlotTerm.URI, "supervisor", SvoTerm.URI);
        assertThat(result).isEmpty();
    }
}
```

- [ ] **Step 7.2: Rewrite `DispositionVocabularyTest.java`**

```java
package io.casehub.eidos.examples;

import io.casehub.eidos.api.VocabularyRegistry;
import io.casehub.eidos.vocab.ConscientiousnessTerm;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.*;

@QuarkusTest
class DispositionVocabularyTest {

    @Inject VocabularyRegistry vocabRegistry;

    @Test
    void conscientiousness_vocabulary_is_registered() {
        assertThat(vocabRegistry.isRegistered(ConscientiousnessTerm.URI)).isTrue();
    }

    @Test
    void typed_resolve_strict_by_primary_value() {
        var result = vocabRegistry.resolve(ConscientiousnessTerm.class, "strict");
        assertThat(result).contains(ConscientiousnessTerm.STRICT);
    }

    @Test
    void typed_resolve_conservative_by_alias() {
        var result = vocabRegistry.resolve(ConscientiousnessTerm.class, "risk-averse");
        assertThat(result).contains(ConscientiousnessTerm.CONSERVATIVE);
    }

    @Test
    void all_terms_returns_12_in_declaration_order() {
        var terms = vocabRegistry.allTerms(ConscientiousnessTerm.URI);
        assertThat(terms).hasSize(12);
        // First term in declaration order is STRICT (rule-following axis)
        assertThat(terms.get(0).value()).isEqualTo("strict");
        // Last term is AUTONOMOUS (autonomy axis)
        assertThat(terms.get(11).value()).isEqualTo("autonomous");
    }

    @Test
    void resolve_rule_following_axis_values() {
        assertThat(vocabRegistry.resolve(ConscientiousnessTerm.class, "strict")).isPresent();
        assertThat(vocabRegistry.resolve(ConscientiousnessTerm.class, "principled")).isPresent();
        assertThat(vocabRegistry.resolve(ConscientiousnessTerm.class, "flexible")).isPresent();

        var strict = vocabRegistry.resolve(ConscientiousnessTerm.class, "strict").get();
        assertThat(strict.label()).isEqualTo("Strict Rule Following");
        assertThat(strict.aliases()).contains("rule-bound");
    }

    @Test
    void resolve_risk_appetite_axis_values() {
        assertThat(vocabRegistry.resolve(ConscientiousnessTerm.class, "conservative")).isPresent();
        assertThat(vocabRegistry.resolve(ConscientiousnessTerm.class, "measured")).isPresent();
        assertThat(vocabRegistry.resolve(ConscientiousnessTerm.class, "bold")).isPresent();

        var bold = vocabRegistry.resolve(ConscientiousnessTerm.class, "bold").get();
        assertThat(bold.aliases()).contains("risk-tolerant");
    }

    @Test
    void resolve_social_orientation_axis_values() {
        assertThat(vocabRegistry.resolve(ConscientiousnessTerm.class, "collaborative")).isPresent();
        assertThat(vocabRegistry.resolve(ConscientiousnessTerm.class, "independent")).isPresent();
        assertThat(vocabRegistry.resolve(ConscientiousnessTerm.class, "facilitative")).isPresent();
    }

    @Test
    void resolve_autonomy_axis_values() {
        assertThat(vocabRegistry.resolve(ConscientiousnessTerm.class, "directed")).isPresent();
        assertThat(vocabRegistry.resolve(ConscientiousnessTerm.class, "semi-autonomous")).isPresent();
        assertThat(vocabRegistry.resolve(ConscientiousnessTerm.class, "autonomous")).isPresent();

        var autonomous = vocabRegistry.resolve(ConscientiousnessTerm.class, "autonomous").get();
        assertThat(autonomous.aliases()).contains("self-governing", "agentic");
    }

    @Test
    void resolve_by_alias() {
        var term = vocabRegistry.resolve(ConscientiousnessTerm.URI, "risk-averse");
        assertThat(term).isPresent();
        assertThat(term.get().value()).isEqualTo("conservative");
    }
}
```

- [ ] **Step 7.3: Run examples module tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl examples/agent-scenarios \
  -Dtest="CrossVocabularyDiscoveryTest,DispositionVocabularyTest"
```
Expected: BUILD SUCCESS, all tests pass

- [ ] **Step 7.4: Run full build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test
```
Expected: BUILD SUCCESS — all modules, all tests pass

- [ ] **Step 7.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos \
  add examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/CrossVocabularyDiscoveryTest.java \
      examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/DispositionVocabularyTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#40): update integration tests — typed vocab API Closes #40"
```

---

## Self-Review

### Spec coverage check

| Spec requirement | Task |
|---|---|
| `DispositionAxis` enum (4 values) | Task 1 |
| `@VocabularyMetadata` annotation with `name/version` defaults | Task 2 |
| `AgentDisposition.get(DispositionAxis)` exhaustive switch | Task 3 |
| `VocabularyTerm` interface replacing record | Task 4 |
| `VocabularyRegistrar` SPI in `api/spi/` | Task 4 |
| `VocabularyRegistry` rewrite — both typed and string-based APIs | Task 4 |
| `CdiVocabularyRegistry` — three-map, validate-then-write, immutable orderedList | Task 4 |
| Typed `equivalentValues` bypasses registration (by design) | Task 4 test |
| `ConscientiousnessTerm` with all 12 terms + descriptions + aliases | Task 5 |
| `SvoTerm` with preserved aliases and descriptions | Task 6 |
| `CasehubSlotTerm` with correct URI (`casehub-slot` with hyphen) | Task 6 |
| Bidirectional SVO ↔ CasehubSlot cross-vocab mapping | Task 6 |
| `allTerms()` returns List in declaration order | Task 4 impl + Task 7 test |
| Registration validation (missing annotation, zero constants, duplicate values, alias collisions, duplicate URI) | Task 4 tests |
| Old `Vocabulary` record deleted | Task 4 |
| Old `@Produces Vocabulary` producers deleted | Task 4 |
| Example tests updated | Task 7 |

### Placeholder scan

No TBDs, TODOs, or incomplete sections found.

### Type consistency check

- `VocabularyTerm` interface: `value()`, `label()`, `description()`, `aliases()`, `exactMatch(Class<?>)`, `axisExactMatch(Class<?>, DispositionAxis)` — used consistently throughout all tasks.
- `VocabularyRegistry`: `register(Class<T>)`, `isRegistered(String)`, `resolve(String, String)`, `resolve(Class<T>, String)`, `allTerms(String)`, `equivalentValues(...)` — method signatures match between SPI (Task 4) and all test call sites (Tasks 4, 7).
- `DispositionAxis.SOCIAL_ORIENTATION`, `.RULE_FOLLOWING`, `.RISK_APPETITE`, `.AUTONOMY` — same names in enum (Task 1), `AgentDisposition.get()` switch (Task 3), and test switch in `AxisSourceTerm` (Task 4).
- URI constants: `SvoTerm.URI = "urn:casehub:vocab:svo"`, `CasehubSlotTerm.URI = "urn:casehub:vocab:casehub-slot"` (with hyphen), `ConscientiousnessTerm.URI = "urn:casehub:vocab:conscientiousness"` — consistent across enums and tests.
