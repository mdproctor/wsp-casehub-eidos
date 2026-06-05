# Vocabulary System Enum Redesign + DispositionAxis
**Issue:** eidos#40  
**Date:** 2026-06-05  
**Status:** approved

---

## Problem

`VocabularyRegistry.equivalentValues(String fromVocab, String value, String toVocab)` cannot
support the DISC-as-disposition-vocabulary architecture (ADR 0003) because a single DISC type
maps to *different* Conscientiousness values depending on which `AgentDisposition` axis is being
resolved. There is no axis parameter, so there is no way to distinguish.

Separately: the vocabulary system was built with `Vocabulary` and `VocabularyTerm` as stringly-typed
records. Vocabulary terms are fixed at compile time — even consumer-defined vocabularies (devtown's
`"planner"`/`"reviewer"`) are defined in code, not discovered at runtime from external sources.
Strings give up the compile-time safety that enums would provide: invalid term names, invalid axis
names, and cross-vocabulary references that silently return empty sets rather than failing at compile
time.

This issue redesigns the vocabulary system from String-based records to an enum-based interface,
adding `DispositionAxis` as a typed axis key, and extends `VocabularyRegistry` with typed generic
methods alongside the string-based methods needed for runtime descriptor lookups.

---

## Deleted Types

- `Vocabulary` record — removed entirely; the enum class IS the vocabulary
- `VocabularyTerm` record — removed; replaced by the `VocabularyTerm` interface

---

## New Types in `casehub-eidos-api`

### `VocabularyTerm` (interface — replaces the deleted record of the same name)

```java
package io.casehub.eidos.api;

public interface VocabularyTerm {
    // Vocabulary-level (same for all constants in an enum)
    String vocabUri();
    default String vocabName()    { return ""; }
    default String vocabVersion() { return ""; }

    // Term-level
    String value();
    String label();
    default String description()   { return ""; }
    default List<String> aliases() { return List.of(); }

    // Axis-unaware cross-vocab equivalence: target vocab class → equivalent constant
    default Map<Class<?>, VocabularyTerm> exactMatches() { return Map.of(); }

    // Axis-aware cross-vocab equivalence: target vocab class → axis → equivalent constant
    default Map<Class<?>, Map<DispositionAxis, VocabularyTerm>> axisExactMatches() {
        return Map.of();
    }
}
```

Vocabularies are Java enums implementing this interface. Each enum constant IS a term. The enum
class IS the vocabulary. `vocabUri()` returns the same value for all constants in a given enum.

Cross-vocabulary references in `exactMatches()` and `axisExactMatches()` use the target enum
class as the key — not a URI string. This means references are compile-time constants: renaming
a term constant is a compiler error at every reference site.

### `DispositionAxis` (enum)

```java
package io.casehub.eidos.api;

public enum DispositionAxis {
    SOCIAL_ORIENT,
    RULE_FOLLOWING,
    RISK_APPETITE,
    AUTONOMY;
}
```

Enumerates every valid axis of `AgentDisposition`. When eidos#38 adds `conflictMode` as a 5th
axis, `CONFLICT_MODE` is added here and every `axisExactMatches()` implementation that covers
the target vocabulary is immediately visible as incomplete.

### `VocabularyRegistrar` (SPI — `api/spi/`)

```java
package io.casehub.eidos.api.spi;

@FunctionalInterface
public interface VocabularyRegistrar {
    Class<? extends Enum<? extends VocabularyTerm>> vocabulary();
}
```

Implemented by `@ApplicationScoped` CDI beans in vocab modules. `CdiVocabularyRegistry` injects
`Instance<VocabularyRegistrar>` and calls `register()` for each at startup. Replaces the old
`@Produces Vocabulary` pattern.

---

## Updated `VocabularyRegistry` SPI

Two parallel APIs on the same interface:

- **String-based** — for runtime values: `AgentDescriptor.dispositionVocabulary` is a URI string;
  `AgentDisposition.socialOrient` is a string. These paths are needed when resolving descriptors
  loaded from storage where the enum class is not statically known.
- **Typed** — for compile-time-known vocabulary classes. Returns typed enum constants; no casting.

```java
public interface VocabularyRegistry {

    // Registration
    <T extends Enum<T> & VocabularyTerm> void register(Class<T> vocab);
    boolean isRegistered(String vocabUri);

    // String-based resolution (runtime values / unknown vocab class)
    Optional<? extends VocabularyTerm> resolve(String vocabUri, String value);
    Set<String> equivalentValues(String fromUri, String value, String toUri);
    Set<String> equivalentValues(String fromUri, String value, String toUri, DispositionAxis axis);

    // Typed resolution (compile-time-known vocab class)
    <T extends Enum<T> & VocabularyTerm>
        Optional<T> resolve(Class<T> vocab, String value);

    <S extends Enum<S> & VocabularyTerm, T extends Enum<T> & VocabularyTerm>
        Optional<T> equivalentValues(S from, Class<T> targetVocab);

    <S extends Enum<S> & VocabularyTerm, T extends Enum<T> & VocabularyTerm>
        Optional<T> equivalentValues(S from, Class<T> targetVocab, DispositionAxis axis);
}
```

The old `register(Vocabulary vocabulary)` and `find(String uri)` methods are removed.

The typed `equivalentValues` methods return `Optional<T>` (not `Set<T>`) because a source
constant maps to at most one constant per target vocabulary. The string-based methods retain
`Set<String>` because alias collisions across multiple terms remain possible.

The typed equivalentValues implementations are thin — they delegate directly to the source
constant's own interface methods. The registry state is only needed for the string-based path.

---

## `CdiVocabularyRegistry` Implementation

**Internal state:**

```
Map<String, Class<? extends Enum<?>>> byUri      // vocabUri → enum class
Map<Class<?>, Map<String, VocabularyTerm>> byClass // class → (value + aliases) → constant
```

**`@PostConstruct`:** iterates `Instance<VocabularyRegistrar>`, calls `register(vocab.vocabulary())`
for each.

**`register(Class<T>)`:**
1. Calls `T.getEnumConstants()` to get all constants.
2. Asserts all constants return the same `vocabUri()` — throws `IllegalArgumentException` if not.
3. Builds the value+alias lookup map; throws on duplicate primary value.
4. Stores in both internal maps.

**Typed equivalentValues** (barely touch registry state):
```java
public <S extends Enum<S> & VocabularyTerm, T extends Enum<T> & VocabularyTerm>
Optional<T> equivalentValues(S from, Class<T> targetVocab) {
    return Optional.ofNullable((T) from.exactMatches().get(targetVocab));
}

public <S extends Enum<S> & VocabularyTerm, T extends Enum<T> & VocabularyTerm>
Optional<T> equivalentValues(S from, Class<T> targetVocab, DispositionAxis axis) {
    var axisMap = from.axisExactMatches().get(targetVocab);
    return axisMap == null ? Optional.empty() : Optional.ofNullable((T) axisMap.get(axis));
}
```

**String-based equivalentValues** — routes through internal maps to find the source constant,
then delegates to the typed path and extracts `.value()`:
```java
public Set<String> equivalentValues(String fromUri, String value, String toUri) {
    var sourceClass = byUri.get(fromUri);
    if (sourceClass == null) return Set.of();
    var term = byClass.get(sourceClass).get(value);
    if (term == null) return Set.of();
    var targetClass = byUri.get(toUri);
    if (targetClass == null) return Set.of();
    var match = term.exactMatches().get(targetClass);
    return match != null ? Set.of(match.value()) : Set.of();
}

public Set<String> equivalentValues(String fromUri, String value, String toUri, DispositionAxis axis) {
    var sourceClass = byUri.get(fromUri);
    if (sourceClass == null) return Set.of();
    var term = byClass.get(sourceClass).get(value);
    if (term == null) return Set.of();
    var targetClass = byUri.get(toUri);
    if (targetClass == null) return Set.of();
    var axisMap = term.axisExactMatches().get(targetClass);
    if (axisMap == null) return Set.of();
    var match = axisMap.get(axis);
    return match != null ? Set.of(match.value()) : Set.of();
}
```

---

## Vocabulary Enums (casehub-eidos-vocab)

Three existing producer classes replaced by three enum + three registrar classes.

### `ConscientiousnessTerm` enum

12 terms across 4 disposition axes. The axis grouping is informational (comments / structure)
but not enforced by the type — `DispositionAxis` is used in `axisExactMatches` keys by DISC,
not by Conscientiousness itself.

```java
public enum ConscientiousnessTerm implements VocabularyTerm {
    // ruleFollowing axis
    STRICT      ("strict",      "Strict Rule Following",   "Follows rules rigidly",      List.of("rule-bound", "compliant")),
    PRINCIPLED  ("principled",  "Principled",              "Follows intent of rules",    List.of("values-based")),
    FLEXIBLE    ("flexible",    "Flexible",                "Adapts rules to context",    List.of("adaptive", "pragmatic")),
    // riskAppetite axis
    CONSERVATIVE("conservative","Conservative Risk",       "Avoids uncertainty",         List.of("risk-averse", "cautious")),
    MEASURED    ("measured",    "Measured Risk",           "Balances risk and reward",   List.of("balanced")),
    BOLD        ("bold",        "Bold Risk",               "Accepts uncertainty for reward", List.of("risk-tolerant", "adventurous")),
    // socialOrient axis
    COLLABORATIVE("collaborative","Collaborative",         "Works with others by default",  List.of("team-oriented", "cooperative")),
    INDEPENDENT  ("independent","Independent",             "Works alone by preference",  List.of("autonomous-social", "self-directed")),
    FACILITATIVE ("facilitative","Facilitative",           "Enables others to work",     List.of("supportive", "enabling")),
    // autonomy axis
    DIRECTED     ("directed",   "Directed Autonomy",      "Follows explicit instructions", List.of("instruction-following")),
    SEMI_AUTONOMOUS("semi-autonomous","Semi-Autonomous",   "Acts within defined boundaries",List.of("bounded-autonomy")),
    AUTONOMOUS   ("autonomous", "Autonomous",             "Acts on own judgment",        List.of("self-governing", "agentic"));

    // standard constructor + vocabUri()
    public static final String URI = "urn:casehub:vocab:conscientiousness";
}
```

### `SvoTerm` enum

3 terms with `exactMatches()` to CasehubSlot (bidirectional with `CasehubSlotTerm`).

### `CasehubSlotTerm` enum

3 terms with `exactMatches()` back to SVO.

### Companion registrars

Each vocab enum gets a thin companion:

```java
@ApplicationScoped
public class ConscientiousnessVocabRegistrar implements VocabularyRegistrar {
    @Override public Class<ConscientiousnessTerm> vocabulary() {
        return ConscientiousnessTerm.class;
    }
}
```

---

## Test Coverage

### `CdiVocabularyRegistryTest` (runtime module — `@QuarkusTest`)

Rewritten against the new typed API. Covers:
- Programmatic `register()` of a test enum
- `isRegistered()` true/false
- `resolve(Class<T>, String value)` by primary value and by alias
- `resolve(String uri, String value)` string-based path
- `equivalentValues(S from, Class<T>)` typed — returns correct constant
- `equivalentValues(S from, Class<T>, DispositionAxis)` typed — returns axis-scoped constant
- `equivalentValues(String, String, String)` string-based
- `equivalentValues(String, String, String, DispositionAxis)` string-based
- Missing vocab / missing value / missing axis → empty / empty optional
- Startup validation: inconsistent `vocabUri()` across constants → `IllegalArgumentException`
- Startup validation: duplicate primary value → `IllegalArgumentException`

### `CrossVocabularyDiscoveryTest` (examples module — `@QuarkusTest`)

Updated: `find(URI)` → `isRegistered(URI)`. Typed cross-vocab assertions:
```java
Optional<CasehubSlotTerm> result =
    vocabRegistry.equivalentValues(SvoTerm.EVALUATOR, CasehubSlotTerm.class);
assertThat(result).contains(CasehubSlotTerm.REVIEWER);
```

### `DispositionVocabularyTest` (examples module — `@QuarkusTest`)

Updated: typed `resolve(ConscientiousnessTerm.class, "strict")` replaces string-based resolve.

### Three vocab tests

Updated to exercise the enum constants directly.

---

## Scope Summary

| File | Action |
|------|--------|
| `api/Vocabulary.java` | **Delete** |
| `api/VocabularyTerm.java` | **Delete record; create interface of same name** |
| `api/DispositionAxis.java` | **Create** |
| `api/spi/VocabularyRegistrar.java` | **Create** |
| `api/VocabularyRegistry.java` | **Breaking rewrite** |
| `runtime/vocabulary/CdiVocabularyRegistry.java` | **Full rewrite** |
| `runtime/vocabulary/CdiVocabularyRegistryTest.java` | **Full rewrite** |
| `vocab/ConscientiousnessVocabularyProducer.java` | **Delete; create ConscientiousnessTerm + ConscientiousnessVocabRegistrar** |
| `vocab/SvoVocabularyProducer.java` | **Delete; create SvoTerm + SvoVocabRegistrar** |
| `vocab/CasehubSlotVocabularyProducer.java` | **Delete; create CasehubSlotTerm + CasehubSlotVocabRegistrar** |
| `vocab/ConscientiousnessVocabularyTest.java` | **Update** |
| `vocab/SvoVocabularyTest.java` | **Update** |
| `vocab/CasehubSlotVocabularyTest.java` | **Update** |
| `examples/CrossVocabularyDiscoveryTest.java` | **Update** |
| `examples/DispositionVocabularyTest.java` | **Update** |

No cross-repo changes. Nothing outside `casehub-eidos` uses `Vocabulary`, `VocabularyTerm`, or
`VocabularyRegistry`. All changes are self-contained within the eidos module tree.

---

## Out of Scope

- `DiscTerm` and `BelbinTerm` enums — eidos#26 (DISC/Belbin vocabulary module). This spec
  establishes the foundation they build on.
- `conflictMode` as 5th `AgentDisposition` axis — eidos#38. When it lands, `CONFLICT_MODE`
  is added to `DispositionAxis`.
- JPA persistence of vocabularies — currently all vocabularies are CDI-discovered in-memory.
  Persistence is out of scope for this and is not tracked.
