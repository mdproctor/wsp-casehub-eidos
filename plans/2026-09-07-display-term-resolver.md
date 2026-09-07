# DisplayTermResolver Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #170 — feat: DisplayTermResolver — vocabulary-based display label resolution (pre-platform)
**Issue group:** #170

**Goal:** Build a DisplayTermResolver SPI that resolves vocabulary term values to display labels, with cross-vocabulary swap support for domain terminology aliasing.

**Architecture:** SPI interface in eidos-api (Tier 1, pure Java) with DefaultDisplayTermResolver @DefaultBean in eidos-runtime delegating to VocabularyRegistry. Three resolution modes: direct label lookup, cross-vocabulary swap (axis-unaware via exactMatch), and axis-aware swap (via axisExactMatch for disposition terms).

**Tech Stack:** Java 21, Quarkus 3.32.2, existing VocabularyRegistry infrastructure

## Global Constraints

- eidos-api stays Tier 1 (pure Java, no CDI/platform-api dependency)
- `@DefaultBean` pattern for the runtime implementation (consumer-displaceable)
- No tenancy-based vocabulary derivation — vocab context comes from domain objects
- Auto-discovery (null sourceVocabUri) is best-effort — explicit sourceVocabUri preferred

---

## Batch 1: DisplayTermResolver SPI + implementation

### Task 1: DisplayTermResolver SPI in eidos-api + DefaultDisplayTermResolver in eidos-runtime with tests

**Files:**
- Create: `api/src/main/java/io/casehub/eidos/api/DisplayTermResolver.java`
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/display/DefaultDisplayTermResolver.java`
- Create: `runtime/src/test/java/io/casehub/eidos/runtime/display/DefaultDisplayTermResolverTest.java`

**Interfaces:**
- Consumes: `VocabularyRegistry.resolve(vocabUri, value)`, `VocabularyRegistry.equivalentValues(fromUri, value, toUri)`, `VocabularyRegistry.equivalentValues(fromUri, value, toUri, axis)`, `VocabularyRegistry.registeredUris()`
- Produces: `DisplayTermResolver.resolveLabel(value, sourceVocabUri, targetVocabUri, axis)` + default overloads

- [ ] **Step 1: Create the SPI interface in eidos-api**

Create `api/src/main/java/io/casehub/eidos/api/DisplayTermResolver.java`:

```java
package io.casehub.eidos.api;

public interface DisplayTermResolver {

    String resolveLabel(String value, String sourceVocabUri,
                        String targetVocabUri, DispositionAxis axis);

    default String resolveLabel(String value, String sourceVocabUri,
                                String targetVocabUri) {
        return resolveLabel(value, sourceVocabUri, targetVocabUri, null);
    }

    default String resolveLabel(String value, String sourceVocabUri) {
        return resolveLabel(value, sourceVocabUri, null, null);
    }
}
```

- [ ] **Step 2: Verify api module compiles**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api`
Expected: BUILD SUCCESS

- [ ] **Step 3: Write the test class with test vocabularies and all test cases**

Create `runtime/src/test/java/io/casehub/eidos/runtime/display/DefaultDisplayTermResolverTest.java`:

```java
package io.casehub.eidos.runtime.display;

import io.casehub.eidos.api.*;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class DefaultDisplayTermResolverTest {

    @Inject
    VocabularyRegistry registry;

    @Inject
    DisplayTermResolver resolver;

    @VocabularyMetadata(uri = "urn:test:devtown-roles", name = "Devtown Roles", version = "1.0")
    enum DevtownRole implements VocabularyTerm {
        PLANNER("planner", "Planner"),
        REVIEWER("reviewer", "Reviewer"),
        OBSERVER("observer", "Observer") {
            @Override public Optional<VocabularyTerm> exactMatch(Class<?> target) {
                if (target == GastownRole.class) return Optional.of(GastownRole.WITNESS);
                return Optional.empty();
            }
        };
        final String v, l;
        DevtownRole(String v, String l) { this.v = v; this.l = l; }
        @Override public String value() { return v; }
        @Override public String label() { return l; }
    }

    @VocabularyMetadata(uri = "urn:test:gastown-roles", name = "Gastown Roles", version = "1.0")
    enum GastownRole implements VocabularyTerm {
        WITNESS("witness", "Witness") {
            @Override public Optional<VocabularyTerm> exactMatch(Class<?> target) {
                if (target == DevtownRole.class) return Optional.of(DevtownRole.OBSERVER);
                return Optional.empty();
            }
        },
        POLECAT("polecat", "Polecat"),
        DEACON("deacon", "Deacon");
        final String v, l;
        GastownRole(String v, String l) { this.v = v; this.l = l; }
        @Override public String value() { return v; }
        @Override public String label() { return l; }
    }

    @VocabularyMetadata(uri = "urn:test:axis-vocab-a", name = "Axis Vocab A", version = "1.0")
    enum AxisVocabA implements VocabularyTerm {
        BOLD("bold", "Bold") {
            @Override public Optional<VocabularyTerm> axisExactMatch(Class<?> target, DispositionAxis axis) {
                if (target == AxisVocabB.class && axis == DispositionAxis.RISK_APPETITE)
                    return Optional.of(AxisVocabB.ADVENTUROUS);
                return Optional.empty();
            }
        };
        final String v, l;
        AxisVocabA(String v, String l) { this.v = v; this.l = l; }
        @Override public String value() { return v; }
        @Override public String label() { return l; }
    }

    @VocabularyMetadata(uri = "urn:test:axis-vocab-b", name = "Axis Vocab B", version = "1.0")
    enum AxisVocabB implements VocabularyTerm {
        ADVENTUROUS("adventurous", "Adventurous");
        final String v, l;
        AxisVocabB(String v, String l) { this.v = v; this.l = l; }
        @Override public String value() { return v; }
        @Override public String label() { return l; }
    }

    @BeforeEach
    void ensureRegistered() {
        if (!registry.isRegistered("urn:test:devtown-roles"))
            registry.register(DevtownRole.class);
        if (!registry.isRegistered("urn:test:gastown-roles"))
            registry.register(GastownRole.class);
        if (!registry.isRegistered("urn:test:axis-vocab-a"))
            registry.register(AxisVocabA.class);
        if (!registry.isRegistered("urn:test:axis-vocab-b"))
            registry.register(AxisVocabB.class);
    }

    @Test
    void direct_resolution_returns_label() {
        assertThat(resolver.resolveLabel("witness", "urn:test:gastown-roles"))
            .isEqualTo("Witness");
    }

    @Test
    void direct_resolution_unknown_value_returns_raw() {
        assertThat(resolver.resolveLabel("unknown-role", "urn:test:gastown-roles"))
            .isEqualTo("unknown-role");
    }

    @Test
    void direct_resolution_unknown_vocab_returns_raw() {
        assertThat(resolver.resolveLabel("witness", "urn:nonexistent"))
            .isEqualTo("witness");
    }

    @Test
    void cross_vocab_swap_returns_target_label() {
        assertThat(resolver.resolveLabel("observer", "urn:test:devtown-roles",
            "urn:test:gastown-roles"))
            .isEqualTo("Witness");
    }

    @Test
    void cross_vocab_swap_no_match_falls_back_to_source_label() {
        assertThat(resolver.resolveLabel("planner", "urn:test:devtown-roles",
            "urn:test:gastown-roles"))
            .isEqualTo("Planner");
    }

    @Test
    void cross_vocab_swap_target_not_registered_falls_back() {
        assertThat(resolver.resolveLabel("witness", "urn:test:gastown-roles",
            "urn:nonexistent"))
            .isEqualTo("Witness");
    }

    @Test
    void axis_aware_swap_returns_target_label() {
        assertThat(resolver.resolveLabel("bold", "urn:test:axis-vocab-a",
            "urn:test:axis-vocab-b", DispositionAxis.RISK_APPETITE))
            .isEqualTo("Adventurous");
    }

    @Test
    void axis_aware_swap_wrong_axis_falls_back() {
        assertThat(resolver.resolveLabel("bold", "urn:test:axis-vocab-a",
            "urn:test:axis-vocab-b", DispositionAxis.AUTONOMY))
            .isEqualTo("Bold");
    }

    @Test
    void auto_discovery_null_source_finds_term() {
        assertThat(resolver.resolveLabel("witness", null))
            .isEqualTo("Witness");
    }

    @Test
    void auto_discovery_null_source_unknown_value_returns_raw() {
        assertThat(resolver.resolveLabel("nonexistent-term", null))
            .isEqualTo("nonexistent-term");
    }

    @Test
    void same_source_and_target_returns_source_label() {
        assertThat(resolver.resolveLabel("witness", "urn:test:gastown-roles",
            "urn:test:gastown-roles"))
            .isEqualTo("Witness");
    }

    @Test
    void null_value_returns_null() {
        assertThat(resolver.resolveLabel(null, "urn:test:gastown-roles"))
            .isNull();
    }
}
```

- [ ] **Step 4: Run tests to verify they fail (DisplayTermResolver not yet implemented)**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=DefaultDisplayTermResolverTest`
Expected: FAIL — no `DisplayTermResolver` bean available

- [ ] **Step 5: Implement DefaultDisplayTermResolver**

Create `runtime/src/main/java/io/casehub/eidos/runtime/display/DefaultDisplayTermResolver.java`:

```java
package io.casehub.eidos.runtime.display;

import io.casehub.eidos.api.DispositionAxis;
import io.casehub.eidos.api.DisplayTermResolver;
import io.casehub.eidos.api.VocabularyRegistry;
import io.casehub.eidos.api.VocabularyTerm;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

@DefaultBean
@ApplicationScoped
public class DefaultDisplayTermResolver implements DisplayTermResolver {

    private final VocabularyRegistry registry;

    @Inject
    public DefaultDisplayTermResolver(VocabularyRegistry registry) {
        this.registry = registry;
    }

    @Override
    public String resolveLabel(String value, String sourceVocabUri,
                               String targetVocabUri, DispositionAxis axis) {
        if (value == null) return null;

        // Step 1: find source term
        String resolvedSourceUri = sourceVocabUri;
        VocabularyTerm sourceTerm = null;

        if (sourceVocabUri != null) {
            sourceTerm = registry.resolve(sourceVocabUri, value).orElse(null);
        } else {
            for (String uri : registry.registeredUris()) {
                var term = registry.resolve(uri, value);
                if (term.isPresent()) {
                    sourceTerm = term.get();
                    resolvedSourceUri = uri;
                    break;
                }
            }
        }

        if (sourceTerm == null) return value;

        // Step 2: resolve display label
        if (targetVocabUri == null || targetVocabUri.equals(resolvedSourceUri)) {
            return sourceTerm.label();
        }

        // Step 3: cross-vocab swap
        var targetValue = axis != null
            ? registry.equivalentValues(resolvedSourceUri, value, targetVocabUri, axis)
            : registry.equivalentValues(resolvedSourceUri, value, targetVocabUri);

        if (targetValue.isPresent()) {
            var targetTerm = registry.resolve(targetVocabUri, targetValue.get());
            if (targetTerm.isPresent()) {
                return targetTerm.get().label();
            }
            return targetValue.get();
        }

        return sourceTerm.label();
    }
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=DefaultDisplayTermResolverTest`
Expected: All 12 tests PASS

- [ ] **Step 7: Run full api + runtime test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api,runtime`
Expected: All tests PASS

- [ ] **Step 8: Commit**

```bash
git add api/src/main/java/io/casehub/eidos/api/DisplayTermResolver.java
git add runtime/src/main/java/io/casehub/eidos/runtime/display/DefaultDisplayTermResolver.java
git add runtime/src/test/java/io/casehub/eidos/runtime/display/DefaultDisplayTermResolverTest.java
git commit -m "feat(#170): add DisplayTermResolver SPI with cross-vocabulary swap

DisplayTermResolver in eidos-api with DefaultDisplayTermResolver @DefaultBean
in eidos-runtime. Resolves vocabulary term values to display labels with
cross-vocabulary swap via exactMatch/axisExactMatch. Auto-discovery when
sourceVocabUri is null. Temporary home before move to platform-api (#283).

Closes #170"
```

---

## References

- [2026-09-07-display-term-resolver-design.md] — design spec this plan implements
- [VocabularyRegistry.java:30-36] — resolve() and equivalentValues() methods
- [VocabularyTerm.java:16-17] — value() and label()
- [VocabularyTerm.java:27] — exactMatch() for axis-unaware cross-vocab
- [VocabularyTerm.java:44] — axisExactMatch() for axis-aware cross-vocab
- [CdiVocabularyRegistry.java:418-438] — resolve and equivalentValues implementations
- [DefaultCapabilityHealthTest.java:29-42] — test vocabulary enum pattern
- [GitHub #170] — focal issue
- [GitHub casehubio/platform#283] — eventual platform-api home
- [GitHub casehubio/blocks-ui#157] — org diagram consumer
