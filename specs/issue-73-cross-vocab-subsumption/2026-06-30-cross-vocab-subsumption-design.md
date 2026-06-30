# Cross-Vocabulary Subsumption Design

**Issue:** casehubio/eidos#73
**Date:** 2026-06-30

## Problem

`VocabularyTerm.specializes()` is intra-vocabulary only. Application-tier vocabularies cannot declare terms that specialize foundation-tier terms (e.g., `clinical-documentation-review` specializing `documentation` from `CasehubCapabilityTerm`). Without this, applications must either pollute foundation vocabularies with domain terms (violating the Application Tier Rule) or duplicate foundation terms in their own vocabulary (fragile).

## Research Basis

The design draws on established ontology/knowledge-organization standards:

- **OWL / OWLS-MX** — Subsumption is global. `rdfs:subClassOf` crosses namespace boundaries freely. OWLS-MX (the basis for eidos's `MatchDegree` model) classifies all service ontologies into a single matchmaker ontology and reasons globally. There is no per-ontology scope for subsumption queries.
- **SKOS** — Distinguishes intra-scheme hierarchy (`broader`/`narrower`) from inter-scheme mappings (`broadMatch`/`narrowMatch`). Mapping properties are explicitly non-transitive to avoid compound errors from imprecise mappings.
- **XKOS** — Refines SKOS with `specializes`/`generalizes` (the direct inspiration for `VocabularyTerm.specializes()`). Uses `Correspondence`/`ConceptAssociation` for cross-classification mappings.
- **Conservativity principle** (Solimando et al., 2017) — Cross-ontology mappings must not introduce new subsumption relations between concepts already in the same source ontology.

**eidos is OWL-like, not SKOS-like.** Vocabularies are Java enums with formal interfaces. `specializes()` declares precise is-a relationships, not informal navigational hierarchies. The match degrees are from OWLS-MX, which operates on formal ontologies. SKOS's rationale for non-transitive cross-scheme mappings (compound errors from imprecise mappings) does not apply — eidos relationships are precise by construction. Therefore, **subsumption in eidos is global, not namespace-scoped.**

## Design Decisions

### 1. Keep the single-URI API signature

`match(vocabUri, declaredValue, requestedValue)` does not change. The `vocabUri` parameter means "from this vocabulary's hierarchy perspective, including cross-vocabulary edges."

**Why not dual-URI?** Consumers (`DefaultCapabilityHealth`, `InMemoryAgentRegistry`) use `capability.capabilityVocabulary()` as the vocabulary URI and pass the requested capability as an unqualified string. The request is intentionally unqualified — the engine asks "who can do documentation?" not "who can do `urn:casehub:vocab:capability#documentation`?" A dual-URI signature would cascade through `AgentQuery`, `ProbeContext`, and `CapabilitySpecializationStore` for zero practical benefit.

### 2. Two-pass registration

Separate term registration from hierarchy computation. Currently, `register()` does both in one pass. With cross-vocabulary edges, a vocabulary's parent terms may not be registered yet (CDI discovery order is arbitrary).

**Pass 1 — Term registration:** Validate vocabulary metadata, constants, duplicates, URI uniqueness. Populate `byUri`, `byClass`, `byClassOrdered`. No hierarchy work.

**Pass 2 — Hierarchy construction:** Called once after all vocabularies are registered. Builds the global DAG, validates cross-vocabulary references, detects cycles, computes ancestor/descendant indexes.

### 3. Per-vocabulary index augmentation via injection

Each vocabulary's ancestor index is augmented with entries for terms from other vocabularies that transitively specialize terms in this vocabulary. This enables bidirectional matching:

- **Specialization match** (declared is more specific than requested): Agent declares `clinical-documentation-review` in clinical vocab. Query asks for `documentation`. The clinical vocab's ancestor index includes `documentation` (from foundation vocab) as an ancestor of `clinical-documentation-review`. → Specialization(1).

- **Plugin match** (declared is more general than requested): Agent declares `documentation` in foundation vocab. Query asks for `clinical-documentation-review`. The foundation vocab's ancestor index is augmented with an entry for `clinical-documentation-review` → ancestors include `documentation`. → Plugin(1).

The conservativity principle is satisfied: injected terms are new to the target vocabulary's index. No new subsumption relations are created between terms already in the same vocabulary.

## API Surface — No Changes

| Method | Signature Change | Behavior Change |
|--------|-----------------|-----------------|
| `VocabularyTerm.specializes()` | None | Can now return terms from other vocabularies (was rejected at registration; type was always compatible) |
| `VocabularyRegistry.match()` | None | Traverses cross-vocabulary hierarchy transparently |
| `VocabularyRegistry.subsumes()` | None | Traverses cross-vocabulary hierarchy transparently |
| `VocabularyRegistry.ancestors()` | None | Returns cross-vocabulary ancestors |
| `VocabularyRegistry.descendants()` | None | Returns cross-vocabulary descendants |
| `VocabularyRegistry.expandForMatchingByVocabulary()` | None | Return map may contain additional vocabulary keys for cross-vocabulary terms |
| `VocabularyRegistry.register()` | None | After `@PostConstruct`, triggers full hierarchy rebuild |
| `VocabularyRegistry.resolve()` | None | Unchanged — resolution stays per-vocabulary |

## Internal Changes

### `AncestorEntry` / `DescendantEntry`

Add `vocabUri` field to record which vocabulary declared the term:

```java
private record AncestorEntry(VocabularyTerm term, int depth, String vocabUri) {}
private record DescendantEntry(VocabularyTerm term, int depth, String vocabUri) {}
```

Used by `expandForMatchingByVocabulary()` to group cross-vocabulary terms under their declaring vocabulary URI.

### `CdiVocabularyRegistry` refactoring

**`@PostConstruct init()`:**
```
for each registrar:
    registerTerms(registrar.vocabulary())
buildAllHierarchyIndexes()
```

**`registerTerms()`:** Current `register()` up to the `byUri.put()` call. No hierarchy work.

**`buildAllHierarchyIndexes()`:**

1. **Build global edge map.** For every term T in every registered vocabulary, collect `T.specializes()` edges. For cross-vocabulary edges, resolve the parent term's vocabulary URI via `((Enum<?>) parent).getDeclaringClass()` → reverse lookup in `byUri`.

2. **Validate cross-vocabulary references.** For each cross-vocabulary edge, verify the parent term's vocabulary is registered. Fail fast if not.

3. **Cycle detection.** Kahn's algorithm over the global DAG (all terms from all vocabularies as nodes).

4. **Compute global ancestor and descendant maps.** For each term, BFS through the global DAG. Each entry records `(VocabularyTerm, depth, declaringVocabUri)`.

5. **Populate per-vocabulary indexes:**
   - **Ancestor index for V:** entries for every term declared in V (full global ancestor list) + entries for every term from other vocabularies that has at least one ancestor in V (injection — full global ancestor list).
   - **Descendant index for V:** entries for every term declared in V (full global descendant list including cross-vocabulary descendants).
   - **`valueToVocabs`:** only the declaring vocabulary for each term (no injection).

**`register()` (public API):** When called after `@PostConstruct`, calls `registerTerms()` then `buildAllHierarchyIndexes()`. Rebuilding all indexes is necessary because a new vocabulary might introduce cross-vocabulary edges affecting existing vocabularies.

### `expandForMatchingByVocabulary()` implementation change

When iterating ancestor and descendant entries, group each by its `vocabUri` field (from the augmented records). Cross-vocabulary terms land under their declaring vocabulary's key in the result map.

Example — expanding `"documentation"`:
```
"urn:casehub:vocab:capability" → {"documentation", "security-code-review", ...}
"urn:clinical:vocab:capability" → {"clinical-documentation-review"}
```

Each term appears under its declaring vocabulary URI because that is how agent capabilities are qualified. Any registry implementation matching capabilities by (vocabulary, name) pairs gets correct cross-vocabulary matching.

## Validation Rules

| Rule | When | Severity |
|------|------|----------|
| Parent term's vocabulary must be registered | Hierarchy build (pass 2) | Error — fail fast |
| Parent term must exist in its vocabulary | Hierarchy build (pass 2) | Error — fail fast |
| No cycles in the global DAG | Hierarchy build (pass 2) | Error — fail fast with involved terms |
| Cross-vocabulary edge logged | Hierarchy build (pass 2) | Info — for observability |

## Modules Affected

| Module | Changes |
|--------|---------|
| `api` | None — `VocabularyTerm.specializes()` already returns `List<VocabularyTerm>` |
| `runtime` | `CdiVocabularyRegistry` — two-pass registration, `AncestorEntry`/`DescendantEntry` augmentation, `expandForMatchingByVocabulary` grouping |
| `persistence-memory` | None — `InMemoryAgentRegistry` calls `match()` which is transparently augmented |
| `vocab` | None — `CasehubCapabilityTerm` is unchanged; application vocabularies add cross-vocabulary `specializes()` |
| `eval` | None |
| `deployment` | None |

## Test Strategy

**Unit tests in `CdiVocabularyRegistryTest`:**
- Cross-vocabulary `specializes()` edge accepted (was previously rejected)
- Cycle detection across vocabulary boundaries
- `ancestors()` returns cross-vocabulary ancestors
- `descendants()` returns cross-vocabulary descendants
- `subsumes()` across vocabulary boundaries
- `match()` — Specialization match when declared is app-specific, requested is foundation
- `match()` — Plugin match when declared is foundation, requested is app-specific
- `expandForMatchingByVocabulary()` returns cross-vocabulary terms under their declaring vocabulary URI
- Registration order independence (register app vocab before foundation, then foundation — same result)
- Validation: cross-vocabulary reference to unregistered vocabulary fails
- Validation: cross-vocabulary reference to nonexistent term fails
- Dynamic `register()` after init triggers hierarchy rebuild with cross-vocabulary edges

**Integration tests:**
- `CapabilityVocabularyIntegrationTest` — cross-vocabulary subsumption scenario with `CasehubCapabilityTerm` as foundation and a test vocabulary as application tier

## Out of Scope

- **Cross-vocabulary `exactMatch()`** — already works (delegates to VocabularyTerm method, no registry involvement)
- **`CapabilityVocabularyValidator` changes** — validates that a capability's name exists in its declared vocabulary. Cross-vocabulary subsumption doesn't change this — the capability is still grounded in its declaring vocabulary.
- **Schema/migration changes** — no persistence changes needed; hierarchy is computed at startup from enum declarations
- **Reactive registry parity** — `DefaultReactiveCapabilityHealth` delegates to the blocking `VocabularyRegistry`; no separate changes needed
