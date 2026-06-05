# Vocabulary System Enum Redesign + DispositionAxis
**Issue:** eidos#40
**Date:** 2026-06-05
**Status:** approved (rev 2)

---

## Problem

`VocabularyRegistry.equivalentValues(String fromVocab, String value, String toVocab)` cannot
support the DISC-as-disposition-vocabulary architecture (ADR 0003) because a single DISC type
maps to *different* Conscientiousness values depending on which `AgentDisposition` axis is being
resolved. There is no axis parameter, so there is no way to distinguish.

The existing `Vocabulary` and `VocabularyTerm` records are stringly-typed throughout: vocabulary
URIs, term values, and cross-vocabulary references are all plain strings. Vocabulary terms are
fixed at compile time — even consumer-defined vocabularies (devtown's `"planner"`/`"reviewer"`)
are defined in code, not discovered from external sources at runtime. Strings give up the
compile-time safety that enums would provide.

This issue redesigns the vocabulary system from String-based records to an enum-based interface
with typed cross-vocabulary methods, adding `DispositionAxis` as a typed axis key, and extends
`VocabularyRegistry` with typed generic methods alongside the string-based methods required for
runtime descriptor lookups.

---

## End-to-End Trace

The original problem, traced through the new API:

1. Agent registered with `dispositionVocabulary = "urn:casehub:vocab:disc"` and
   `AgentDisposition.riskAppetite = "dominance"` (D type declared on risk axis).
2. casehub-engine receives a query: "find agents whose riskAppetite is equivalent to
   `ConscientiousnessTerm.BOLD` in the Conscientiousness vocabulary."
3. Engine calls the string-based API (descriptor values are strings at runtime):
   ```java
   registry.equivalentValues(
       desc.dispositionVocabulary(),       // "urn:casehub:vocab:disc"
       desc.disposition().get(DispositionAxis.RISK_APPETITE).orElseThrow(),  // "dominance"
       ConscientiousnessVocab.URI,
       DispositionAxis.RISK_APPETITE       // typed axis
   )
   // → Optional.of("bold")
   ```
4. Engine finds agents where `riskAppetite = "dominance"` (DISC) OR `riskAppetite = "bold"`
   (Conscientiousness) — both express the same behavioral profile on that axis.

The `AgentDisposition.get(DispositionAxis)` bridge (see below) is the only place the engine
needs to know which field maps to which axis. When `CONFLICT_MODE` is added to `DispositionAxis`,
that switch fails to compile, forcing the engine to handle it.

---

## Deleted Types

- `Vocabulary` record — removed entirely; the enum class IS the vocabulary
- `VocabularyTerm` record — removed; replaced by the `VocabularyTerm` interface of the same name

---

## New Types in `casehub-eidos-api`

### `@VocabularyMetadata` annotation

Vocabulary-level metadata belongs on the class, not on instances. A per-constant `vocabUri()`
would require every constant to return the same value — a constraint that must be validated at
startup rather than being structurally impossible to violate. An annotation on the enum class
eliminates the repetition and the validation entirely.

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface VocabularyMetadata {
    String uri();
    String name()    default "";
    String version() default "";
}
```

`register(Class<T>)` reads this annotation. Throws `IllegalArgumentException` if absent.

### `VocabularyTerm` interface (replaces the deleted record of the same name)

Pure term-level data. No vocabulary-level metadata — that lives on the annotation.

Cross-vocabulary methods return `Optional<VocabularyTerm>` rather than exposing a map. This
enables implementations to use an exhaustive switch on `DispositionAxis` (see `axisExactMatch`
below), which is the actual compile-time completeness gate. A `Map`-based approach silently
omits new axis values; a switch with no default fails to compile.

```java
package io.casehub.eidos.api;

public interface VocabularyTerm {
    String value();
    String label();
    default String description()   { return ""; }
    default List<String> aliases() { return List.of(); }

    /**
     * Axis-unaware cross-vocabulary equivalence.
     * Override to return the equivalent constant in {@code targetVocab}, or empty if none.
     * Implementations should check {@code targetVocab} identity and return the typed constant;
     * the registry calls {@code targetVocab.cast()} on the result.
     */
    default Optional<VocabularyTerm> exactMatch(Class<?> targetVocab) {
        return Optional.empty();
    }

    /**
     * Axis-aware cross-vocabulary equivalence.
     * Implementations MUST use an exhaustive switch on {@code axis} (no default branch) so
     * that adding a new {@link DispositionAxis} value causes a compile error in every
     * implementation that covers a given target vocabulary.
     */
    default Optional<VocabularyTerm> axisExactMatch(Class<?> targetVocab, DispositionAxis axis) {
        return Optional.empty();
    }
}
```

### `DispositionAxis` enum

```java
package io.casehub.eidos.api;

public enum DispositionAxis {
    SOCIAL_ORIENTATION,
    RULE_FOLLOWING,
    RISK_APPETITE,
    AUTONOMY;
}
```

When eidos#38 adds `conflictMode` as a 5th axis, `CONFLICT_MODE` is added here. Every exhaustive
switch on `DispositionAxis` in vocabulary implementations — and the `AgentDisposition.get()`
method — fails to compile until it covers the new value.

### `VocabularyRegistrar` SPI (`api/spi/`)

```java
package io.casehub.eidos.api.spi;

@FunctionalInterface
public interface VocabularyRegistrar {
    Class<? extends Enum<? extends VocabularyTerm>> vocabulary();
}
```

Implemented by `@ApplicationScoped` CDI beans. `CdiVocabularyRegistry` injects
`Instance<VocabularyRegistrar>` and calls `register(r.vocabulary())` for each. Replaces the
old `@Produces Vocabulary` pattern. The vocabulary URI comes from `@VocabularyMetadata` on the
returned class.

---

## Updated `VocabularyRegistry` SPI

Two parallel APIs on the same interface.

**String-based** — for runtime values from `AgentDescriptor` (vocabulary URI and term value are
strings in the descriptor; enum class is not statically known at the call site).

**Typed** — for compile-time-known vocabulary classes. Returns typed enum constants;
`targetVocab.cast()` makes the cast safe and immediate rather than deferred to first use.

```java
public interface VocabularyRegistry {

    // --- Registration ---
    <T extends Enum<T> & VocabularyTerm> void register(Class<T> vocab);
    boolean isRegistered(String vocabUri);

    // --- String-based resolution (runtime values / unknown vocab class) ---
    Optional<? extends VocabularyTerm> resolve(String vocabUri, String value);
    Set<? extends VocabularyTerm>      allTerms(String vocabUri);
    Optional<String> equivalentValues(String fromUri, String value, String toUri);
    Optional<String> equivalentValues(String fromUri, String value, String toUri, DispositionAxis axis);

    // --- Typed resolution (compile-time-known vocab class) ---
    <T extends Enum<T> & VocabularyTerm>
        Optional<T> resolve(Class<T> vocab, String value);

    <S extends Enum<S> & VocabularyTerm, T extends Enum<T> & VocabularyTerm>
        Optional<T> equivalentValues(S from, Class<T> targetVocab);

    <S extends Enum<S> & VocabularyTerm, T extends Enum<T> & VocabularyTerm>
        Optional<T> equivalentValues(S from, Class<T> targetVocab, DispositionAxis axis);
}
```

**`allTerms(String vocabUri)`** restores the ability to enumerate a vocabulary's terms by URI
without knowing the enum class at compile time (the caller who has a URI-only reference has no
other way to get this information).

**`Optional<String>` for string-based equivalentValues** — Alias uniqueness is enforced at
registration (see below). With that constraint, any given string resolves to at most one source
constant, and each source constant maps to at most one constant per target vocabulary. `Set<String>`
would always be size 0 or 1 — semantically dishonest.

The old `register(Vocabulary vocabulary)` and `find(String uri)` methods are removed.

---

## `CdiVocabularyRegistry` Implementation

**Internal state:**

```
Map<String, Class<? extends Enum<?>>>  byUri    // vocabUri → enum class
Map<Class<?>, Map<String, VocabularyTerm>> byClass  // class → (value + aliases) → constant
```

**`register(Class<T>)`:**
1. Reads `@VocabularyMetadata` annotation; throws `IllegalArgumentException` if absent.
2. Calls `vocab.getEnumConstants()`. Throws if array is empty (zero-constant enums are forbidden).
3. Builds value+alias lookup map. Throws on:
   - Duplicate primary value across constants in this vocab.
   - Any alias that duplicates another alias or any primary value in this vocab.
4. Throws if `byUri` already contains the URI from a different class (duplicate URI conflict).
5. Stores both maps.

**Typed equivalentValues** — delegate directly to source constant's interface methods:
```java
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
```

`targetVocab.cast()` provides an immediate, informative `ClassCastException` if a vocabulary
implementation returns the wrong type — a programming error caught at registration-test time
rather than deferred to an unrelated call site.

**String-based equivalentValues** — resolve source term via internal maps, then delegate to
typed path:
```java
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
public Optional<String> equivalentValues(String fromUri, String value, String toUri, DispositionAxis axis) {
    var sourceClass = byUri.get(fromUri);
    if (sourceClass == null) return Optional.empty();
    var term = byClass.get(sourceClass).get(value);
    if (term == null) return Optional.empty();
    var targetClass = byUri.get(toUri);
    if (targetClass == null) return Optional.empty();
    return term.axisExactMatch(targetClass, axis).map(VocabularyTerm::value);
}
```

---

## `AgentDisposition.get(DispositionAxis)` — new method

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

This exhaustive switch is where the "adding `CONFLICT_MODE` causes compile errors everywhere
it matters" actually lives. Without this method, callers must manually know which field maps to
which axis — the correspondence exists only in convention. With it, adding
`DispositionAxis.CONFLICT_MODE` fails to compile here, forcing the caller (casehub-engine) to
handle it explicitly alongside adding the field to `AgentDisposition`.

---

## Vocabulary Enums (casehub-eidos-vocab)

### `ConscientiousnessTerm`

12 terms across 4 disposition axes. The axis grouping is informational structure only — the
type system does not enforce that DISC maps `BOLD` to the correct axis. That constraint is the
responsibility of tests in the DISC module (eidos#26), not of the Conscientiousness vocabulary
itself.

```java
@VocabularyMetadata(uri = "urn:casehub:vocab:conscientiousness",
                    name = "Conscientiousness Disposition Axes", version = "1.0")
public enum ConscientiousnessTerm implements VocabularyTerm {
    // RULE_FOLLOWING axis
    STRICT      ("strict",       "Strict Rule Following",
                 "Follows rules rigidly",           List.of("rule-bound", "compliant")),
    PRINCIPLED  ("principled",   "Principled",
                 "Follows intent of rules",          List.of("values-based")),
    FLEXIBLE    ("flexible",     "Flexible",
                 "Adapts rules to context",          List.of("adaptive", "pragmatic")),
    // RISK_APPETITE axis
    CONSERVATIVE("conservative", "Conservative Risk",
                 "Avoids uncertainty",               List.of("risk-averse", "cautious")),
    MEASURED    ("measured",     "Measured Risk",
                 "Balances risk and reward",         List.of("balanced")),
    BOLD        ("bold",         "Bold Risk",
                 "Accepts uncertainty for reward",   List.of("risk-tolerant", "adventurous")),
    // SOCIAL_ORIENTATION axis
    COLLABORATIVE("collaborative","Collaborative",
                 "Works with others by default",     List.of("team-oriented", "cooperative")),
    INDEPENDENT ("independent",  "Independent",
                 "Works alone by preference",        List.of("autonomous-social", "self-directed")),
    FACILITATIVE("facilitative", "Facilitative",
                 "Enables others to work",           List.of("supportive", "enabling")),
    // AUTONOMY axis
    DIRECTED    ("directed",     "Directed Autonomy",
                 "Follows explicit instructions",    List.of("instruction-following")),
    SEMI_AUTONOMOUS("semi-autonomous","Semi-Autonomous",
                 "Acts within defined boundaries",   List.of("bounded-autonomy")),
    AUTONOMOUS  ("autonomous",   "Autonomous",
                 "Acts on own judgment",             List.of("self-governing", "agentic"));

    private final String value, label, description;
    private final List<String> aliases;

    ConscientiousnessTerm(String value, String label, String description, List<String> aliases) {
        this.value = value; this.label = label;
        this.description = description; this.aliases = aliases;
    }

    @Override public String value()       { return value; }
    @Override public String label()       { return label; }
    @Override public String description() { return description; }
    @Override public List<String> aliases() { return aliases; }
}
```

### `SvoTerm`

```java
@VocabularyMetadata(uri = "urn:casehub:vocab:svo", name = "Subject-Verb-Object Roles", version = "1.0")
public enum SvoTerm implements VocabularyTerm {
    COORDINATOR("coordinator", "Coordinator", List.of()) {
        @Override public Optional<VocabularyTerm> exactMatch(Class<?> t) {
            return t == CasehubSlotTerm.class ? Optional.of(CasehubSlotTerm.PLANNER) : Optional.empty();
        }
    },
    PERFORMER("performer", "Performer", List.of()) {
        @Override public Optional<VocabularyTerm> exactMatch(Class<?> t) {
            return t == CasehubSlotTerm.class ? Optional.of(CasehubSlotTerm.EXECUTOR) : Optional.empty();
        }
    },
    EVALUATOR("evaluator", "Evaluator", List.of("reviewer")) {
        @Override public Optional<VocabularyTerm> exactMatch(Class<?> t) {
            return t == CasehubSlotTerm.class ? Optional.of(CasehubSlotTerm.REVIEWER) : Optional.empty();
        }
    };

    private final String value, label;
    private final List<String> aliases;
    SvoTerm(String value, String label, List<String> aliases) {
        this.value = value; this.label = label; this.aliases = aliases;
    }
    @Override public String value()         { return value; }
    @Override public String label()         { return label; }
    @Override public List<String> aliases() { return aliases; }
}
```

### `CasehubSlotTerm`

```java
@VocabularyMetadata(uri = "urn:casehub:vocab:casehubslot", name = "CaseHub Slot Roles", version = "1.0")
public enum CasehubSlotTerm implements VocabularyTerm {
    PLANNER("planner", "Planner", List.of()) {
        @Override public Optional<VocabularyTerm> exactMatch(Class<?> t) {
            return t == SvoTerm.class ? Optional.of(SvoTerm.COORDINATOR) : Optional.empty();
        }
    },
    EXECUTOR("executor", "Executor", List.of()) {
        @Override public Optional<VocabularyTerm> exactMatch(Class<?> t) {
            return t == SvoTerm.class ? Optional.of(SvoTerm.PERFORMER) : Optional.empty();
        }
    },
    REVIEWER("reviewer", "Reviewer", List.of()) {
        @Override public Optional<VocabularyTerm> exactMatch(Class<?> t) {
            return t == SvoTerm.class ? Optional.of(SvoTerm.EVALUATOR) : Optional.empty();
        }
    },
    SUPERVISOR("supervisor", "Supervisor", List.of());

    private final String value, label;
    private final List<String> aliases;
    CasehubSlotTerm(String value, String label, List<String> aliases) {
        this.value = value; this.label = label; this.aliases = aliases;
    }
    @Override public String value()         { return value; }
    @Override public String label()         { return label; }
    @Override public List<String> aliases() { return aliases; }
}
```

### Partial `DiscTerm` example (eidos#26 will complete this)

Shown to validate that the API closes the original gap:

```java
@VocabularyMetadata(uri = "urn:casehub:vocab:disc", name = "DISC Behavioural Types", version = "1.0")
public enum DiscTerm implements VocabularyTerm {
    DOMINANCE("dominance", "Dominance", List.of("D")) {
        @Override
        public Optional<VocabularyTerm> axisExactMatch(Class<?> targetVocab, DispositionAxis axis) {
            if (targetVocab != ConscientiousnessTerm.class) return Optional.empty();
            // Exhaustive switch — compile error when new DispositionAxis value is added
            return Optional.of(switch (axis) {
                case SOCIAL_ORIENTATION -> ConscientiousnessTerm.INDEPENDENT;
                case RULE_FOLLOWING     -> ConscientiousnessTerm.FLEXIBLE;
                case RISK_APPETITE      -> ConscientiousnessTerm.BOLD;
                case AUTONOMY           -> ConscientiousnessTerm.AUTONOMOUS;
            });
        }
    },
    INFLUENCE    ("influence",     "Influence",     List.of("I")),
    STEADINESS   ("steadiness",    "Steadiness",    List.of("S")),
    CONSCIENTIOUSNESS_TYPE("conscientiousness-type","Conscientiousness Type",List.of("C"));

    // standard constructor + value()/label()/aliases() implementations
}
```

The exhaustive switch with no default branch is the actual compile-time completeness gate.
When `DispositionAxis.CONFLICT_MODE` is added, this method fails to compile until the DISC
implementor adds a mapping for it.

### Companion registrars

One thin `@ApplicationScoped` bean per vocabulary enum. As future improvement, `EidosProcessor`
could scan for `@VocabularyMetadata`-annotated enum classes at Quarkus build time and
auto-generate registrars, eliminating the companion pattern entirely. Not implemented here.

```java
@ApplicationScoped
public class ConscientiousnessVocabRegistrar implements VocabularyRegistrar {
    @Override public Class<ConscientiousnessTerm> vocabulary() { return ConscientiousnessTerm.class; }
}
```

### Consumer registration (devtown example)

A consumer app defines its own vocabulary enum and registrar — no changes to eidos required:

```java
// In devtown's own module
@VocabularyMetadata(uri = "urn:devtown:vocab:role", name = "Devtown Roles", version = "1.0")
public enum DevtownRoleTerm implements VocabularyTerm {
    PLANNER("planner", "Planner", List.of()) {
        @Override public Optional<VocabularyTerm> exactMatch(Class<?> t) {
            return t == CasehubSlotTerm.class ? Optional.of(CasehubSlotTerm.PLANNER) : Optional.empty();
        }
    },
    REVIEWER("reviewer", "Reviewer", List.of());
    // ...
}

@ApplicationScoped
public class DevtownRoleRegistrar implements VocabularyRegistrar {
    @Override public Class<DevtownRoleTerm> vocabulary() { return DevtownRoleTerm.class; }
}
```

devtown depends on `casehub-eidos-vocab` to reference `CasehubSlotTerm`. The `DevtownRoleTerm`
registrar is picked up by `CdiVocabularyRegistry` automatically.

---

## Known Constraint: One-Directional Mapping

Cross-vocabulary references in `exactMatch()` use the target enum class as the key. This means
the source vocabulary must have a compile-time dependency on the target vocabulary. Outbound
mappings work for any vocabulary that can reference its target:

- `DevtownRoleTerm.PLANNER.exactMatch(CasehubSlotTerm.class)` → works (devtown depends on eidos-vocab)
- `CasehubSlotTerm.PLANNER.exactMatch(DevtownRoleTerm.class)` → cannot work (eidos-vocab cannot
  depend on devtown)

Consequence: `equivalentValues(CasehubSlotTerm.PLANNER, DevtownRoleTerm.class)` always returns
empty, even if devtown declared the reverse mapping. The string-based path has the same
limitation — it resolves via the source term's `exactMatch()`, which has the same class
dependency constraint.

For all current use cases (SVO ↔ CasehubSlot in the same module, DISC → Conscientiousness in
the same module), this constraint does not apply.

**Path forward:** A `VocabularyMapping<S, T>` SPI registered alongside vocabularies would
decouple mappings from the source term entirely:
```java
public interface VocabularyMapping<S extends Enum<S> & VocabularyTerm,
                                    T extends Enum<T> & VocabularyTerm> {
    Class<S> source();
    Class<T> target();
    Map<S, T> mappings();
}
```

The registry would maintain a reverse index, support third-party bridge modules, and make
bidirectionality explicit and verifiable. Not implemented here.

---

## Registration Semantics

| Condition | Behaviour |
|---|---|
| Zero-constant enum | throws `IllegalArgumentException` at registration |
| `@VocabularyMetadata` annotation absent | throws `IllegalArgumentException` at registration |
| Duplicate primary value within vocab | throws `IllegalArgumentException` at registration |
| Alias duplicates another alias within vocab | throws `IllegalArgumentException` at registration |
| Alias matches a primary value within vocab | throws `IllegalArgumentException` at registration |
| Same URI from two different enum classes | throws `IllegalArgumentException` at registration |
| Same URI + same class (re-registration) | overwrites silently (idempotent) |

---

## Scope of Changes

| File | Action |
|------|--------|
| `api/Vocabulary.java` | **Delete** |
| `api/VocabularyTerm.java` | **Delete record; create interface of same name** |
| `api/VocabularyMetadata.java` | **Create** (annotation) |
| `api/DispositionAxis.java` | **Create** |
| `api/spi/VocabularyRegistrar.java` | **Create** |
| `api/VocabularyRegistry.java` | **Breaking rewrite** |
| `api/AgentDisposition.java` | **Add `get(DispositionAxis)` method** |
| `runtime/vocabulary/CdiVocabularyRegistry.java` | **Full rewrite** |
| `runtime/vocabulary/CdiVocabularyRegistryTest.java` | **Full rewrite** |
| `vocab/ConscientiousnessVocabularyProducer.java` | **Delete; create `ConscientiousnessTerm` + `ConscientiousnessVocabRegistrar`** |
| `vocab/SvoVocabularyProducer.java` | **Delete; create `SvoTerm` + `SvoVocabRegistrar`** |
| `vocab/CasehubSlotVocabularyProducer.java` | **Delete; create `CasehubSlotTerm` + `CasehubSlotVocabRegistrar`** |
| `vocab/*VocabularyTest.java` (3 files) | **Update** |
| `examples/CrossVocabularyDiscoveryTest.java` | **Update** |
| `examples/DispositionVocabularyTest.java` | **Update** |

No cross-repo changes. Nothing outside `casehub-eidos` uses `Vocabulary`, `VocabularyTerm`,
or `VocabularyRegistry`.

---

## Out of Scope

- `DiscTerm` and `BelbinTerm` enums — eidos#26. This spec provides the foundation.
- `conflictMode` as 5th `AgentDisposition` axis — eidos#38. When it lands: add
  `CONFLICT_MODE` to `DispositionAxis`; compiler identifies every exhaustive switch and
  `AgentDisposition.get()` that needs updating.
- `VocabularyMapping` SPI for open-world bidirectional mappings — acknowledged as path
  forward; not needed for any current use case.
- Build-time `@VocabularyMetadata` scanning via `EidosProcessor` — acknowledged as future
  alternative to companion registrar beans.
- JPA vocabulary persistence — all vocabularies are CDI-discovered in-memory.
