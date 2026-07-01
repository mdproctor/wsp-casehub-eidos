---
layout: post
title: "Subsumption Crosses the Fence"
date: 2026-07-01
entry_type: note
subtype: diary
projects: [eidos]
tags: [vocabulary, subsumption, ontology, cross-vocabulary]
---

The vocabulary system we built for eidos had a deliberate constraint: `specializes()` could only reference terms within the same vocabulary. An application couldn't say "my `clinical-documentation-review` is a kind of `documentation`" if `documentation` lived in the foundation vocabulary. The constraint was there because the hierarchy index was per-vocabulary — there was nowhere to store a cross-boundary edge.

The problem is architectural, not technical. The Application Tier Rule in PLATFORM.md says domain logic belongs in application repos. Without cross-vocabulary subsumption, applications that need specialised capability terms face two bad choices: pollute the foundation vocabulary with domain terms, or duplicate foundation terms in their own vocabulary and hope they stay in sync. Neither is acceptable.

I spent time in the literature before proposing anything. The question was whether eidos's vocabulary system is more like OWL (formal ontology, global subsumption) or SKOS (informal knowledge organisation, namespace-scoped hierarchies with weaker cross-scheme mappings). SKOS explicitly makes its cross-scheme properties non-transitive — the rationale is avoiding compound errors from chaining imprecise mappings. But eidos relationships are precise by construction: Java enums, compile-time declarations, not thesaurus-style navigational aids. The match degrees come from OWLS-MX, which operates on formal OWL ontologies with a single merged knowledge base. That settled it — subsumption in eidos is global.

The design has three moving parts. First, two-pass registration: separate term storage from hierarchy computation, because CDI discovers vocabulary beans in arbitrary order and a child vocabulary's parent might not be registered yet. Second, a global DAG built across all vocabulary boundaries — cycle detection, ancestor/descendant BFS, the whole graph computed once after all vocabularies are in. Third, per-vocabulary index injection: when an application term specializes a foundation term, the application term gets injected into the foundation vocabulary's ancestor index. This is what makes `match()` bidirectional without changing its signature — the `vocabUri` parameter becomes "from this vocabulary's hierarchy perspective" rather than "both terms must belong to this vocabulary."

The conservativity principle from ontology alignment research was the correctness test. Cross-vocabulary mappings must not introduce new subsumption relations between concepts already in the same source ontology. Injection satisfies this: injected terms are new to the target vocabulary's index, so no existing intra-vocabulary relationships change.

One constraint the injection mechanism imposes: term values must be unique within a connected hierarchy. Two independent vocabularies can both define `review` — their indexes never intersect. But if vocabulary B's `review` specializes anything in vocabulary A (which also has `review`), the injection would overwrite A's native entry. We catch this inline during injection — before a second `put()` can silently destroy evidence of the first — and the error names both colliding terms and their declaring vocabularies. The fix is straightforward: use distinct values (`clinical-review` instead of `review`). Same-named terms with different semantics in the same hierarchy is a vocabulary design error, and catching it at registration time forces vocabulary designers to be explicit.

The design review also surfaced the shallow-copy problem in late `register()` rollback. The backup of `valueToVocabs` shared mutable `Set<String>` references with the live map. If the hierarchy rebuild mutated a set and then failed, the backup was already corrupted — restoring it restored garbage. Deep-copying the sets on save fixed it.

The API didn't change at all. `match()`, `subsumes()`, `ancestors()`, `descendants()` — same signatures, same consumers. `expandForMatchingByVocabulary()` now returns maps spanning multiple vocabulary URIs, but the JPA registry already builds per-vocabulary OR clauses from the map entries.

What this enables: an application like clinical or devtown can define `clinical-documentation-review` specializing `CasehubCapabilityTerm.DOCUMENTATION`, and the engine's capability matching finds agents with either term when querying for the other. The hierarchy is one connected graph, regardless of which module defined which term.
