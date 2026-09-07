# DisplayTermResolver — Design Spec

**Issue:** casehubio/eidos#170
**Date:** 2026-09-07
**Scope:** S — one SPI, one implementation, convenience overloads, unit tests
**Temporary home:** eidos-api + eidos-runtime (moves to platform-api per casehubio/platform#283)

---

## Problem

Different CaseHub domains use different terminology for the same agent concepts. Devtown calls them "planner", "reviewer", "observer". Gastown calls them "witness", "polecat", "deacon". The vocabulary system already models these as separate vocabularies with `exactMatch()` and `axisExactMatch()` cross-references — but no service exists to resolve "given this internal value and a target vocabulary, what display label should I show?"

Every consumer (blocks-ui org diagram, SystemPromptRenderer, qhorus channel rendering) would need to re-implement the same resolution chain: look up value → cross-vocab translate → resolve label → fallback. This should be one service.

## Solution

A `DisplayTermResolver` SPI in eidos-api with a `DefaultDisplayTermResolver` implementation in eidos-runtime that delegates to `VocabularyRegistry`.

### API

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

**Parameters:**
- `value` — the internal term value (e.g. `"witness"`, `"collaborative"`)
- `sourceVocabUri` — vocabulary the value belongs to (nullable — searches all registered vocabularies if null; explicit is preferred, auto-discovery is best-effort)
- `targetVocabUri` — vocabulary to display in (nullable — null means use source vocab's own label)
- `axis` — disposition axis for axis-aware cross-vocab (nullable — null means axis-unaware `exactMatch`)

### Resolution Chain

```
1. FIND SOURCE TERM
   if sourceVocabUri != null:
     sourceTerm = registry.resolve(sourceVocabUri, value)
   else:
     for each uri in registry.registeredUris():
       sourceTerm = registry.resolve(uri, value)
       if found → break (best-effort, first match)

   if sourceTerm not found → return value (raw passthrough)

2. RESOLVE DISPLAY LABEL
   if targetVocabUri == null OR targetVocabUri == sourceVocabUri:
     return sourceTerm.label()

   if axis != null:
     targetValue = registry.equivalentValues(sourceUri, value, targetVocabUri, axis)
   else:
     targetValue = registry.equivalentValues(sourceUri, value, targetVocabUri)

   if targetValue present:
     targetTerm = registry.resolve(targetVocabUri, targetValue)
     if targetTerm present → return targetTerm.label()
     else → return targetValue (resolved but no term metadata)

   return sourceTerm.label()  (no cross-vocab match → fallback to source label)
```

The two-step cross-vocab resolution (`equivalentValues` → `resolve`) is necessary because `equivalentValues` returns the target *value* (String), not the term. The second `resolve` call retrieves the term for its `label()`.

### Consumer Usage

**Org diagram — slot labels:**
```java
// Agent descriptor carries vocabUriForSlot()
String label = resolver.resolveLabel(
    descriptor.slot(),
    descriptor.vocabUriForSlot().orElse(null),
    userSelectedDisplayVocab);
```

**Org diagram — disposition axes:**
```java
// Disposition value on a specific axis
String label = resolver.resolveLabel(
    disposition.get(DispositionAxis.AUTONOMY).getFirst().term(),
    descriptor.vocabUriForAxis(DispositionAxis.AUTONOMY).orElse(null),
    userSelectedDisplayVocab,
    DispositionAxis.AUTONOMY);
```

**Terminology swap — "show gastown in devtown terms":**
```java
// The UI sets a display vocabulary preference
String displayVocab = "urn:devtown:vocab:roles";

// Every resolve call passes it as targetVocabUri
resolver.resolveLabel("witness", "urn:gastown:vocab:org", displayVocab)
// → finds "witness" in gastown → cross-vocab to devtown → "observer" label
```

---

## Implementation — eidos-runtime

### DefaultDisplayTermResolver

```java
package io.casehub.eidos.runtime.display;

@DefaultBean
@ApplicationScoped
public class DefaultDisplayTermResolver implements DisplayTermResolver {

    private final VocabularyRegistry registry;

    @Inject
    public DefaultDisplayTermResolver(VocabularyRegistry registry) {
        this.registry = registry;
    }
}
```

`@DefaultBean` so consumers can displace with `@Alternative` when the service moves to platform or when a deployment needs custom resolution logic.

---

## Testing

### Unit tests (DefaultDisplayTermResolver)

| Test | Setup | Expected |
|---|---|---|
| Direct resolution — value found | Register test vocab, resolve with sourceVocabUri | Returns `term.label()` |
| Direct resolution — value not found | Resolve unknown value | Returns raw value |
| Direct resolution — vocabUri not registered | Resolve with unknown URI | Returns raw value |
| Cross-vocab swap — match found | Two vocabs with exactMatch, resolve with targetVocabUri | Returns target `term.label()` |
| Cross-vocab swap — no match | Two vocabs without mapping, resolve with targetVocabUri | Falls back to source `term.label()` |
| Cross-vocab swap — target vocab not registered | Resolve with unknown targetVocabUri | Falls back to source `term.label()` |
| Axis-aware swap — match found | Two vocabs with axisExactMatch, resolve with axis | Returns target `term.label()` |
| Axis-aware swap — axis has no mapping | Resolve with axis that returns empty | Falls back to source `term.label()` |
| Auto-discovery — sourceVocabUri null, value found | Register test vocab, resolve with null sourceUri | Returns `term.label()` |
| Auto-discovery — sourceVocabUri null, value not found | Resolve unknown value with null sourceUri | Returns raw value |
| Same source and target vocab | Resolve with sourceUri == targetUri | Returns source `term.label()` (no cross-vocab) |
| Null value | Resolve null | Returns null |

Test infrastructure: register test vocabulary enums with `VocabularyRegistry` in `@BeforeEach`. Use the existing `@QuarkusTest` pattern from `DefaultCapabilityHealthTest` (CDI-wired registry). Create a pair of test vocabularies with `exactMatch` and `axisExactMatch` mappings to test cross-vocab swap.

---

## Module Changes Summary

| Module | Change |
|---|---|
| `api/` | New `DisplayTermResolver` interface (3 methods: primary + 2 default overloads) |
| `runtime/` | New `DefaultDisplayTermResolver` `@DefaultBean @ApplicationScoped` in `runtime/display/` |
| All other modules | No changes |

---

## Not in Scope

- **Tenant-level default vocabulary** — tenancy is legal/customer separation, not a vocabulary preference. Vocabulary context comes from domain objects.
- **Render-target-specific labels** — all render targets get the same label for now
- **Label overrides beyond vocabulary** — per-deployment customization without vocabulary changes
- **SystemPromptRenderer integration** — follow-on; renderer already has VocabularyRegistry and can adopt DisplayTermResolver when ready
- **Move to platform-api** — tracked in casehubio/platform#283; this spec builds the initial implementation in eidos

---

## References

- `VocabularyRegistry.java:30-36` — resolve() and equivalentValues() (axis-aware + unaware)
- `VocabularyTerm.java:16-17` — value() and label()
- `VocabularyTerm.java:27` — exactMatch() for axis-unaware cross-vocab
- `VocabularyTerm.java:44` — axisExactMatch() for axis-aware cross-vocab
- `AgentDescriptor.java:112-118` — vocabUriForSlot() resolution chain
- `AgentDescriptor.java:118+` — vocabUriForAxis() resolution chain
- `CdiVocabularyRegistry.java:418-422` — resolve implementation (byUri → byClass index)
- `CdiVocabularyRegistry.java:431-438` — equivalentValues implementation
- casehubio/platform#283 — eventual platform-api home
- casehubio/blocks-ui#157 — org diagram consumer (primary motivating use case)
- D1-D5 decision records (`decisions.md`)
