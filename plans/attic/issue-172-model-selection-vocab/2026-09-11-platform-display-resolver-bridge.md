# Platform DisplayTermResolver Bridge Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #171 — refactor: bridge DefaultDisplayTermResolver to platform DisplayTermResolver SPI
**Issue group:** #171

**Goal:** Make DefaultDisplayTermResolver implement both platform and eidos DisplayTermResolver SPIs so CDI injection works against either type.

**Architecture:** Single class implements both interfaces. Platform's `resolveLabel(String, String)` matches eidos's default overload — one implementation satisfies both. Platform's `mapTerm` methods are new additions delegating to `VocabularyRegistry.equivalentValues()`. `String mappingContext` maps to `DispositionAxis` via `jsonKey()`.

**Tech Stack:** Java 21, Quarkus 3.32.2, platform-api 0.2-SNAPSHOT

## Global Constraints

- No changes to eidos-api `DisplayTermResolver` interface
- No POM changes — platform-api is already a runtime dependency
- `mapTerm` requires explicit source/target vocab URIs (no auto-discovery)
- Unknown `mappingContext` strings fall back to axis-unaware mapping (no exception)

---

## Batch 1: Platform SPI bridge

### Task 1: Add platform DisplayTermResolver implementation + mapTerm methods + tests

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/display/DefaultDisplayTermResolver.java`
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/display/DefaultDisplayTermResolverTest.java`

**Interfaces:**
- Consumes: `VocabularyRegistry.equivalentValues(fromUri, value, toUri)`, `VocabularyRegistry.equivalentValues(fromUri, value, toUri, axis)`, `DispositionAxis.jsonKey()`
- Produces: `DefaultDisplayTermResolver` now satisfies CDI injection of both `io.casehub.platform.api.display.DisplayTermResolver` and `io.casehub.eidos.api.DisplayTermResolver`

- [ ] **Step 1: Write failing tests for mapTerm and platform SPI injection**

Add to `DefaultDisplayTermResolverTest.java` (test vocabularies already exist from #170):

```java
    @Test
    void mapTerm_cross_vocab_returns_target_value() {
        assertThat(((io.casehub.platform.api.display.DisplayTermResolver) resolver)
            .mapTerm("observer", "urn:test:devtown-roles", "urn:test:gastown-roles"))
            .contains("witness");
    }

    @Test
    void mapTerm_no_match_returns_empty() {
        assertThat(((io.casehub.platform.api.display.DisplayTermResolver) resolver)
            .mapTerm("planner", "urn:test:devtown-roles", "urn:test:gastown-roles"))
            .isEmpty();
    }

    @Test
    void mapTerm_null_value_returns_empty() {
        assertThat(((io.casehub.platform.api.display.DisplayTermResolver) resolver)
            .mapTerm(null, "urn:test:devtown-roles", "urn:test:gastown-roles"))
            .isEmpty();
    }

    @Test
    void mapTerm_with_context_axis_aware() {
        assertThat(((io.casehub.platform.api.display.DisplayTermResolver) resolver)
            .mapTerm("bold", "urn:test:axis-vocab-a", "urn:test:axis-vocab-b", "riskAppetite"))
            .contains("adventurous");
    }

    @Test
    void mapTerm_with_unknown_context_falls_back_to_axis_unaware() {
        assertThat(((io.casehub.platform.api.display.DisplayTermResolver) resolver)
            .mapTerm("bold", "urn:test:axis-vocab-a", "urn:test:axis-vocab-b", "unknown"))
            .isEmpty();
    }

    @Test
    void platform_resolveLabel_matches_eidos() {
        io.casehub.platform.api.display.DisplayTermResolver platformResolver =
            (io.casehub.platform.api.display.DisplayTermResolver) resolver;
        assertThat(platformResolver.resolveLabel("witness", "urn:test:gastown-roles"))
            .isEqualTo("Witness");
    }
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api -DskipTests && mvn test -pl runtime -Dtest=DefaultDisplayTermResolverTest`
Expected: FAIL — `DefaultDisplayTermResolver` does not implement platform `DisplayTermResolver`, cast fails

- [ ] **Step 3: Add platform SPI to implements clause and add mapTerm + parseAxis**

In `DefaultDisplayTermResolver.java`, update the class declaration and add three new methods:

```java
import java.util.Arrays;
import java.util.Optional;

@DefaultBean
@ApplicationScoped
public class DefaultDisplayTermResolver
        implements io.casehub.eidos.api.DisplayTermResolver,
                   io.casehub.platform.api.display.DisplayTermResolver {

    // ... existing fields and constructor unchanged ...

    // ... existing resolveLabel method unchanged ...

    @Override
    public Optional<String> mapTerm(String value, String sourceVocabUri,
                                     String targetVocabUri) {
        return mapTerm(value, sourceVocabUri, targetVocabUri, null);
    }

    @Override
    public Optional<String> mapTerm(String value, String sourceVocabUri,
                                     String targetVocabUri, String mappingContext) {
        if (value == null || sourceVocabUri == null || targetVocabUri == null) {
            return Optional.empty();
        }
        DispositionAxis axis = parseAxis(mappingContext);
        return axis != null
            ? registry.equivalentValues(sourceVocabUri, value, targetVocabUri, axis)
            : registry.equivalentValues(sourceVocabUri, value, targetVocabUri);
    }

    private static DispositionAxis parseAxis(String mappingContext) {
        if (mappingContext == null) return null;
        for (DispositionAxis axis : DispositionAxis.values()) {
            if (axis.jsonKey().equals(mappingContext)) return axis;
        }
        return null;
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=DefaultDisplayTermResolverTest`
Expected: All 18 tests PASS (12 existing + 6 new)

- [ ] **Step 5: Run full api + runtime test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api,runtime`
Expected: All tests PASS

- [ ] **Step 6: Commit**

```bash
git add runtime/src/main/java/io/casehub/eidos/runtime/display/DefaultDisplayTermResolver.java
git add runtime/src/test/java/io/casehub/eidos/runtime/display/DefaultDisplayTermResolverTest.java
git commit -m "feat(#171): bridge DefaultDisplayTermResolver to platform SPI

DefaultDisplayTermResolver now implements both eidos and platform
DisplayTermResolver SPIs. Adds mapTerm() for cross-vocabulary value
translation. String mappingContext maps to DispositionAxis via jsonKey().

Closes #171"
```

---

## References

- [2026-09-11-platform-display-resolver-bridge-design.md] — design spec
- [DefaultDisplayTermResolver.java] — current implementation from #170
- [DisplayTermResolver.java (platform-api)] — platform SPI from platform#283
- [DisplayTermResolver.java (eidos-api)] — eidos SPI from #170
- [DispositionAxis.java] — jsonKey() for context mapping
- [GitHub #171] — focal issue
- [GitHub casehubio/platform#283] — platform SPI (closed)
