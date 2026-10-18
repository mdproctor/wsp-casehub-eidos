# Domain Vocabulary Keywords Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #199 — Domain vocabulary keywords in VocabularyRegistry for assertion support
**Issue group:** #199

**Goal:** Enable data-driven vocabulary registration with keyword support, and ship an epistemic domain vocabulary so consumers can replace hardcoded domain keyword mappings.

**Architecture:** Extend VocabularyTerm with `keywords()`, introduce `DataDrivenTerm` record and `DataDrivenVocabularyRegistrar` SPI, refactor CdiVocabularyRegistry to URI-keyed primary storage, load vocabularies from `META-INF/eidos/vocabularies/*.yaml` via `ClasspathYamlVocabularyRegistrar`, and ship ~20 epistemic domain terms with hierarchy and keywords.

**Tech Stack:** Java 21 (on Java 26 JVM), Quarkus 3.32.2, Jackson YAML (via casehub-yaml-jackson)

## Global Constraints

- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- Test: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl <module>`
- Use `mvn` not `./mvnw`
- No `@VocabularyMetadata` annotation changes — it stays as enum-class declaration mechanism
- `epistemicDomains` keys remain freeform strings — no validation against vocabulary
- No renderer changes
- No cross-vocabulary specialization for data-driven terms

---

## Batch 1: API Foundation — types compile, tests pass

### Task 1: VocabularyTerm.keywords() + VocabularyInfo record + VocabularyRegistry contract changes

**Files:**
- Modify: `api/src/main/java/io/casehub/eidos/api/VocabularyTerm.java`
- Create: `api/src/main/java/io/casehub/eidos/api/VocabularyInfo.java`
- Modify: `api/src/main/java/io/casehub/eidos/api/VocabularyRegistry.java`
- Test: `api/src/test/java/io/casehub/eidos/api/VocabularyInfoTest.java`

**Interfaces:**
- Produces: `VocabularyTerm.keywords()` → `List<String>`, `VocabularyInfo(uri, name, version, description)`, `VocabularyRegistry.vocabularyMetadata()` returns `Optional<VocabularyInfo>`, `VocabularyRegistry.register(String uri, String name, String version, String description, List<? extends VocabularyTerm> terms)`

- [ ] **Step 1: Add keywords() default method to VocabularyTerm**

Add after line 65 (`default String antiPatternWarning() { return ""; }`):

```java
default List<String> keywords() { return List.of(); }
```

Update Javadoc on the interface (line 6) from "Implemented by enum constants" to:

```java
/**
 * A term within a vocabulary. Implemented by enum constants or data-driven records
 * (see {@link DataDrivenTerm}).
 * ...
 */
```

- [ ] **Step 2: Create VocabularyInfo record**

```java
package io.casehub.eidos.api;

public record VocabularyInfo(String uri, String name, String version, String description) {
    public VocabularyInfo {
        if (uri == null || uri.isBlank())
            throw new IllegalArgumentException("VocabularyInfo.uri must not be blank");
        if (name == null) name = "";
        if (version == null) version = "";
        if (description == null) description = "";
    }
}
```

- [ ] **Step 3: Write VocabularyInfo test**

```java
package io.casehub.eidos.api;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

class VocabularyInfoTest {
    @Test void creates_with_all_fields() {
        var info = new VocabularyInfo("urn:test", "Test", "1.0", "desc");
        assertThat(info.uri()).isEqualTo("urn:test");
        assertThat(info.name()).isEqualTo("Test");
    }

    @Test void null_coercion() {
        var info = new VocabularyInfo("urn:test", null, null, null);
        assertThat(info.name()).isEqualTo("");
        assertThat(info.version()).isEqualTo("");
        assertThat(info.description()).isEqualTo("");
    }

    @Test void blank_uri_rejected() {
        assertThatThrownBy(() -> new VocabularyInfo("", "n", "v", "d"))
            .isInstanceOf(IllegalArgumentException.class);
    }

    @Test void null_uri_rejected() {
        assertThatThrownBy(() -> new VocabularyInfo(null, "n", "v", "d"))
            .isInstanceOf(IllegalArgumentException.class);
    }
}
```

- [ ] **Step 4: Run VocabularyInfo test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=VocabularyInfoTest`
Expected: PASS

- [ ] **Step 5: Update VocabularyRegistry interface**

Change `vocabularyMetadata` return type (line 70):

```java
Optional<VocabularyInfo> vocabularyMetadata(String uri);
```

Add new register method after the existing `register(Class<T>)` method:

```java
void register(String uri, String name, String version, String description,
              List<? extends VocabularyTerm> terms);
```

- [ ] **Step 6: Fix all VocabularyRegistry implementations to compile**

7 implementations need updating. For each, add:

**`vocabularyMetadata()` return type** — change `Optional<VocabularyMetadata>` to `Optional<VocabularyInfo>`:
- `CdiVocabularyRegistry` (runtime) — full implementation in Task 3
- `TestVocabularyRegistry` (persistence-memory/src/test) — return `Optional.empty()` for now
- `StubVocabularyRegistry` in `CapabilityResolverTest` (api/src/test) — return `Optional.empty()`
- `StubVocabularyRegistry` in `DefaultCapabilityHealthBehavioralViolationTest` (runtime/src/test) — return `Optional.empty()`
- `NoOpVocabRegistry` in `DescriptorCollectorTest` (runtime/src/test) — return `Optional.empty()`
- Anonymous impl in `ImpliesSupervisionTest` (vocab/src/test) — return `Optional.empty()`
- Anonymous impl in `BehavioralExpectationsTest` (api/src/test) — return `Optional.empty()`

**`register(String, ...)` method** — add no-op or throwing stub to each:
- `CdiVocabularyRegistry` — full implementation in Task 3
- All 6 test implementations — add: `throw new UnsupportedOperationException();`

**Callers of `vocabularyMetadata()`** — update ~8 sites that access `VocabularyMetadata` annotation methods. Method names (`name()`, `description()`, `version()`) are identical on `VocabularyInfo`, so changes are mechanical — only the variable type changes.

- [ ] **Step 7: Run full API + runtime compile check**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api,eidos-core,runtime,persistence-memory,vocab`
Expected: BUILD SUCCESS

- [ ] **Step 8: Commit**

```bash
git add api/ runtime/ persistence-memory/ vocab/
git commit -m "feat(api): add keywords() to VocabularyTerm, VocabularyInfo record, registry contract changes

Refs #199"
```

### Task 2: DataDrivenTerm + DataDrivenVocabulary + DataDrivenVocabularyRegistrar + DomainKeywords

**Files:**
- Create: `api/src/main/java/io/casehub/eidos/api/DataDrivenTerm.java`
- Create: `api/src/main/java/io/casehub/eidos/api/DataDrivenVocabulary.java`
- Create: `api/src/main/java/io/casehub/eidos/api/spi/DataDrivenVocabularyRegistrar.java`
- Create: `api/src/main/java/io/casehub/eidos/api/DomainKeywords.java`
- Test: `api/src/test/java/io/casehub/eidos/api/DataDrivenTermTest.java`
- Test: `api/src/test/java/io/casehub/eidos/api/DataDrivenVocabularyTest.java`
- Test: `api/src/test/java/io/casehub/eidos/api/DomainKeywordsTest.java`

**Interfaces:**
- Consumes: `VocabularyTerm` (from api), `VocabularyRegistry.resolve()`, `VocabularyRegistry.ancestors()` (from api)
- Produces: `DataDrivenTerm`, `DataDrivenVocabulary`, `DataDrivenVocabularyRegistrar`, `DomainKeywords.expanded()`

- [ ] **Step 1: Write DataDrivenTerm tests**

```java
package io.casehub.eidos.api;

import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.*;

class DataDrivenTermTest {
    @Test void minimal_term() {
        var t = new DataDrivenTerm("test", "Test", "", List.of(), List.of(), List.of());
        assertThat(t.value()).isEqualTo("test");
        assertThat(t.label()).isEqualTo("Test");
        assertThat(t.keywords()).isEmpty();
        assertThat(t.specializes()).isEmpty();
    }

    @Test void null_coercion() {
        var t = new DataDrivenTerm("test", null, null, null, null, null);
        assertThat(t.label()).isEqualTo("");
        assertThat(t.description()).isEqualTo("");
        assertThat(t.aliases()).isEmpty();
        assertThat(t.keywords()).isEmpty();
        assertThat(t.specializes()).isEmpty();
    }

    @Test void blank_value_rejected() {
        assertThatThrownBy(() -> new DataDrivenTerm("", "L", "D", List.of(), List.of(), List.of()))
            .isInstanceOf(IllegalArgumentException.class);
    }

    @Test void null_value_rejected() {
        assertThatThrownBy(() -> new DataDrivenTerm(null, "L", "D", List.of(), List.of(), List.of()))
            .isInstanceOf(IllegalArgumentException.class);
    }

    @Test void banned_codepoint_rejected() {
        assertThatThrownBy(() -> new DataDrivenTerm("te​st", "L", "D", List.of(), List.of(), List.of()))
            .isInstanceOf(AgentValidationException.class);
    }

    @Test void blank_keyword_rejected() {
        assertThatThrownBy(() -> new DataDrivenTerm("t", "L", "D", List.of(), List.of("ok", ""), List.of()))
            .isInstanceOf(IllegalArgumentException.class);
    }

    @Test void duplicate_keywords_deduplicated() {
        var t = new DataDrivenTerm("t", "L", "D", List.of(), List.of("a", "b", "a"), List.of());
        assertThat(t.keywords()).containsExactly("a", "b");
    }

    @Test void lists_are_immutable() {
        var t = new DataDrivenTerm("t", "L", "D", List.of("a1"), List.of("k1"), List.of());
        assertThatThrownBy(() -> t.keywords().add("k2")).isInstanceOf(UnsupportedOperationException.class);
        assertThatThrownBy(() -> t.aliases().add("a2")).isInstanceOf(UnsupportedOperationException.class);
    }

    @Test void specializes_returns_resolved_parents() {
        var parent = new DataDrivenTerm("parent", "Parent", "", List.of(), List.of(), List.of());
        var child = new DataDrivenTerm("child", "Child", "", List.of(), List.of(), List.of(parent));
        assertThat(child.specializes()).containsExactly(parent);
    }
}
```

- [ ] **Step 2: Implement DataDrivenTerm**

```java
package io.casehub.eidos.api;

import java.util.ArrayList;
import java.util.LinkedHashSet;
import java.util.List;

public record DataDrivenTerm(
    String value, String label, String description,
    List<String> aliases, List<String> keywords,
    List<VocabularyTerm> resolvedParents
) implements VocabularyTerm {

    public DataDrivenTerm {
        if (value == null || value.isBlank())
            throw new IllegalArgumentException("DataDrivenTerm.value must not be blank");
        if (label == null) label = "";
        if (description == null) description = "";
        if (aliases == null) aliases = List.of();
        if (keywords == null) keywords = List.of();
        if (resolvedParents == null) resolvedParents = List.of();

        AgentDescriptorValidator.validateChars("value", value);
        if (!label.isEmpty()) AgentDescriptorValidator.validateChars("label", label);
        if (!description.isEmpty()) AgentDescriptorValidator.validateChars("description", description);
        for (var a : aliases) AgentDescriptorValidator.validateChars("alias", a);
        for (var k : keywords) {
            if (k == null || k.isBlank())
                throw new IllegalArgumentException("DataDrivenTerm keywords must not be blank");
            AgentDescriptorValidator.validateChars("keyword", k);
        }

        var deduped = new ArrayList<>(new LinkedHashSet<>(keywords));
        keywords = List.copyOf(deduped);
        aliases = List.copyOf(aliases);
        resolvedParents = List.copyOf(resolvedParents);
    }

    @Override public List<VocabularyTerm> specializes() { return resolvedParents; }
}
```

- [ ] **Step 3: Run DataDrivenTerm tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=DataDrivenTermTest`
Expected: PASS

- [ ] **Step 4: Write and implement DataDrivenVocabulary**

Test file `DataDrivenVocabularyTest.java`:

```java
package io.casehub.eidos.api;

import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.*;

class DataDrivenVocabularyTest {
    @Test void valid_vocabulary() {
        var t = new DataDrivenTerm("a", "A", "", List.of(), List.of(), List.of());
        var v = new DataDrivenVocabulary("urn:test", "Test", "1.0", "desc", List.of(t));
        assertThat(v.uri()).isEqualTo("urn:test");
        assertThat(v.terms()).hasSize(1);
    }

    @Test void blank_uri_rejected() {
        var t = new DataDrivenTerm("a", "A", "", List.of(), List.of(), List.of());
        assertThatThrownBy(() -> new DataDrivenVocabulary("", "n", "v", "d", List.of(t)))
            .isInstanceOf(IllegalArgumentException.class);
    }

    @Test void empty_terms_rejected() {
        assertThatThrownBy(() -> new DataDrivenVocabulary("urn:test", "n", "v", "d", List.of()))
            .isInstanceOf(IllegalArgumentException.class);
    }

    @Test void null_terms_rejected() {
        assertThatThrownBy(() -> new DataDrivenVocabulary("urn:test", "n", "v", "d", null))
            .isInstanceOf(NullPointerException.class);
    }

    @Test void duplicate_values_rejected() {
        var t1 = new DataDrivenTerm("a", "A", "", List.of(), List.of(), List.of());
        var t2 = new DataDrivenTerm("a", "B", "", List.of(), List.of(), List.of());
        assertThatThrownBy(() -> new DataDrivenVocabulary("urn:test", "n", "v", "d", List.of(t1, t2)))
            .isInstanceOf(IllegalArgumentException.class);
    }

    @Test void null_coercion() {
        var t = new DataDrivenTerm("a", "A", "", List.of(), List.of(), List.of());
        var v = new DataDrivenVocabulary("urn:test", null, null, null, List.of(t));
        assertThat(v.name()).isEqualTo("");
    }
}
```

Implementation:

```java
package io.casehub.eidos.api;

import java.util.HashSet;
import java.util.List;
import java.util.Objects;

public record DataDrivenVocabulary(
    String uri, String name, String version, String description,
    List<DataDrivenTerm> terms
) {
    public DataDrivenVocabulary {
        if (uri == null || uri.isBlank())
            throw new IllegalArgumentException("DataDrivenVocabulary.uri must not be blank");
        Objects.requireNonNull(terms, "terms must not be null");
        if (terms.isEmpty())
            throw new IllegalArgumentException("DataDrivenVocabulary must have at least one term");
        if (name == null) name = "";
        if (version == null) version = "";
        if (description == null) description = "";
        var seen = new HashSet<String>();
        for (var t : terms) {
            if (!seen.add(t.value()))
                throw new IllegalArgumentException("Duplicate term value: " + t.value());
        }
        terms = List.copyOf(terms);
    }
}
```

- [ ] **Step 5: Create DataDrivenVocabularyRegistrar SPI**

```java
package io.casehub.eidos.api.spi;

import io.casehub.eidos.api.DataDrivenVocabulary;
import java.util.List;

@FunctionalInterface
public interface DataDrivenVocabularyRegistrar {
    List<DataDrivenVocabulary> vocabularies();
}
```

- [ ] **Step 6: Write and implement DomainKeywords utility**

Test file `DomainKeywordsTest.java` — uses existing `StubVocabularyRegistry` from `CapabilityResolverTest` as a pattern, but creates a minimal inline stub:

```java
package io.casehub.eidos.api;

import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Optional;
import static org.assertj.core.api.Assertions.*;

class DomainKeywordsTest {
    @Test void returns_own_keywords_when_no_ancestors() {
        var term = new DataDrivenTerm("java", "Java", "", List.of(),
            List.of("jvm", "bytecode"), List.of());
        var registry = stubRegistry("urn:test", term);
        assertThat(DomainKeywords.expanded(registry, "urn:test", "java"))
            .containsExactly("jvm", "bytecode");
    }

    @Test void includes_ancestor_keywords() {
        var parent = new DataDrivenTerm("se", "SE", "", List.of(),
            List.of("code", "debug"), List.of());
        var child = new DataDrivenTerm("java", "Java", "", List.of(),
            List.of("jvm"), List.of(parent));
        var registry = stubRegistry("urn:test", parent, child);
        assertThat(DomainKeywords.expanded(registry, "urn:test", "java"))
            .containsExactly("jvm", "code", "debug");
    }

    @Test void deduplicates_keywords() {
        var parent = new DataDrivenTerm("se", "SE", "", List.of(),
            List.of("code", "jvm"), List.of());
        var child = new DataDrivenTerm("java", "Java", "", List.of(),
            List.of("jvm", "bytecode"), List.of(parent));
        var registry = stubRegistry("urn:test", parent, child);
        var result = DomainKeywords.expanded(registry, "urn:test", "java");
        assertThat(result).containsExactly("jvm", "bytecode", "code");
    }

    @Test void returns_empty_for_unknown_term() {
        var registry = stubRegistry("urn:test",
            new DataDrivenTerm("a", "A", "", List.of(), List.of("x"), List.of()));
        assertThat(DomainKeywords.expanded(registry, "urn:test", "unknown")).isEmpty();
    }

    @Test void returns_empty_for_unknown_vocabulary() {
        var registry = stubRegistry("urn:test",
            new DataDrivenTerm("a", "A", "", List.of(), List.of("x"), List.of()));
        assertThat(DomainKeywords.expanded(registry, "urn:other", "a")).isEmpty();
    }

    // Minimal stub — only implements resolve() and ancestors()
    private VocabularyRegistry stubRegistry(String uri, DataDrivenTerm... terms) {
        // Build a minimal VocabularyRegistry that supports resolve + ancestors
        // by delegating to a CdiVocabularyRegistry with data-driven registration.
        // For the DomainKeywords test, we only need resolve() and ancestors().
        // Use an anonymous impl that stores the terms in a map.
        var termMap = new java.util.LinkedHashMap<String, VocabularyTerm>();
        for (var t : terms) termMap.put(t.value(), t);
        return new MinimalTestRegistry(uri, termMap);
    }
}
```

Note: `MinimalTestRegistry` is a private static inner class implementing the subset of VocabularyRegistry needed. It only needs `resolve(uri, value)` and `ancestors(uri, value)` to work. All other methods throw `UnsupportedOperationException`.

Implementation:

```java
package io.casehub.eidos.api;

import java.util.ArrayList;
import java.util.LinkedHashSet;
import java.util.List;

public final class DomainKeywords {
    private DomainKeywords() {}

    public static List<String> expanded(VocabularyRegistry registry,
                                         String vocabUri, String value) {
        var term = registry.resolve(vocabUri, value);
        if (term.isEmpty()) return List.of();
        var result = new LinkedHashSet<>(term.get().keywords());
        for (var ancestor : registry.ancestors(vocabUri, value)) {
            result.addAll(ancestor.keywords());
        }
        return List.copyOf(new ArrayList<>(result));
    }
}
```

- [ ] **Step 7: Run all API tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add api/
git commit -m "feat(api): add DataDrivenTerm, DataDrivenVocabulary, DataDrivenVocabularyRegistrar SPI, DomainKeywords utility

Refs #199"
```

---

## Batch 2: Registry Refactor — CdiVocabularyRegistry supports data-driven vocabularies

### Task 3: CdiVocabularyRegistry URI-keyed refactor + data-driven registration

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/vocabulary/CdiVocabularyRegistry.java`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/EidosCoreProducer.java`
- Test: `runtime/src/test/java/io/casehub/eidos/runtime/vocabulary/CdiVocabularyRegistryTest.java` (existing, extend)

**Interfaces:**
- Consumes: `DataDrivenTerm`, `DataDrivenVocabulary`, `DataDrivenVocabularyRegistrar` (from api), `VocabularyInfo` (from api)
- Produces: Full data-driven vocabulary registration, URI-keyed storage, hierarchy builder supporting non-enum terms

- [ ] **Step 1: Write tests for data-driven registration**

Add to existing `CdiVocabularyRegistryTest.java`:

```java
@Test void register_data_driven_vocabulary() {
    var parent = new DataDrivenTerm("parent", "Parent", "desc", List.of(), List.of("kw1"), List.of());
    var child = new DataDrivenTerm("child", "Child", "desc", List.of(), List.of("kw2"), List.of(parent));
    registry.register("urn:test:dd", "Test DD", "1.0", "desc", List.of(parent, child));

    assertThat(registry.isRegistered("urn:test:dd")).isTrue();
    assertThat(registry.registeredUris()).contains("urn:test:dd");
    assertThat(registry.resolve("urn:test:dd", "parent")).isPresent();
    assertThat(registry.resolve("urn:test:dd", "child")).isPresent();
    assertThat(registry.resolve("urn:test:dd", "child").get().keywords()).containsExactly("kw2");
    assertThat(registry.allTerms("urn:test:dd")).hasSize(2);
}

@Test void data_driven_hierarchy_works() {
    var parent = new DataDrivenTerm("parent", "P", "", List.of(), List.of(), List.of());
    var child = new DataDrivenTerm("child", "C", "", List.of(), List.of(), List.of(parent));
    registry.register("urn:test:dd", "T", "1", "", List.of(parent, child));

    assertThat(registry.subsumes("urn:test:dd", "parent", "child")).isTrue();
    assertThat(registry.ancestors("urn:test:dd", "child")).extracting(VocabularyTerm::value).containsExactly("parent");
    assertThat(registry.descendants("urn:test:dd", "parent")).extracting(VocabularyTerm::value).containsExactly("child");

    var match = registry.match("urn:test:dd", "parent", "child");
    assertThat(match).isInstanceOf(MatchDegree.Plugin.class);
}

@Test void data_driven_vocabularyMetadata_returns_VocabularyInfo() {
    var t = new DataDrivenTerm("a", "A", "", List.of(), List.of(), List.of());
    registry.register("urn:test:dd", "My Vocab", "2.0", "A description", List.of(t));

    var info = registry.vocabularyMetadata("urn:test:dd");
    assertThat(info).isPresent();
    assertThat(info.get().name()).isEqualTo("My Vocab");
    assertThat(info.get().version()).isEqualTo("2.0");
}

@Test void data_driven_duplicate_uri_rejected() {
    var t = new DataDrivenTerm("a", "A", "", List.of(), List.of(), List.of());
    registry.register("urn:test:dd", "T", "1", "", List.of(t));
    assertThatThrownBy(() -> registry.register("urn:test:dd", "T2", "1", "", List.of(t)))
        .isInstanceOf(IllegalArgumentException.class);
}

@Test void data_driven_uri_collision_with_enum_rejected() {
    // Register an enum vocab first, then try data-driven with same URI
    registry.register(SvoTerm.class);
    var t = new DataDrivenTerm("a", "A", "", List.of(), List.of(), List.of());
    assertThatThrownBy(() -> registry.register(SvoTerm.URI, "T", "1", "", List.of(t)))
        .isInstanceOf(IllegalArgumentException.class);
}

@Test void enum_vocabularyMetadata_returns_VocabularyInfo() {
    registry.register(SvoTerm.class);
    var info = registry.vocabularyMetadata(SvoTerm.URI);
    assertThat(info).isPresent();
    assertThat(info.get().uri()).isEqualTo(SvoTerm.URI);
}

@Test void equivalentValues_returns_empty_for_data_driven_vocab() {
    var t = new DataDrivenTerm("a", "A", "", List.of(), List.of(), List.of());
    registry.register("urn:test:dd", "T", "1", "", List.of(t));
    assertThat(registry.equivalentValues("urn:test:dd", "a", SvoTerm.URI)).isEmpty();
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=CdiVocabularyRegistryTest`
Expected: FAIL (methods not yet implemented)

- [ ] **Step 3: Implement CdiVocabularyRegistry refactor**

This is the largest single implementation step. The changes are:

1. Add new URI-keyed maps: `termsByUri`, `orderedByUri`, `metadataByUri` (all `ConcurrentHashMap`)
2. Add `@Inject @Any Instance<DataDrivenVocabularyRegistrar> dataDrivenRegistrars`
3. Update `init()` to Phase 1a (enum) + Phase 1b (data-driven) + Phase 2 (hierarchy)
4. Add `registerDataDriven(DataDrivenVocabulary)` — populates URI-keyed maps, validates URI collision against both `byUri` and `metadataByUri`
5. Implement `register(String uri, ...)` public method — delegates to `registerDataDriven`
6. Update `registerTerms()` to also populate `termsByUri`, `orderedByUri`, `metadataByUri` (from `@VocabularyMetadata` annotation → `VocabularyInfo`). Add cross-type collision check against `metadataByUri`.
7. Update `vocabularyMetadata()` to return `Optional<VocabularyInfo>` from `metadataByUri`
8. Update `buildAllHierarchyIndexes()` — iterate `orderedByUri` instead of `vocabSnapshot`, use `termToVocabUri` map instead of `getDeclaringClass()` cast
9. Update all string-based read methods to use `termsByUri`/`orderedByUri`: `resolve(uri, value)`, `allTerms(uri)`, `isRegistered()`, `registeredUris()`, `equivalentValues(fromUri, ...)` (returns empty when source or target is data-driven — no backing class for `exactMatch`)
10. Update `register(Class<T>)` (late registration) to also populate URI-keyed maps and check `metadataByUri` for cross-type collision

- [ ] **Step 4: Run registry tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=CdiVocabularyRegistryTest`
Expected: PASS

- [ ] **Step 5: Run full runtime test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: PASS — existing enum vocabulary tests still work after refactor

- [ ] **Step 7: Commit**

```bash
git add runtime/
git commit -m "feat(runtime): refactor CdiVocabularyRegistry to URI-keyed storage, support data-driven vocabularies

Refs #199"
```

---

## Batch 3: YAML Loader + Schema + Domain Vocabulary — end-to-end working

### Task 4: ClasspathYamlVocabularyRegistrar + CDI wiring

**Files:**
- Create: `eidos-core/src/main/java/io/casehub/eidos/core/vocabulary/ClasspathYamlVocabularyRegistrar.java`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/EidosCoreProducer.java`
- Test: `runtime/src/test/java/io/casehub/eidos/runtime/vocabulary/ClasspathYamlVocabularyRegistrarTest.java`
- Test resource: `runtime/src/test/resources/META-INF/eidos/vocabularies.yaml`

**Interfaces:**
- Consumes: `DataDrivenVocabulary`, `DataDrivenTerm`, `DataDrivenVocabularyRegistrar` (from api)
- Produces: `ClasspathYamlVocabularyRegistrar` — loads `META-INF/eidos/vocabularies/*.yaml`, resolves intra-vocabulary specializes references, returns `List<DataDrivenVocabulary>`

- [ ] **Step 1: Create test YAML vocabulary**

File `runtime/src/test/resources/META-INF/eidos/vocabularies.yaml`:

```yaml
uri: "urn:test:yaml-vocab"
name: Test YAML Vocabulary
version: "1.0"
description: Vocabulary for ClasspathYamlVocabularyRegistrar tests

terms:
  - value: parent-term
    label: Parent Term
    description: A top-level term
    keywords:
      - keyword-a
      - keyword-b

  - value: child-term
    label: Child Term
    description: A child term
    specializes: [parent-term]
    keywords:
      - keyword-c
    aliases:
      - child-alias
```

- [ ] **Step 2: Write ClasspathYamlVocabularyRegistrar tests**

```java
package io.casehub.eidos.runtime.vocabulary;

import io.casehub.eidos.core.vocabulary.ClasspathYamlVocabularyRegistrar;
import org.junit.jupiter.api.Test;
import java.io.InputStream;
import static org.assertj.core.api.Assertions.*;

class ClasspathYamlVocabularyRegistrarTest {

    @Test void loads_vocabulary_from_yaml() {
        var registrar = new ClasspathYamlVocabularyRegistrar();
        var vocabs = registrar.vocabularies();
        var testVocab = vocabs.stream()
            .filter(v -> v.uri().equals("urn:test:yaml-vocab"))
            .findFirst();
        assertThat(testVocab).isPresent();
        assertThat(testVocab.get().terms()).hasSize(2);
        assertThat(testVocab.get().name()).isEqualTo("Test YAML Vocabulary");
    }

    @Test void resolves_specializes_references() {
        var registrar = new ClasspathYamlVocabularyRegistrar();
        var vocab = registrar.vocabularies().stream()
            .filter(v -> v.uri().equals("urn:test:yaml-vocab")).findFirst().orElseThrow();
        var child = vocab.terms().stream()
            .filter(t -> t.value().equals("child-term")).findFirst().orElseThrow();
        assertThat(child.specializes()).hasSize(1);
        assertThat(child.specializes().get(0).value()).isEqualTo("parent-term");
    }

    @Test void loads_keywords_and_aliases() {
        var registrar = new ClasspathYamlVocabularyRegistrar();
        var vocab = registrar.vocabularies().stream()
            .filter(v -> v.uri().equals("urn:test:yaml-vocab")).findFirst().orElseThrow();
        var parent = vocab.terms().stream()
            .filter(t -> t.value().equals("parent-term")).findFirst().orElseThrow();
        assertThat(parent.keywords()).containsExactly("keyword-a", "keyword-b");
        var child = vocab.terms().stream()
            .filter(t -> t.value().equals("child-term")).findFirst().orElseThrow();
        assertThat(child.aliases()).containsExactly("child-alias");
    }

    @Test void loadFrom_with_bad_specializes_throws() {
        var yaml = """
            uri: "urn:bad"
            name: Bad
            version: "1"
            description: ""
            terms:
              - value: orphan
                label: Orphan
                specializes: [nonexistent]
            """;
        var registrar = new ClasspathYamlVocabularyRegistrar();
        assertThatThrownBy(() -> registrar.loadFrom(
            new java.io.ByteArrayInputStream(yaml.getBytes())))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("nonexistent");
    }
}
```

- [ ] **Step 3: Implement ClasspathYamlVocabularyRegistrar**

```java
package io.casehub.eidos.core.vocabulary;

import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.eidos.api.DataDrivenTerm;
import io.casehub.eidos.api.DataDrivenVocabulary;
import io.casehub.eidos.api.VocabularyTerm;
import io.casehub.eidos.api.spi.DataDrivenVocabularyRegistrar;
import io.casehub.yaml.jackson.YamlMappers;

import java.io.IOException;
import java.io.InputStream;
import java.net.URL;
import java.util.ArrayList;
import java.util.Enumeration;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

public class ClasspathYamlVocabularyRegistrar implements DataDrivenVocabularyRegistrar {

    private static final String RESOURCE_DIR = "META-INF/eidos/vocabularies/";
    private static final ObjectMapper YAML_MAPPER = YamlMappers.create()
        .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, true);

    @Override
    public List<DataDrivenVocabulary> vocabularies() {
        // Scan for all YAML files under META-INF/eidos/vocabularies/
        // Using the resource discovery pattern from ClasspathYamlDescriptorRegistrar
        // but scanning for all .yaml files in the directory
        var result = new ArrayList<DataDrivenVocabulary>();
        try {
            var urls = Thread.currentThread().getContextClassLoader()
                .getResources(RESOURCE_DIR);
            while (urls.hasMoreElements()) {
                // Each URL points to a directory; scan for .yaml files
                // Use a different approach: scan well-known resource paths
            }
        } catch (IOException e) {
            throw new IllegalStateException("Failed to scan " + RESOURCE_DIR, e);
        }
        // Alternative: use a single well-known path per vocabulary
        // or scan all resources matching the pattern
        // For simplicity, scan a manifest or use individual resource paths
        // Implementation: iterate classpath resources ending in .yaml
        try {
            var classLoader = Thread.currentThread().getContextClassLoader();
            // Java classloaders can't list directory contents easily.
            // Use the same pattern as descriptors: each vocab is a separate resource.
            // Consumers put their vocab files at META-INF/eidos/vocabularies/<name>.yaml
            // We discover them via a classpath index or explicit naming.
            // For now: scan using ClassGraph or similar, OR use a different approach:
            // each JAR registers a vocab resource path in META-INF/eidos/vocabularies.index
            // Actually, simplest approach: use getResources() with known filenames,
            // or iterate the classpath. Let's use a practical pattern:
            // enumerate all resources under the directory.

            // Use Java SPI-style: list resources. Since we can't list directory contents
            // portably, we'll scan for each .yaml file the classloader can find.
            // Practical approach: use ClassPathHelper or iterate known pattern.
            // For Quarkus: Jandex-based scanning happens at build time.
            // For runtime: use Thread.currentThread().getContextClassLoader()

            // Simplest robust approach: each vocabulary YAML adds itself to
            // META-INF/eidos/vocabularies/index listing its filenames.
            // OR: we just look for all resources matching the path pattern.

            // Actually: Java's getResources() returns exact path matches.
            // We need a specific filename. Let's scan for each yaml file
            // by using a known list or scanning jars.
            // For Quarkus apps, the build step can discover these.
            // For now, support a simple index-free approach:
            // iterate over all resources discovered by getResources()
            // for specific filenames. But this requires knowing the filenames.

            // Best pragmatic approach for classpath scanning without build-time:
            // Use the same pattern as ServiceLoader — a text file listing vocab files.
            // OR: simply load a fixed path like "META-INF/eidos/vocabularies.yaml"
            // that can contain multiple vocabs (like descriptors.yaml contains
            // multiple descriptors).

            // Wait — looking at ClasspathYamlTemplateRegistrar, it uses a SINGLE
            // resource path: META-INF/eidos/templates.yaml. Multiple JARs can each
            // have their own copy, and getResources() returns all of them.
            // Do the same: META-INF/eidos/vocabularies.yaml, each file contains
            // one vocabulary definition. Multiple JARs = multiple vocabs.

            // This is simpler and follows the existing pattern exactly.
        } catch (Exception e) {
            throw new IllegalStateException("Scan failed", e);
        }
        return List.copyOf(result);
    }

    public DataDrivenVocabulary loadFrom(InputStream yaml) {
        // Parse YAML, resolve specializes references, return DataDrivenVocabulary
        if (yaml == null) return null;
        try {
            var raw = YAML_MAPPER.readValue(yaml, VocabularyYaml.class);
            return resolveAndBuild(raw);
        } catch (IOException e) {
            throw new IllegalStateException("Failed to parse vocabulary YAML: " + e.getMessage(), e);
        }
    }

    private DataDrivenVocabulary resolveAndBuild(VocabularyYaml raw) {
        // First pass: create terms without resolved parents
        var termsByValue = new LinkedHashMap<String, DataDrivenTerm>();
        var rawTerms = raw.terms != null ? raw.terms : List.<TermYaml>of();

        // Build unresolved terms first
        for (var t : rawTerms) {
            var term = new DataDrivenTerm(t.value, t.label, t.description,
                t.aliases != null ? t.aliases : List.of(),
                t.keywords != null ? t.keywords : List.of(),
                List.of()); // empty parents for now
            termsByValue.put(t.value, term);
        }

        // Second pass: resolve specializes and rebuild with parents
        var resolved = new ArrayList<DataDrivenTerm>();
        for (var t : rawTerms) {
            List<VocabularyTerm> parents = List.of();
            if (t.specializes != null && !t.specializes.isEmpty()) {
                var parentList = new ArrayList<VocabularyTerm>();
                for (var parentValue : t.specializes) {
                    var parent = termsByValue.get(parentValue);
                    if (parent == null) {
                        throw new IllegalArgumentException(
                            "Unresolvable specializes reference '" + parentValue
                            + "' in term '" + t.value + "' of vocabulary '" + raw.uri + "'");
                    }
                    parentList.add(parent);
                }
                parents = List.copyOf(parentList);
            }
            resolved.add(new DataDrivenTerm(t.value, t.label, t.description,
                t.aliases != null ? t.aliases : List.of(),
                t.keywords != null ? t.keywords : List.of(),
                parents));
        }

        return new DataDrivenVocabulary(raw.uri, raw.name, raw.version, raw.description, resolved);
    }

    // Jackson YAML binding classes
    static class VocabularyYaml {
        public String uri, name, version, description;
        public List<TermYaml> terms;
    }

    static class TermYaml {
        public String value, label, description;
        public List<String> aliases, keywords, specializes;
    }
}
```

**Resource path:** `META-INF/eidos/vocabularies.yaml` (single file per JAR, same pattern as `META-INF/eidos/descriptors.yaml` and `META-INF/eidos/templates.yaml`). The `vocabularies()` method uses `getResources("META-INF/eidos/vocabularies.yaml")` and loads from each URL. Each file contains one vocabulary definition. Multiple JARs can each provide their own `vocabularies.yaml` — `getResources()` discovers all of them.

Update the constant to `private static final String RESOURCE_PATH = "META-INF/eidos/vocabularies.yaml";` and the `vocabularies()` method to iterate URLs from `getResources(RESOURCE_PATH)`, calling `loadFrom(stream)` for each.

- [ ] **Step 4: Run registrar tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=ClasspathYamlVocabularyRegistrarTest`
Expected: PASS

- [ ] **Step 5: Add CDI producer in EidosCoreProducer**

```java
@Produces @ApplicationScoped
DataDrivenVocabularyRegistrar classpathYamlVocabularyRegistrar() {
    return new ClasspathYamlVocabularyRegistrar();
}
```

Add import for `ClasspathYamlVocabularyRegistrar` and `DataDrivenVocabularyRegistrar`.

- [ ] **Step 6: Commit**

```bash
git add eidos-core/ runtime/
git commit -m "feat(core): add ClasspathYamlVocabularyRegistrar for YAML-driven vocabularies

Refs #199"
```

### Task 5: JSON Schema + Epistemic Domain Vocabulary YAML + Integration Test

**Files:**
- Create: `api/src/main/resources/META-INF/eidos/vocabulary.schema.json`
- Create: `vocab/src/main/resources/META-INF/eidos/vocabularies.yaml` (epistemic-domains vocabulary)
- Test: `vocab/src/test/java/io/casehub/eidos/vocab/EpistemicDomainVocabTest.java`

**Interfaces:**
- Consumes: `ClasspathYamlVocabularyRegistrar` (from core), `CdiVocabularyRegistry` (from runtime), `DomainKeywords` (from api)
- Produces: `vocabulary.schema.json`, `epistemic-domains` vocabulary with ~20 terms

- [ ] **Step 1: Create vocabulary.schema.json**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://casehub.io/eidos/vocabulary.schema.json",
  "title": "Eidos Vocabulary Definition",
  "description": "Schema for META-INF/eidos/vocabularies.yaml — data-driven vocabulary definitions for CaseHub Eidos",
  "type": "object",
  "required": ["uri", "terms"],
  "additionalProperties": false,
  "properties": {
    "uri": {
      "type": "string",
      "description": "Unique vocabulary URI"
    },
    "name": {
      "type": "string",
      "description": "Human-readable vocabulary name"
    },
    "version": {
      "type": "string",
      "description": "Vocabulary version"
    },
    "description": {
      "type": "string",
      "description": "What this vocabulary covers"
    },
    "terms": {
      "type": "array",
      "minItems": 1,
      "items": { "$ref": "#/$defs/Term" }
    }
  },
  "$defs": {
    "Term": {
      "type": "object",
      "required": ["value"],
      "additionalProperties": false,
      "properties": {
        "value": {
          "type": "string",
          "description": "Primary term identifier"
        },
        "label": {
          "type": "string",
          "description": "Human-readable display name"
        },
        "description": {
          "type": "string",
          "description": "What this term means"
        },
        "aliases": {
          "type": "array",
          "items": { "type": "string" },
          "description": "Alternative names for this term"
        },
        "keywords": {
          "type": "array",
          "items": { "type": "string" },
          "description": "Associated concepts for text matching and LLM understanding"
        },
        "specializes": {
          "type": "array",
          "items": { "type": "string" },
          "description": "Parent term values within this vocabulary (hierarchy)"
        }
      }
    }
  }
}
```

- [ ] **Step 2: Create epistemic-domains vocabulary YAML**

File `vocab/src/main/resources/META-INF/eidos/vocabularies.yaml` — the full ~20-term vocabulary with all keywords as specified in the design spec. Include the `yaml-language-server` schema comment at the top.

- [ ] **Step 3: Write integration test**

```java
package io.casehub.eidos.vocab;

import io.casehub.eidos.api.DataDrivenTerm;
import io.casehub.eidos.api.DomainKeywords;
import io.casehub.eidos.api.MatchDegree;
import io.casehub.eidos.core.vocabulary.ClasspathYamlVocabularyRegistrar;
import io.casehub.eidos.runtime.vocabulary.CdiVocabularyRegistry;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.*;

class EpistemicDomainVocabTest {

    static final String URI = "urn:casehub:vocab:epistemic-domain";
    static CdiVocabularyRegistry registry;

    @BeforeAll static void setup() {
        registry = new CdiVocabularyRegistry();
        // Register enum vocabs first (existing registrars needed for CdiVocabularyRegistry)
        // Then register data-driven vocabs
        var registrar = new ClasspathYamlVocabularyRegistrar();
        for (var vocab : registrar.vocabularies()) {
            registry.register(vocab.uri(), vocab.name(), vocab.version(),
                vocab.description(), vocab.terms());
        }
    }

    @Test void vocabulary_is_registered() {
        assertThat(registry.isRegistered(URI)).isTrue();
    }

    @Test void has_approximately_20_terms() {
        var terms = registry.allTerms(URI);
        assertThat(terms).hasSizeGreaterThanOrEqualTo(18);
        assertThat(terms).hasSizeLessThanOrEqualTo(25);
    }

    @Test void java_specializes_software_engineering() {
        assertThat(registry.subsumes(URI, "software-engineering", "java")).isTrue();
    }

    @Test void java_has_keywords() {
        var term = registry.resolve(URI, "java");
        assertThat(term).isPresent();
        assertThat(term.get().keywords()).contains("jvm", "bytecode");
    }

    @Test void expanded_keywords_include_ancestors() {
        var expanded = DomainKeywords.expanded(registry, URI, "java");
        assertThat(expanded).contains("jvm"); // java's own
        assertThat(expanded).contains("code"); // software-engineering's
    }

    @Test void owlsmx_matching_works() {
        // software-engineering declared, java requested → Plugin
        var match = registry.match(URI, "software-engineering", "java");
        assertThat(match).isInstanceOf(MatchDegree.Plugin.class);
    }

    @Test void static_analysis_specializes_analysis() {
        assertThat(registry.subsumes(URI, "analysis", "static-analysis")).isTrue();
    }

    @Test void top_level_domains_have_no_parents() {
        assertThat(registry.ancestors(URI, "software-engineering")).isEmpty();
        assertThat(registry.ancestors(URI, "analysis")).isEmpty();
        assertThat(registry.ancestors(URI, "security")).isEmpty();
    }

    @Test void vocabulary_metadata_populated() {
        var info = registry.vocabularyMetadata(URI);
        assertThat(info).isPresent();
        assertThat(info.get().name()).isEqualTo("Epistemic Domain Taxonomy");
    }
}
```

- [ ] **Step 4: Run integration test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl vocab -Dtest=EpistemicDomainVocabTest`
Expected: PASS

- [ ] **Step 5: Add WARN log for unmatched epistemicDomains keys**

In `eidos-core/src/main/java/io/casehub/eidos/core/registrar/DescriptorCollector.java`, in the `collectAndValidate()` method, after existing validation, add a check: if the epistemic domain vocabulary is registered (`vocabRegistry.isRegistered("urn:casehub:vocab:epistemic-domain")`), then for each descriptor's capabilities, for each `epistemicDomains` key, check `vocabRegistry.resolve("urn:casehub:vocab:epistemic-domain", key)` — if empty, log at WARN: `"Descriptor '{}' has epistemicDomains key '{}' not found in epistemic domain vocabulary"`.

Guard with `isRegistered()` first — if eidos-vocab is not on the classpath, skip entirely (avoids useless WARN spam).

Add a test in `DescriptorCollectorTest` verifying the WARN is logged for an unknown domain key when the vocabulary is registered.

- [ ] **Step 6: Run full project build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS — all modules compile, all tests pass

- [ ] **Step 6: Commit**

```bash
git add api/src/main/resources/ vocab/
git commit -m "feat(vocab): add epistemic domain vocabulary with ~20 terms, keywords, and hierarchy

Closes #199"
```

---

## References

- [2026-10-18-domain-vocab-keywords-design.md] — design spec this plan implements
- [decisions.md] — 14 validated design decisions
- VocabularyTerm.java:14-71 — existing interface, keywords() addition target
- VocabularyRegistry.java:8-115 — existing SPI, register() and vocabularyMetadata() changes
- CdiVocabularyRegistry.java:38-573 — production implementation, URI-keyed refactor target
- ClasspathYamlDescriptorRegistrar.java:20-94 — YAML registrar pattern
- ClasspathYamlTemplateRegistrar.java:16-70 — YAML registrar pattern
- EidosCoreProducer.java:46-151 — CDI producer, new producer method target
- AgentDescriptorValidator.java:129 — isBanned() character validation for DataDrivenTerm
- CasehubCapabilityTerm.java — hierarchical vocabulary pattern
- GitHub #199 — focal issue
- GitHub #198 — parent epic
