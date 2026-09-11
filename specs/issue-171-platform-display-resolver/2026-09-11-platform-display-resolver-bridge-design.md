# Platform DisplayTermResolver Bridge — Design Spec

**Issue:** casehubio/eidos#171
**Date:** 2026-09-11
**Scope:** S — one class modified, two new methods, mapping utility, tests
**Upstream:** casehubio/platform#283 (closed — SPI landed)

---

## Problem

Platform now ships `io.casehub.platform.api.display.DisplayTermResolver` as the platform-wide SPI. Eidos has its own `io.casehub.eidos.api.DisplayTermResolver` with a typed `DispositionAxis` parameter. Consumers that only need display label resolution should depend on platform-api, not eidos-api. Eidos's `DefaultDisplayTermResolver` needs to implement both SPIs so it satisfies CDI injection against either type.

## Solution

`DefaultDisplayTermResolver` implements both interfaces. The platform SPI's `resolveLabel(String, String)` signature is identical to the eidos SPI's default overload — one implementation satisfies both. The platform's `mapTerm` methods are new additions that expose the cross-vocabulary translation step (value → value) without label resolution.

### Platform SPI (from platform-api)

```java
package io.casehub.platform.api.display;

public interface DisplayTermResolver {
    String resolveLabel(String value, String vocabUri);
    Optional<String> mapTerm(String value, String sourceVocabUri, String targetVocabUri);
    Optional<String> mapTerm(String value, String sourceVocabUri, String targetVocabUri,
                             String mappingContext);
}
```

### Eidos SPI (unchanged)

```java
package io.casehub.eidos.api;

public interface DisplayTermResolver {
    String resolveLabel(String value, String sourceVocabUri,
                        String targetVocabUri, DispositionAxis axis);
    default String resolveLabel(String value, String sourceVocabUri, String targetVocabUri) { ... }
    default String resolveLabel(String value, String sourceVocabUri) { ... }
}
```

### Method resolution

| Platform method | Implementation |
|---|---|
| `resolveLabel(value, vocabUri)` | Satisfied by eidos `resolveLabel(value, sourceVocabUri)` — same signature, same semantics |
| `mapTerm(value, src, tgt)` | `registry.equivalentValues(src, value, tgt)` — axis-unaware cross-vocab translation |
| `mapTerm(value, src, tgt, context)` | Parse `context` to `DispositionAxis` via `jsonKey()` match, then `registry.equivalentValues(src, value, tgt, axis)` |

### mappingContext → DispositionAxis

```java
private static DispositionAxis parseAxis(String mappingContext) {
    if (mappingContext == null) return null;
    for (DispositionAxis axis : DispositionAxis.values()) {
        if (axis.jsonKey().equals(mappingContext)) return axis;
    }
    return null;
}
```

Null or unrecognized `mappingContext` → axis-unaware mapping (falls back to `exactMatch`). Silent fallback — no exception for unknown contexts.

---

## Changes

### DefaultDisplayTermResolver

```java
@DefaultBean
@ApplicationScoped
public class DefaultDisplayTermResolver
        implements io.casehub.eidos.api.DisplayTermResolver,
                   io.casehub.platform.api.display.DisplayTermResolver {

    // existing resolveLabel methods unchanged

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

### Import handling

Both interfaces share the name `DisplayTermResolver`. The implementation class uses fully-qualified names in the `implements` clause. Internal code continues to import the eidos version.

---

## Testing

### New tests in DefaultDisplayTermResolverTest

| Test | Expected |
|---|---|
| `mapTerm_cross_vocab_returns_target_value` | `mapTerm("observer", devtown, gastown)` → `Optional.of("witness")` |
| `mapTerm_no_match_returns_empty` | `mapTerm("planner", devtown, gastown)` → `Optional.empty()` |
| `mapTerm_null_value_returns_empty` | `mapTerm(null, devtown, gastown)` → `Optional.empty()` |
| `mapTerm_with_context_axis_aware` | `mapTerm("bold", axisA, axisB, "riskAppetite")` → `Optional.of("adventurous")` |
| `mapTerm_with_unknown_context_falls_back` | `mapTerm("bold", axisA, axisB, "unknown")` → `Optional.empty()` (no axis-unaware match for axisExactMatch-only term) |
| `platform_resolveLabel_matches_eidos` | Inject as `io.casehub.platform.api.display.DisplayTermResolver`, call `resolveLabel("witness", gastown)` → `"Witness"` |

Test vocabularies from the existing test class are reused (DevtownRole, GastownRole, AxisVocabA, AxisVocabB).

---

## Module Changes

| Module | Change |
|---|---|
| `runtime/` | `DefaultDisplayTermResolver` adds `implements io.casehub.platform.api.display.DisplayTermResolver`, adds `mapTerm` methods + `parseAxis` utility |
| `runtime/pom.xml` | platform-api already a dependency — no POM change needed |
| `api/` | No changes — eidos SPI stays as-is |

---

## Not in Scope

- Removing the eidos `DisplayTermResolver` SPI — consumers use the typed `DispositionAxis` parameter
- Auto-discovery in `mapTerm` (null sourceVocabUri) — `mapTerm` requires explicit source/target
- Reactive parity — not needed for a display-time concern

---

## References

- `DefaultDisplayTermResolver.java` — current implementation from #170
- `DisplayTermResolver.java` (platform-api) — platform SPI from platform#283
- `DisplayTermResolver.java` (eidos-api) — eidos SPI from #170
- `DispositionAxis.java` — `jsonKey()` for context string mapping
- D1-D3 decision records (`decisions.md`)
