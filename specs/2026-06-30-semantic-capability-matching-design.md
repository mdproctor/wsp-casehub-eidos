# Semantic Capability Matching — Subsumption in VocabularyRegistry

**Issue:** casehubio/eidos#71  
**Date:** 2026-06-30  
**Status:** Approved

## Problem

`AgentRegistry.find(AgentQuery)` does exact string matching on capability names. An agent declaring `code-review` does not match a query for `security-code-review`, even though code-review structurally subsumes security-code-review. Direct registry callers (claudony, application-tier consumers) get empty results for semantically valid queries. `CapabilityHealth.probe()` has the same exact-match limitation — subsumption-discovered agents would be rejected at health probe time without a coordinated fix. The engine's `AgentCandidateFactory` has a separate exact-match path that bypasses `AgentRegistry` entirely — engine integration is a follow-up against casehub-engine.

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

**OWLS-MX mapping:** Exact = Exact. Plugin(n) = Plug-in at depth n (agent capability is an ancestor of requested — more general, can handle it). Specialization(n) = Subsumes at depth n (agent capability is a descendant of requested — more specific than needed). None = the boundary where structural matching stops and SemanticAgentRoutingStrategy takes over.

**Depth is information, not a cutoff.** The consumer decides what to accept. `Plugin(1)` is high-confidence. `Plugin(5)` is sketchy. No threshold baked in.

### 3. VocabularyRegistry Extensions

Four new methods:

```java
boolean subsumes(String vocabUri, String generalValue, String specificValue);

MatchDegree match(String vocabUri, String declaredValue, String requestedValue);

List<? extends VocabularyTerm> ancestors(String vocabUri, String value);

List<? extends VocabularyTerm> descendants(String vocabUri, String value);

Map<String, Set<String>> expandForMatchingByVocabulary(String value);
```

**`subsumes()`** — true if generalValue is an ancestor of specificValue, or they are equal.

**`match()`** — computes match degree: if declared == requested → Exact; if declared ∈ ancestors(requested) → Plugin(depth); if declared ∈ descendants(requested) → Specialization(depth); otherwise → None.

**`ancestors()`** / **`descendants()`** — transitive navigation, ordered by depth (immediate first). Empty list for unknown terms.

**`expandForMatchingByVocabulary()`** — given a term, returns a map from vocabulary URI to {term} ∪ ancestors ∪ descendants within that vocabulary. Scoped per-vocabulary to prevent hierarchy conflation across unrelated vocabularies. The JPA registry uses this to build vocabulary-scoped conditions: `(c.capability_vocabulary = :vocab AND c.name IN :expanded)` per vocabulary.

**Error handling:** unknown vocabulary or unknown term returns false / None / empty. No exceptions. Consistent with `resolve()` returning `Optional.empty()`.

### 4. CdiVocabularyRegistry DAG Index

All hierarchy data is precomputed at registration time. Lookups are O(1) map access.

**New internal data structures:**

```java
// value → set of vocabUris containing this value (reverse index for expandForMatchingByVocabulary)
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

**Validation:** Format validation in compact constructor (optional string length). Existence validation (name is a valid term in the declared vocabulary) at registration time in `AgentRegistry.register()`, which has `VocabularyRegistry` access via CDI injection. Both JPA and InMemory registries validate before persisting. `DescriptorCollector` is unchanged — it validates duplicate agentId+tenancyId pairs only and has no VocabularyRegistry access.

**Builder:** gains `capabilityVocabulary(String v)`.

**JPA:** `AgentCapabilityEntity` gains nullable `capability_vocabulary` column, added directly to `V1__initial_schema.sql` (no deployed instances — ARC42STORIES §7). No V6 migration. Mapper maps it through.

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

**JpaAgentRegistry:** Injects `VocabularyRegistry`. Before building JPQL, expands the capability name per vocabulary via `expandForMatchingByVocabulary()`:

```java
Map<String, Set<String>> vocabExpansions = query.capabilityName() != null
    ? vocabularyRegistry.expandForMatchingByVocabulary(query.capabilityName())
    : Map.of();

if (vocabExpansions.isEmpty()) {
    // No vocabulary knows this term — exact match only
    jpql.append(" AND c.name = :capabilityName");
} else {
    // Exact match for ungrounded capabilities + vocabulary-scoped subsumption
    jpql.append(" AND (c.name = :capabilityName");
    int idx = 0;
    for (var entry : vocabExpansions.entrySet()) {
        jpql.append(" OR (c.capabilityVocabulary = :vocab" + idx
            + " AND c.name IN :expanded" + idx + ")");
        idx++;
    }
    jpql.append(")");
}
```

**Behavioral equivalence with InMemoryAgentRegistry:** Both paths produce identical results:
- Ungrounded capabilities (`capabilityVocabulary == null`) match by exact name only — no subsumption
- Grounded capabilities match by exact name OR by subsumption within the agent's declared vocabulary

**AgentQuery unchanged.** No new fields. Vocabulary context comes from registered capabilities, not from the query. Subsumption is always scoped by the agent's declared `capabilityVocabulary`, not by the query.

**Reactive registries** get the same changes, wrapped in Uni.

**Match degree for ranking:** consumers call `VocabularyRegistry.match()` directly when they need degrees. The registry does discovery; routing does ranking.

### 7. CapabilityHealth.probe() Integration

`DefaultCapabilityHealth` gains `VocabularyRegistry` injection. The capability lookup in step 2 becomes subsumption-aware:

```java
private AgentCapability findCapability(List<AgentCapability> capabilities,
                                        String capabilityTag) {
    // Exact match first
    for (var c : capabilities) {
        if (c.name().equals(capabilityTag)) return c;
    }
    // Subsumption match — find declared capability that covers the requested one
    for (var c : capabilities) {
        if (c.capabilityVocabulary() != null) {
            MatchDegree degree = vocabularyRegistry.match(
                c.capabilityVocabulary(), c.name(), capabilityTag);
            if (!(degree instanceof MatchDegree.None)) return c;
        }
    }
    return null;
}
```

Once the matching capability is found, all downstream probe steps (degradation, exclusion, epistemic check) apply to it unchanged. `DefaultReactiveCapabilityHealth` delegates to the imperative implementation — no separate change needed.

### 8. Starter Capability Vocabulary

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

- **Engine integration** — `AgentCandidateFactory.buildCandidates()` in casehub-engine does its own exact match (`w.capabilityNames().contains(capabilityName)`), bypassing `AgentRegistry.find()` entirely. Changing it to use subsumption is tracked as a casehub-engine issue (see Follow-Up Issues below). This spec delivers subsumption for direct `AgentRegistry.find()` callers (claudony, application-tier consumers) and for `CapabilityHealth.probe()`.
- **Cross-vocabulary subsumption** — terms specializing terms in different vocabularies. `specializes()` is intra-vocabulary only by design. Applications can define their own vocabulary with internal hierarchies; subsumption works within each vocabulary independently. Cross-vocabulary subsumption (e.g., application term specializing a foundation term) is a future extension tracked as a casehub-eidos issue. `exactMatch()` handles cross-vocabulary equivalence (not subsumption) for now.
- **I/O concept matching** — matching on inputTypes/outputTypes (the OWLS-MX I/O matching model). AgentCapability already carries these; subsumption over them is a separate concern.
- **Match degree in AgentQuery results** — returning `List<MatchedDescriptor>` instead of `List<AgentDescriptor>`. Deferred; match degree is computed via `VocabularyRegistry.match()` by consumers.

## Follow-Up Issues

| Issue | Repo | Blocks |
|-------|------|--------|
| [engine#611](https://github.com/casehubio/engine/issues/611) — AgentCandidateFactory subsumption support | casehub-engine | End-to-end engine dispatch via subsumption |
| [eidos#73](https://github.com/casehubio/eidos/issues/73) — Cross-vocabulary subsumption | casehub-eidos | Application-tier capability hierarchies extending foundation terms |

## Module Impact

| Module | Changes |
|--------|---------|
| `casehub-eidos-api` | VocabularyTerm.specializes(), MatchDegree sealed interface, VocabularyRegistry new methods, AgentCapability.capabilityVocabulary |
| `casehub-eidos` (runtime) | CdiVocabularyRegistry DAG index, JpaAgentRegistry vocabulary-scoped subsumption, JpaReactiveAgentRegistry, AgentCapabilityEntity column (in V1), DefaultCapabilityHealth subsumption-aware probe, AgentRegistry.register() vocabulary validation |
| `casehub-eidos-memory` | InMemoryAgentRegistry subsumption matching, InMemoryReactiveAgentRegistry |
| `casehub-eidos-vocab` | CasehubCapabilityTerm enum, CasehubCapabilityRegistrar |

## Research Backing

- OWLS-MX (Klusch, Fries, Sycara — Journal of Web Semantics, 2009): Graded match degrees, hybrid subsumption + IR, index-at-registration
- XKOS (DDI Alliance): specializes/generalizes as sub-properties of SKOS broader/narrower — explicit is-a semantics without RDF infrastructure
- SkillNet (arXiv 2603.04448, Feb 2026): Three-layer skill ontology with typed relations; only modern LLM-agent system with concrete hierarchy
- OWL-S (Martin et al., W3C 2004/2007): Service discovery as subsumption tests between capability profiles
- A2A (Google, 2025): Flat string tags, no hierarchy — confirms the gap this design fills
