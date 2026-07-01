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

Add `vocabUri` field to record the term's **declaring vocabulary** — the vocabulary whose enum class declares this constant. This is a fixed property of the term (derivable via `((Enum<?>) term).getDeclaringClass()` → reverse lookup in `byUri`), not the vocabulary through which the ancestor was reached during traversal. In diamond inheritance cases where a term is reachable through multiple paths, `vocabUri` is always the same — the term's own declaring vocabulary.

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

5. **Compute per-vocabulary indexes into local maps:**
   - **Ancestor index for V:** entries for every term declared in V (full global ancestor list) + entries for every term from other vocabularies that has at least one **transitive** ancestor in V (injection — full global ancestor list). "Transitive" means: if Foundation → Mid → App are three vocabularies where App specializes Mid and Mid specializes Foundation, then Foundation's ancestor index includes App terms even though App doesn't directly specialize Foundation.
   - **Descendant index for V:** entries for every term declared in V (full global descendant list including cross-vocabulary descendants).
   - **`valueToVocabs`:** only the declaring vocabulary for each term (no injection).

6. **Validate all computed indexes (collision check).** Iterate every computed per-vocabulary index. For each vocabulary V, verify no injected term has the same string value as any term already in V's index (native or previously injected). The per-vocabulary index is keyed by term value (string), so a collision would overwrite existing entries and corrupt subsumption semantics. Example: if Foundation defines `review` (no ancestors) and Clinical defines `review` specializing Foundation's `documentation`, injecting Clinical's `review` into Foundation's index would overwrite Foundation's native `review` entry — causing `subsumes("foundation-uri", "documentation", "review")` to spuriously return `true` for Foundation's own unrelated `review` term, violating conservativity. Fail fast with an error identifying both terms and their declaring vocabularies. This validation runs across ALL vocabularies before any class-level maps are written — partial failure cannot corrupt previously-valid state.

7. **Write indexes to class-level maps.** Only after all validation in steps 2–6 passes, swap the computed local maps into the class-level `ancestorIndex`, `descendantIndex`, and `valueToVocabs`. This ensures atomicity: either all indexes are updated consistently, or none are.

**`register()` (public API):** When called after `@PostConstruct`, calls `registerTerms()` then `buildAllHierarchyIndexes()`. Rebuilding all indexes is necessary because a new vocabulary might introduce cross-vocabulary edges affecting existing vocabularies. The cost is O(V × T) where V is the number of registered vocabularies and T is the total term count across all vocabularies. This is acceptable because vocabulary hierarchies are small (CasehubCapabilityTerm has 8 terms) and late registration is rare — CDI discovery during `@PostConstruct` covers the common case. If `buildAllHierarchyIndexes()` fails (cycle, unresolved reference, or value collision), the new vocabulary's terms are rolled back from `byUri`, `byClass`, and `byClassOrdered` — no partial registration state is visible to concurrent readers.

### `expandForMatchingByVocabulary()` implementation change

**Mechanism:** The method starts from `valueToVocabs.get(value)` (declaring vocabularies only). For each vocabulary, it iterates ancestor and descendant entries from the augmented indexes. Each entry is grouped into the result map by its `vocabUri` field (the entry's declaring vocabulary), NOT by the vocabulary whose index it was read from. This is the key mechanism: cross-vocabulary entries in the index are routed to their own vocabulary's key in the result map.

**Example A — expanding `"documentation"` (foundation term, query finds app-tier agents):**

`valueToVocabs.get("documentation")` → `{"urn:casehub:vocab:capability"}`. Iterating the casehub vocab's descendant index for `"documentation"`, entries include `{clinical-documentation-review, 1, "urn:clinical:vocab:capability"}`. Grouping by entry `vocabUri`:
```
"urn:casehub:vocab:capability" → {"documentation", "security-code-review", ...}
"urn:clinical:vocab:capability" → {"clinical-documentation-review"}
```

**Example B — expanding `"clinical-documentation-review"` (app term, query finds foundation agents):**

`valueToVocabs.get("clinical-documentation-review")` → `{"urn:clinical:vocab:capability"}`. Iterating the clinical vocab's ancestor index for `"clinical-documentation-review"`, entries include `{documentation, 1, "urn:casehub:vocab:capability"}`. Grouping by entry `vocabUri`:
```
"urn:clinical:vocab:capability" → {"clinical-documentation-review"}
"urn:casehub:vocab:capability" → {"documentation"}
```

Both directions produce result maps spanning multiple vocabulary URIs. Any registry implementation matching capabilities by (vocabulary, name) pairs gets correct cross-vocabulary matching — the `InMemoryAgentRegistry` (per-agent `match()`) and JPA registry (`expandForMatchingByVocabulary` → query) paths produce identical results.

**Three-vocabulary example — Foundation → Mid → App:**

Foundation vocab defines `engineering`. Mid vocab defines `software-engineering` specializing `engineering`. App vocab defines `frontend-engineering` specializing `software-engineering`.

Expanding `"engineering"`:
```
"urn:foundation" → {"engineering"}
"urn:mid"        → {"software-engineering"}
"urn:app"        → {"frontend-engineering"}
```

The transitive chain ensures App terms appear even though App doesn't directly specialize Foundation.

## Validation Rules

| Rule | When | Severity |
|------|------|----------|
| Parent term's vocabulary must be registered | Hierarchy build (pass 2) | Error — fail fast |
| Parent term must exist in its vocabulary | Hierarchy build (pass 2) | Error — fail fast |
| No cycles in the global DAG | Hierarchy build (pass 2) | Error — fail fast with involved terms |
| Injected term value must not collide with existing term value in target vocabulary index | Hierarchy build (pass 2, step 5) | Error — fail fast with both terms and declaring vocabularies |
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
- Validation: injected term value colliding with native term value in target vocabulary fails (e.g., both vocabularies define `review`, one specializing a term from the other)
- Late `register()` failure atomicity: register a vocabulary with a value collision, verify the exception, then verify existing vocabularies' indexes are unchanged (no corruption from partial rebuild)
- Dynamic `register()` after init triggers hierarchy rebuild with cross-vocabulary edges

**Integration tests:**
- `CapabilityVocabularyIntegrationTest` — cross-vocabulary subsumption scenario with `CasehubCapabilityTerm` as foundation and a test vocabulary as application tier
- `JpaAgentRegistryTest` — cross-vocabulary capability match via `find(AgentQuery)`: agent registered with foundation capability (`documentation`), query for app-tier term (`clinical-documentation-review`) finds the agent via Plugin match; and the reverse direction (agent with app-tier capability found by foundation-tier query via Specialization match). Verifies JPA query path parity with InMemoryAgentRegistry.

## Out of Scope

- **Cross-vocabulary `exactMatch()`** — already works (delegates to VocabularyTerm method, no registry involvement)
- **`CapabilityVocabularyValidator` changes** — validates that a capability's name exists in its declared vocabulary. Cross-vocabulary subsumption doesn't change this — the capability is still grounded in its declaring vocabulary.
- **Schema/migration changes** — no persistence changes needed; hierarchy is computed at startup from enum declarations
- **Reactive registry parity** — `DefaultReactiveCapabilityHealth` delegates to the blocking `VocabularyRegistry`; no separate changes needed
- **ARC42STORIES.MD update** — the subsumption hierarchy (intra-vocabulary from eidos#71 and cross-vocabulary from eidos#73) is not documented in ARC42STORIES.MD. Filed as eidos#75; broader than this spec's scope
