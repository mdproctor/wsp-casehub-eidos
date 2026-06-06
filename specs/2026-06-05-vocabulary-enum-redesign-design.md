# Vocabulary System Enum Redesign + DispositionAxis
**Issue:** eidos#40
**Date:** 2026-06-05
**Status:** approved (rev 4)

---

## Problem

`VocabularyRegistry.equivalentValues(String fromVocab, String value, String toVocab)` cannot
support the DISC-as-disposition-vocabulary architecture (ADR 0003) because a single DISC type
maps to *different* Conscientiousness values depending on which `AgentDisposition` axis is being
resolved. There is no axis parameter, so there is no way to distinguish.

The existing `Vocabulary` and `VocabularyTerm` records are stringly-typed throughout: vocabulary
URIs, term values, and cross-vocabulary references are all plain strings. Vocabulary terms are
fixed at compile time — even consumer-defined vocabularies are defined in code, not discovered
from external sources at runtime. Strings give up the compile-time safety that enums provide.

This issue redesigns the vocabulary system from String-based records to an enum-based interface
with typed cross-vocabulary methods, adding `DispositionAxis` as a typed axis key, and extends
`VocabularyRegistry` with typed generic methods alongside the string-based methods required for
runtime descriptor lookups.

---

## End-to-End Trace

The original problem, traced through the new API:

1. Agent registered with `dispositionVocabulary = "urn:casehub:vocab:disc"` and
   `AgentDisposition.riskAppetite = "dominance"` (D type declared on the risk axis).
2. casehub-engine receives a query: "find agents whose riskAppetite is equivalent to
   `ConscientiousnessTerm.BOLD`."
3. Engine calls the string-based API (descriptor values are strings at runtime):
   ```java
   registry.equivalentValues(
       desc.dispositionVocabulary(),
       desc.disposition().get(DispositionAxis.RISK_APPETITE).orElseThrow(),  // "dominance"
       ConscientiousnessVocab.URI,
       DispositionAxis.RISK_APPETITE
   )
   // → Optional.of("bold")
   ```
4. Engine finds agents where `riskAppetite = "dominance"` (DISC) OR `riskAppetite = "bold"`
   (Conscientiousness) — both express the same profile on that axis.

`AgentDisposition.get(DispositionAxis)` is the only place the engine needs to know which field
maps to which axis. When `CONFLICT_MODE` is added to `DispositionAxis`, that switch fails to
compile, forcing the engine caller to handle it.

---

## Deleted Types

- `Vocabulary` record — removed entirely; the enum class IS the vocabulary
- `VocabularyTerm` record — removed; replaced by the `VocabularyTerm` interface of the same name

---

## New Types in `casehub-eidos-api`

### `@VocabularyMetadata` annotation

Vocabulary-level metadata belongs on the class, not on instances. A per-constant `vocabUri()`
would require every constant to return the same value — a constraint validated at startup rather
than being structurally impossible to violate. An annotation on the enum class eliminates the
repetition and the validation entirely.

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

**Convention for `name()` and `version()`:** empty string (`""`) means "not provided" — the
vocabulary author chose not to supply the metadata. Callers should treat `name().isEmpty()` as
absent. This follows the standard Java annotation convention of using empty string as the absent
sentinel (annotations cannot express `null`).

### `VocabularyTerm` interface (replaces the deleted record of the same name)

Pure term-level data. No vocabulary-level metadata — that lives on the annotation.

```java
package io.casehub.eidos.api;

/**
 * A term within a vocabulary. Implemented by enum constants.
 *
 * <p>{@link #exactMatch} and {@link #axisExactMatch} are independent. A term may implement
 * either, both, or neither. The registry routes axis-aware and axis-unaware lookups to the
 * appropriate method independently — calling the axis-unaware overload against a DISC term
 * returns {@code Optional.empty()} (correct, not an error).
 */
public interface VocabularyTerm {
    String value();
    String label();
    default String description()   { return ""; }
    default List<String> aliases() { return List.of(); }

    /**
     * Axis-unaware cross-vocabulary equivalence.
     * Returns the equivalent constant in {@code targetVocab}, or empty if none.
     * Independent of {@link #axisExactMatch} — implement one, both, or neither.
     */
    default Optional<VocabularyTerm> exactMatch(Class<?> targetVocab) {
        return Optional.empty();
    }

    /**
     * Axis-aware cross-vocabulary equivalence.
     *
     * <p>Implementations that cover a given {@code targetVocab} MUST use an exhaustive switch
     * on {@code axis} with no default branch. This ensures that adding a new
     * {@link DispositionAxis} value causes a compile error in every implementation that covers
     * that target vocabulary, forcing explicit handling of the new axis.
     *
     * <p>{@code Optional.empty()} is a valid switch branch — use it for axes where no
     * meaningful mapping exists. Do NOT wrap the switch in {@code Optional.of()} as that
     * forbids gaps.
     *
     * <pre>{@code
     * return switch (axis) {
     *     case RISK_APPETITE -> Optional.of(ConscientiousnessTerm.BOLD);
     *     case AUTONOMY      -> Optional.empty();   // no clear mapping for this axis
     *     // remaining cases...
     * };
     * }</pre>
     *
     * Independent of {@link #exactMatch} — implement one, both, or neither.
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
switch on `DispositionAxis` — in vocabulary implementations and in `AgentDisposition.get()` —
fails to compile until it covers the new value.

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

```java
public interface VocabularyRegistry {

    // --- Registration ---
    <T extends Enum<T> & VocabularyTerm> void register(Class<T> vocab);
    boolean isRegistered(String vocabUri);

    // --- String-based resolution (runtime values / unknown vocab class) ---
    Optional<? extends VocabularyTerm> resolve(String vocabUri, String value);
    List<? extends VocabularyTerm>     allTerms(String vocabUri);
    Optional<String> equivalentValues(String fromUri, String value, String toUri);
    Optional<String> equivalentValues(String fromUri, String value, String toUri, DispositionAxis axis);

    // --- Typed resolution (compile-time-known vocab class) ---
    // NOTE: typed equivalentValues methods do NOT require the source or target vocabulary to
    // be registered. They delegate directly to the source constant's exactMatch() /
    // axisExactMatch() methods, which are compile-time constants. isRegistered() and the
    // typed equivalentValues methods are independent — registration state does not affect
    // typed lookups. A caller may use equivalentValues(SvoTerm.EVALUATOR, CasehubSlotTerm.class)
    // before either vocabulary is registered and receive the correct answer.
    <T extends Enum<T> & VocabularyTerm>
        Optional<T> resolve(Class<T> vocab, String value);

    <S extends Enum<S> & VocabularyTerm, T extends Enum<T> & VocabularyTerm>
        Optional<T> equivalentValues(S from, Class<T> targetVocab);

    <S extends Enum<S> & VocabularyTerm, T extends Enum<T> & VocabularyTerm>
        Optional<T> equivalentValues(S from, Class<T> targetVocab, DispositionAxis axis);
}
```

**`allTerms(String vocabUri)`** returns constants in enum declaration order (see internal state
below). Return type is `List` (not `Set`) to preserve the vocabulary author's ordering, which
is meaningful for rendering purposes (e.g. `STRICT, PRINCIPLED, FLEXIBLE` expresses a
progression).

**`Optional<String>` for string-based equivalentValues** — Alias uniqueness is enforced at
registration (see Registration Semantics). With uniqueness enforced, any string resolves to at
most one source constant, which maps to at most one constant per target vocabulary.

The old `register(Vocabulary vocabulary)` and `find(String uri)` methods are removed.

---

## `CdiVocabularyRegistry` Implementation

**Internal state — three maps:**

```
Map<String, Class<? extends Enum<?>>>         byUri          // vocabUri → enum class
Map<Class<?>, Map<String, VocabularyTerm>>    byClass        // class → (value + aliases) → constant (lookup index)
Map<Class<?>, List<? extends VocabularyTerm>> byClassOrdered // class → constants in declaration order (for allTerms)
```

All three are `ConcurrentHashMap` for safe concurrent reads after initialization.

**Thread safety:** `register()` is not thread-safe for concurrent calls. It modifies three maps
in sequence without atomicity guarantees — two simultaneous `register()` calls could pass the
duplicate-URI check before either stores the URI. `register()` is designed for single-threaded
initialization via `@PostConstruct`. Concurrent reads after `@PostConstruct` completes are safe
(ConcurrentHashMap). Concurrent registration support is a future concern if needed.

**`@PostConstruct init()`:**
```java
@Inject @Any Instance<VocabularyRegistrar> registrars;

@PostConstruct
void init() {
    for (VocabularyRegistrar r : registrars) {
        register(r.vocabulary());  // validated path — not an internal shortcut
    }
}
```

**`register(Class<T>)`:**
1. Reads `@VocabularyMetadata`; throws `IllegalArgumentException` if absent.
2. Calls `vocab.getEnumConstants()`. Throws `IllegalArgumentException` if empty.
3. Populates `byClassOrdered` with `List.copyOf(Arrays.asList(constants))` — immutable list in
   declaration order (`getEnumConstants()` contract). Stored immutably so `allTerms()` returns it
   directly without defensive copying.
4. Builds `byClass` lookup map. Throws `IllegalArgumentException` on:
   - Duplicate primary value across constants in this vocab.
   - Any alias duplicating another alias or any primary value within this vocab.
5. Throws `IllegalArgumentException` if `byUri` already maps the URI to a *different* class
   (duplicate URI conflict); silently overwrites if same class (idempotent re-registration).
6. Stores in all three maps.

**`allTerms()` implementation:**
```java
@Override
public List<? extends VocabularyTerm> allTerms(String vocabUri) {
    var clazz = byUri.get(vocabUri);
    return clazz == null ? List.of() : byClassOrdered.get(clazz);
}
```

Returns the pre-built immutable list directly. No streaming, no deduplication needed — distinct
by construction (one entry per enum constant at registration time).

**Typed equivalentValues** — delegate to source constant's interface methods; do not touch
internal maps (typed lookups bypass registration state by design, see SPI note above):
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
implementation returns the wrong type — a programming error caught at test time.

**String-based equivalentValues:**
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

This exhaustive switch is where compile-time safety is anchored. Adding
`DispositionAxis.CONFLICT_MODE` fails to compile here, forcing the caller to handle it alongside
adding the corresponding field to `AgentDisposition`.

---

## Vocabulary Enums (casehub-eidos-vocab)

### `ConscientiousnessTerm`

12 terms across 4 disposition axes. `description()` is used by `ClaudeMarkdownRenderer` during
system prompt rendering and must be preserved. The renderer treats `description().isEmpty()` as
absent and omits the description field from the rendered prompt. The axis groupings are
informational (comments) only — the type system does not enforce that DISC maps a term to the
correct axis. Correctness of axis assignment is the responsibility of tests in the DISC module
(eidos#26).

```java
@VocabularyMetadata(uri = "urn:casehub:vocab:conscientiousness",
                    name = "Conscientiousness Disposition Axes", version = "1.0")
public enum ConscientiousnessTerm implements VocabularyTerm {
    // RULE_FOLLOWING axis
    STRICT      ("strict",        "Strict Rule Following",
                 "Follows rules rigidly",              List.of("rule-bound", "compliant")),
    PRINCIPLED  ("principled",    "Principled",
                 "Follows intent of rules",             List.of("values-based")),
    FLEXIBLE    ("flexible",      "Flexible",
                 "Adapts rules to context",             List.of("adaptive", "pragmatic")),
    // RISK_APPETITE axis
    CONSERVATIVE("conservative",  "Conservative Risk",
                 "Avoids uncertainty",                  List.of("risk-averse", "cautious")),
    MEASURED    ("measured",      "Measured Risk",
                 "Balances risk and reward",            List.of("balanced")),
    BOLD        ("bold",          "Bold Risk",
                 "Accepts uncertainty for reward",      List.of("risk-tolerant", "adventurous")),
    // SOCIAL_ORIENTATION axis
    COLLABORATIVE("collaborative","Collaborative",
                 "Works with others by default",        List.of("team-oriented", "cooperative")),
    INDEPENDENT ("independent",   "Independent",
                 "Works alone by preference",           List.of("autonomous-social", "self-directed")),
    FACILITATIVE("facilitative",  "Facilitative",
                 "Enables others to work",              List.of("supportive", "enabling")),
    // AUTONOMY axis
    DIRECTED    ("directed",      "Directed Autonomy",
                 "Follows explicit instructions",       List.of("instruction-following")),
    SEMI_AUTONOMOUS("semi-autonomous","Semi-Autonomous",
                 "Acts within defined boundaries",      List.of("bounded-autonomy")),
    AUTONOMOUS  ("autonomous",    "Autonomous",
                 "Acts on own judgment",                List.of("self-governing", "agentic"));

    private final String value, label, description;
    private final List<String> aliases;

    ConscientiousnessTerm(String v, String l, String d, List<String> a) {
        value = v; label = l; description = d; aliases = a;
    }

    @Override public String value()         { return value; }
    @Override public String label()         { return label; }
    @Override public String description()   { return description; }
    @Override public List<String> aliases() { return aliases; }
}
```

### `SvoTerm`

Descriptions preserved from the original `SvoVocabularyProducer` — used by renderer.

```java
@VocabularyMetadata(uri = "urn:casehub:vocab:svo",
                    name = "Subject-Verb-Object Roles", version = "1.0")
public enum SvoTerm implements VocabularyTerm {
    COORDINATOR("coordinator", "Coordinator", "Orchestrates other agents",       List.of()) {
        @Override public Optional<VocabularyTerm> exactMatch(Class<?> t) {
            // Class identity is correct — Class instances are singletons per class loader
            return t == CasehubSlotTerm.class ? Optional.of(CasehubSlotTerm.PLANNER) : Optional.empty();
        }
    },
    PERFORMER  ("performer",   "Performer",   "Executes the assigned work",      List.of()) {
        @Override public Optional<VocabularyTerm> exactMatch(Class<?> t) {
            return t == CasehubSlotTerm.class ? Optional.of(CasehubSlotTerm.EXECUTOR) : Optional.empty();
        }
    },
    EVALUATOR  ("evaluator",   "Evaluator",   "Assesses quality of work",        List.of("reviewer")) {
        @Override public Optional<VocabularyTerm> exactMatch(Class<?> t) {
            return t == CasehubSlotTerm.class ? Optional.of(CasehubSlotTerm.REVIEWER) : Optional.empty();
        }
    };

    private final String value, label, description;
    private final List<String> aliases;

    SvoTerm(String v, String l, String d, List<String> a) {
        value = v; label = l; description = d; aliases = a;
    }

    @Override public String value()         { return value; }
    @Override public String label()         { return label; }
    @Override public String description()   { return description; }
    @Override public List<String> aliases() { return aliases; }
}
```

### `CasehubSlotTerm`

```java
@VocabularyMetadata(uri = "urn:casehub:vocab:casehubslot",
                    name = "CaseHub Slot Roles", version = "1.0")
public enum CasehubSlotTerm implements VocabularyTerm {
    PLANNER   ("planner",    "Planner",    "Plans and coordinates tasks",         List.of()) {
        @Override public Optional<VocabularyTerm> exactMatch(Class<?> t) {
            // Class identity is correct — Class instances are singletons per class loader
            return t == SvoTerm.class ? Optional.of(SvoTerm.COORDINATOR) : Optional.empty();
        }
    },
    EXECUTOR  ("executor",   "Executor",   "Executes assigned tasks",             List.of()) {
        @Override public Optional<VocabularyTerm> exactMatch(Class<?> t) {
            return t == SvoTerm.class ? Optional.of(SvoTerm.PERFORMER) : Optional.empty();
        }
    },
    REVIEWER  ("reviewer",   "Reviewer",   "Reviews and validates outputs",       List.of()) {
        @Override public Optional<VocabularyTerm> exactMatch(Class<?> t) {
            return t == SvoTerm.class ? Optional.of(SvoTerm.EVALUATOR) : Optional.empty();
        }
    },
    SUPERVISOR("supervisor", "Supervisor", "Oversees team execution",             List.of());

    private final String value, label, description;
    private final List<String> aliases;

    CasehubSlotTerm(String v, String l, String d, List<String> a) {
        value = v; label = l; description = d; aliases = a;
    }

    @Override public String value()         { return value; }
    @Override public String label()         { return label; }
    @Override public String description()   { return description; }
    @Override public List<String> aliases() { return aliases; }
}
```

### Partial `DiscTerm` example — illustrative only

**This enum is NOT created as part of this issue.** eidos#26 owns all DISC and Belbin
implementations. This example exists solely to validate that the API closes the original gap
and to demonstrate the correct `axisExactMatch` pattern — exhaustive switch with
`Optional.empty()` as valid branches.

```java
@VocabularyMetadata(uri = "urn:casehub:vocab:disc",
                    name = "DISC Behavioural Types", version = "1.0")
public enum DiscTerm implements VocabularyTerm {
    DOMINANCE("dominance", "Dominance", "High-D: direct, results-oriented", List.of("D")) {
        @Override
        public Optional<VocabularyTerm> axisExactMatch(Class<?> targetVocab, DispositionAxis axis) {
            if (targetVocab != ConscientiousnessTerm.class) return Optional.empty();
            // Exhaustive switch enforces axis completeness at compile time — adding a new
            // DispositionAxis value causes a compile error here. Adding a new target vocabulary
            // (e.g. BelbinTerm) requires a new if-branch above; no compile-time enforcement
            // exists for the target-vocabulary dimension.
            // Optional.empty() is valid where no meaningful mapping exists.
            return switch (axis) {
                case SOCIAL_ORIENTATION -> Optional.of(ConscientiousnessTerm.INDEPENDENT);
                case RULE_FOLLOWING     -> Optional.of(ConscientiousnessTerm.FLEXIBLE);
                case RISK_APPETITE      -> Optional.of(ConscientiousnessTerm.BOLD);
                case AUTONOMY           -> Optional.of(ConscientiousnessTerm.AUTONOMOUS);
            };
        }
    },
    INFLUENCE("influence", "Influence", "High-I: enthusiastic, collaborative", List.of("I")) {
        @Override
        public Optional<VocabularyTerm> axisExactMatch(Class<?> targetVocab, DispositionAxis axis) {
            if (targetVocab != ConscientiousnessTerm.class) return Optional.empty();
            // Class identity is correct — Class instances are singletons per class loader
            return switch (axis) {
                case SOCIAL_ORIENTATION -> Optional.of(ConscientiousnessTerm.COLLABORATIVE);
                case RULE_FOLLOWING     -> Optional.of(ConscientiousnessTerm.PRINCIPLED);
                case RISK_APPETITE      -> Optional.of(ConscientiousnessTerm.MEASURED);
                case AUTONOMY           -> Optional.empty();  // no clear mapping — eidos#26 to decide
            };
        }
    },
    STEADINESS    ("steadiness",         "Steadiness",         "High-S: patient, reliable",       List.of("S")),
    CONSCIENTIOUSNESS_TYPE("conscientiousness-type","Conscientiousness Type","High-C: analytical, precise",List.of("C"));

    private final String value, label, description;
    private final List<String> aliases;

    DiscTerm(String v, String l, String d, List<String> a) {
        value = v; label = l; description = d; aliases = a;
    }

    @Override public String value()         { return value; }
    @Override public String label()         { return label; }
    @Override public String description()   { return description; }
    @Override public List<String> aliases() { return aliases; }
}
```

`INFLUENCE.AUTONOMY → Optional.empty()` illustrates a legitimate gap. The exhaustive switch
still compiles (all `DispositionAxis` values are covered); `Optional.empty()` is the honest
answer when the mapping does not exist. This is the correct pattern — `Optional.of(switch{...})`
would have forced an invented mapping.

### Companion registrars

```java
@ApplicationScoped
public class ConscientiousnessVocabRegistrar implements VocabularyRegistrar {
    @Override public Class<ConscientiousnessTerm> vocabulary() { return ConscientiousnessTerm.class; }
}
```

One per vocabulary enum. As a future improvement, `EidosProcessor` could scan for
`@VocabularyMetadata`-annotated enum classes at Quarkus build time and auto-generate registrars.
Not implemented here.

### Consumer registration (devtown example)

```java
// In devtown's own module — no changes to eidos required
@VocabularyMetadata(uri = "urn:devtown:vocab:role", name = "Devtown Roles")
public enum DevtownRoleTerm implements VocabularyTerm {
    PLANNER("planner", "Planner", "Owns feature planning", List.of()) {
        @Override public Optional<VocabularyTerm> exactMatch(Class<?> t) {
            return t == CasehubSlotTerm.class
                ? Optional.of(CasehubSlotTerm.PLANNER) : Optional.empty();
        }
    },
    REVIEWER("reviewer", "Reviewer", "Reviews pull requests", List.of());
    // ...
}

@ApplicationScoped
public class DevtownRoleRegistrar implements VocabularyRegistrar {
    @Override public Class<DevtownRoleTerm> vocabulary() { return DevtownRoleTerm.class; }
}
```

---

## Known Constraint: One-Directional Mapping

Cross-vocabulary references use the target enum class as key. The source vocabulary must have
a compile-time dependency on the target. Outbound mappings work for any vocabulary that can
reference its target; the reverse direction cannot be declared from a lower-layer module.

For all current use cases (SVO ↔ CasehubSlot in the same module, DISC → Conscientiousness in
the same module), this constraint does not apply.

**Path forward:** A `VocabularyMapping<S, T>` SPI registered alongside vocabularies would
decouple mappings from the source term, support third-party bridge modules, and make
bidirectionality explicit. Not implemented here.

---

## Registration Semantics

| Condition | Behaviour |
|---|---|
| Zero-constant enum | throws `IllegalArgumentException` |
| `@VocabularyMetadata` absent | throws `IllegalArgumentException` |
| Duplicate primary value within vocab | throws `IllegalArgumentException` |
| Alias duplicates another alias within vocab | throws `IllegalArgumentException` |
| Alias matches any primary value within vocab | throws `IllegalArgumentException` |
| Same URI from two different enum classes | throws `IllegalArgumentException` |
| Same URI + same class (re-registration) | silently overwrites (idempotent) |

---

## Test Coverage

### `CdiVocabularyRegistryTest` (`@QuarkusTest`, runtime module)

**Registration:**
- Programmatic `register()` of a test enum — `isRegistered()` returns true
- `isRegistered()` returns false for unknown URI
- Zero-constant enum → throws at registration
- Missing `@VocabularyMetadata` → throws at registration
- Duplicate primary value → throws at registration
- Alias duplicates another alias → throws at registration
- Alias matches a primary value → throws at registration
- Duplicate URI from different class → throws at registration
- Re-registration with same class → idempotent, no throw

**`allTerms()`:**
- Returns all distinct constants in declaration order (not duplicated by aliases)
- Returns empty list for unknown URI

**`resolve()`:**
- Typed: `resolve(Class<T>, String value)` — by primary value, by alias, missing → empty
- String-based: `resolve(String uri, String value)` — by primary value, by alias

**`equivalentValues()` — axis-unaware:**
- Typed: `equivalentValues(S from, Class<T>)` — returns correct constant; cast validated
- Typed: target vocab not covered by source term → `Optional.empty()`
- String-based: `equivalentValues(String, String, String)` — returns correct value
- String-based: unknown source URI → empty; unknown target URI → empty; unknown value → empty

**`equivalentValues()` — axis-aware:**
- Typed: `equivalentValues(S from, Class<T>, DispositionAxis)` — returns correct constant
- Typed: axis returns `Optional.empty()` from implementation (sparse mapping) → empty
- String-based: `equivalentValues(String, String, String, DispositionAxis)` — correct value
- Axis-unaware overload against axis-only term → empty (correct, not an error)

**Consumer pattern:**
- Test enum with `exactMatch()` to a platform enum registered programmatically; typed
  cross-vocab lookup returns the correct typed constant

### `AgentDispositionTest` (unit test, api module)

- All four `get(DispositionAxis)` cases return the correct field value
- Null field values return `Optional.empty()`

### `CrossVocabularyDiscoveryTest` (`@QuarkusTest`, examples module)

Updated: `find(URI)` replaced by `isRegistered(URI)`. Typed assertions:
```java
assertThat(vocabRegistry.equivalentValues(SvoTerm.EVALUATOR, CasehubSlotTerm.class))
    .contains(CasehubSlotTerm.REVIEWER);
```

### `DispositionVocabularyTest` (`@QuarkusTest`, examples module)

Updated: typed `resolve(ConscientiousnessTerm.class, "strict")` replaces string-based resolve.
`allTerms()` assertion: 12 distinct constants in declaration order.

### Three vocab tests

Updated to exercise enum constants directly.

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

- `DiscTerm` and `BelbinTerm` full implementations — eidos#26
- `conflictMode` as 5th `AgentDisposition` axis — eidos#38; when it lands, add
  `CONFLICT_MODE` to `DispositionAxis` and the compiler identifies all sites
- `VocabularyMapping` SPI for open-world bidirectional mappings — path forward documented
- Build-time `@VocabularyMetadata` scanning via `EidosProcessor` — future alternative to
  companion registrar beans
- JPA vocabulary persistence — all vocabularies are CDI-discovered in-memory
