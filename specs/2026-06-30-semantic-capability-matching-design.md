# Semantic Capability Matching — Subsumption in VocabularyRegistry

**Issue:** casehubio/eidos#71  
**Date:** 2026-06-30  
**Status:** Approved

## Problem

AgentRegistry.find(AgentQuery) and the engine's AgentCandidateFactory both do exact string matching on capability names. An agent declaring `code-review` does not match a query for `security-code-review`, even though code-review structurally subsumes security-code-review. The epistemic domain, exclusion, and semantic routing layers never fire because the candidate set is empty.

Modern agent frameworks (A2A, AutoGen, CrewAI) have the same limitation — flat string tags with no hierarchy. The richest prior art is OWLS-MX (Klusch/Sycara, 2002-2012), which defines graded match degrees over OWL-DL concept subsumption hierarchies. Pure subsumption proved insufficient in practice; hybrid approaches (subsumption + IR similarity) are the consensus. CaseHub already has the hybrid fallback via SemanticAgentRoutingStrategy — this design adds the structural subsumption layer.

## Approach

Extend VocabularyTerm with XKOS-style `specializes()` relationships. VocabularyRegistry gains subsumption queries with depth-aware match degrees (OWLS-MX-informed). Capability names become optionally vocabulary-grounded via a new field on AgentCapability. AgentRegistry.find() expands queries via subsumption when vocabulary information is available.

No RDF, no triples, no description logic reasoner. The XKOS semantic model (specializes/generalizes as explicit is-a relationships) informs the Java data model. Hierarchies are declared via method overrides on enum constants. The DAG is precomputed at registration time — all lookups are O(1).

## Design

### 1. VocabularyTerm.specializes()

One new default method on VocabularyTerm:

```java
default List<VocabularyTerm> specializes() { return List.of(); }
```

**Semantics:** "This term is a specialization of the returned terms." XKOS `specializes` — explicit is-a / kind-of relationship.

**DAG, not tree.** A term can specialize multiple terms. `SAST_REVIEW` can specialize both `SECURITY_CODE_REVIEW` and `STATIC_ANALYSIS`.

**Intra-vocabulary only.** `specializes()` returns terms from the same enum/vocabulary. Cross-vocabulary relationships use the existing `exactMatch()` / `axisExactMatch()` mechanism. The registry validates this at registration time — a term referencing a constant from a different enum class is an error.

### 2. MatchDegree

New sealed interface in `io.casehub.eidos.api`:

```java
public sealed interface MatchDegree
        permits MatchDegree.Exact, MatchDegree.Plugin,
                MatchDegree.Specialization, MatchDegree.None {

    /** Declared capability equals requested capability. */
    record Exact() implements MatchDegree {}

    /** Declared is an ancestor of requested — agent is more general, can handle it.
     *  depth=1 means immediate parent. */
    record Plugin(int depth) implements MatchDegree {}

    /** Declared is a descendant of requested — agent is more specific than needed.
     *  depth=1 means immediate child. */
    record Specialization(int depth) implements MatchDegree {}

    /** No structural relationship. */
    record None() implements MatchDegree {}
}
```

**OWLS-MX mapping:** Exact = Exact. Plugin(n) unifies Plug-in (n=1) and Subsumes (n>1) with depth as discriminator. Specialization(n) = Subsumed-by. None = the boundary where structural matching stops and SemanticAgentRoutingStrategy takes over.

**Depth is information, not a cutoff.** The consumer decides what to accept. `Plugin(1)` is high-confidence. `Plugin(5)` is sketchy. No threshold baked in.

### 3. VocabularyRegistry Extensions

Four new methods:

```java
boolean subsumes(String vocabUri, String generalValue, String specificValue);

MatchDegree match(String vocabUri, String declaredValue, String requestedValue);

List<? extends VocabularyTerm> ancestors(String vocabUri, String value);

List<? extends VocabularyTerm> descendants(String vocabUri, String value);

Set<String> expandForMatching(String value);
```

**`subsumes()`** — true if generalValue is an ancestor of specificValue, or they are equal.

**`match()`** — computes match degree: if declared == requested → Exact; if declared ∈ ancestors(requested) → Plugin(depth); if declared ∈ descendants(requested) → Specialization(depth); otherwise → None.

**`ancestors()`** / **`descendants()`** — transitive navigation, ordered by depth (immediate first). Empty list for unknown terms.

**`expandForMatching()`** — given a term, returns {term} ∪ ancestors ∪ descendants across all registered vocabularies that contain it. This is the bridge to the SQL query layer — the JPA registry uses it to expand `c.name IN :expanded`.

**Error handling:** unknown vocabulary or unknown term returns false / None / empty. No exceptions. Consistent with `resolve()` returning `Optional.empty()`.

### 4. CdiVocabularyRegistry DAG Index

All hierarchy data is precomputed at registration time. Lookups are O(1) map access.

**New internal data structures:**

```java
// value → set of vocabUris containing this value (reverse index for expandForMatching)
ConcurrentHashMap<String, Set<String>> valueToVocabs;

// vocabUri → (value → list of ancestors with depth, ordered depth-first)
ConcurrentHashMap<String, Map<String, List<AncestorEntry>>> ancestorIndex;

// vocabUri → (value → list of descendants with depth, ordered depth-first)
ConcurrentHashMap<String, Map<String, List<DescendantEntry>>> descendantIndex;

private record AncestorEntry(VocabularyTerm term, int depth) {}
private record DescendantEntry(VocabularyTerm term, int depth) {}
```

**Registration-time build** (inside `register()`, after existing validation):

1. Walk `specializes()` for each constant. Validate: every referenced term is a constant of the same enum class.
2. Cycle detection via topological sort. Cycle → `IllegalArgumentException`.
3. BFS from each term to compute transitive ancestors with minimum depth (shortest path in DAG).
4. Invert edges, BFS to compute transitive descendants with minimum depth.
5. Populate `valueToVocabs` reverse index.

**Thread safety:** Same model as existing maps — single-threaded registration at `@PostConstruct`, concurrent reads after.

**No external dependency.** BFS with a visited set. No JGraphT.

### 5. AgentCapability Vocabulary Grounding

AgentCapability gains one optional field:

```java
public record AgentCapability(
        String name,
        String capabilityVocabulary,    // optional vocabulary URI
        Double qualityHint,
        // ... rest unchanged
) { ... }
```

**Per-capability, not per-descriptor.** An agent may declare capabilities from different vocabularies.

**Validation:** Format validation in compact constructor (optional string length). Existence validation (name is a valid term in the declared vocabulary) at registration time in `DescriptorCollector`, which has VocabularyRegistry access.

**Builder:** gains `capabilityVocabulary(String v)`.

**JPA:** `AgentCapabilityEntity` gains nullable `capability_vocabulary` column. Mapper maps it through.

**YAML:**

```yaml
capabilities:
  - name: code-review
    capabilityVocabulary: "urn:casehub:vocab:capability"
    qualityHint: 0.9
```

### 6. AgentRegistry.find() with Subsumption

**SPI unchanged.** `find(AgentQuery)` returns `List<AgentDescriptor>`. Behavior changes — includes subsumption matches.

**InMemoryAgentRegistry:** Injects `VocabularyRegistry`. Capability filter changes from exact string match to:

```java
private boolean matchesCapability(AgentCapability capability, String requestedName) {
    if (capability.name().equals(requestedName)) return true;
    if (capability.capabilityVocabulary() == null) return false;
    MatchDegree degree = vocabularyRegistry.match(
        capability.capabilityVocabulary(), capability.name(), requestedName);
    return !(degree instanceof MatchDegree.None);
}
```

**JpaAgentRegistry:** Injects `VocabularyRegistry`. Before building JPQL, expands the capability name via `expandForMatching()`:

```java
Set<String> capabilityNames = query.capabilityName() != null
    ? vocabularyRegistry.expandForMatching(query.capabilityName())
    : Set.of();

// JPQL uses IN clause when expanded, = when not
if (capabilityNames.size() > 1) {
    jpql.append(" AND c.name IN :capabilityNames");
} else {
    jpql.append(" AND c.name = :capabilityName");
}
```

**AgentQuery unchanged.** No new fields. Vocabulary context comes from registered capabilities, not from the query.

**Reactive registries** get the same changes, wrapped in Uni.

**Match degree for ranking:** consumers call `VocabularyRegistry.match()` directly when they need degrees. The registry does discovery; routing does ranking.

### 7. Starter Capability Vocabulary

`CasehubCapabilityTerm` in `casehub-eidos-vocab`:

```
code-review
├── security-code-review
│   └── sast-review ──┐
└── performance-code-review   │
                              │
analysis                      │
└── static-analysis ──────────┘

testing
documentation
```

`sast-review` specializes both `security-code-review` and `static-analysis` — DAG in action.

Registrar bean follows existing pattern (`CasehubCapabilityRegistrar implements VocabularyRegistrar`).

URI: `urn:casehub:vocab:capability`.

## Out of Scope

- **Engine integration** — `AgentCandidateFactory.buildCandidates()` in casehub-engine does its own exact match (`w.capabilityNames().contains(capabilityName)`). Changing it to use subsumption is a separate issue against casehub-engine.
- **CapabilityHealth.probe() integration** — probe step 2 checks `c.name().equals(capabilityTag)`. Updating to use subsumption is a follow-up.
- **Cross-vocabulary subsumption** — terms specializing terms in different vocabularies. Future extension; `exactMatch()` handles cross-vocabulary equivalence for now.
- **I/O concept matching** — matching on inputTypes/outputTypes (the OWLS-MX I/O matching model). AgentCapability already carries these; subsumption over them is a separate concern.
- **Match degree in AgentQuery results** — returning `List<MatchedDescriptor>` instead of `List<AgentDescriptor>`. Deferred; match degree is computed via `VocabularyRegistry.match()` by consumers.

## Module Impact

| Module | Changes |
|--------|---------|
| `casehub-eidos-api` | VocabularyTerm.specializes(), MatchDegree sealed interface, VocabularyRegistry new methods, AgentCapability.capabilityVocabulary |
| `casehub-eidos` (runtime) | CdiVocabularyRegistry DAG index, JpaAgentRegistry subsumption expansion, JpaReactiveAgentRegistry, AgentCapabilityEntity column, DescriptorCollector validation |
| `casehub-eidos-memory` | InMemoryAgentRegistry subsumption matching, InMemoryReactiveAgentRegistry |
| `casehub-eidos-vocab` | CasehubCapabilityTerm enum, CasehubCapabilityRegistrar |

## Research Backing

- OWLS-MX (Klusch, Fries, Sycara — Journal of Web Semantics, 2009): Graded match degrees, hybrid subsumption + IR, index-at-registration
- XKOS (DDI Alliance): specializes/generalizes as sub-properties of SKOS broader/narrower — explicit is-a semantics without RDF infrastructure
- SkillNet (arXiv 2603.04448, Feb 2026): Three-layer skill ontology with typed relations; only modern LLM-agent system with concrete hierarchy
- OWL-S (Martin et al., W3C 2004/2007): Service discovery as subsumption tests between capability profiles
- A2A (Google, 2025): Flat string tags, no hierarchy — confirms the gap this design fills
