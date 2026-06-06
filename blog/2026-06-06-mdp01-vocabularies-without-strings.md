---
layout: post
title: "Vocabularies Without Strings"
date: 2026-06-06
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-eidos]
tags: [vocabulary, enums, type-safety, java, eidos]
---

eidos#40 started as a small API gap. It became something larger.

The request was straightforward: `equivalentValues(fromVocab, value, toVocab)` had no axis parameter. A DISC type maps to different Conscientiousness values on each disposition axis — `dominance` maps to `independent` on `socialOrient` but `flexible` on `ruleFollowing`. Without an axis, there was no way to resolve which.

The obvious fix was adding `DispositionAxis axis` as a fourth parameter. I started there. But before writing a line of code, something about the existing API stopped me: the terms themselves. `equivalentValues` returned `Set<String>`. The cross-vocabulary mappings were `Map<String, String>` — target vocab URI as key, equivalent term name as value. The vocabulary records held `Map<String, VocabularyTerm>` internally.

Everything was stringly-typed. Vocabulary terms are defined in code — they're not discovered at runtime from external sources. `ConscientiousnessTerm.BOLD` doesn't come from a database. It's a constant. So why was it stored as a string?

The answer was: because nobody had asked.

## The redesign

`VocabularyTerm` became an interface. Vocabulary modules define their terms as enum constants implementing it. `SvoTerm.EVALUATOR`, `CasehubSlotTerm.REVIEWER`, `ConscientiousnessTerm.BOLD` — all compile-time constants. Cross-vocabulary references that used to be `Map.of("urn:casehub:vocab:casehubslot", "reviewer")` are now method overrides returning typed constants:

```java
EVALUATOR("evaluator", "Evaluator", "Assesses quality of work", List.of("reviewer", "judge")) {
    @Override public Optional<VocabularyTerm> exactMatch(Class<?> t) {
        return t == CasehubSlotTerm.class
            ? Optional.of(CasehubSlotTerm.REVIEWER)
            : Optional.empty();
    }
}
```

Rename `CasehubSlotTerm.REVIEWER` using the IDE's refactoring and the compiler tells you about this reference. String maps give you nothing.

The axis-aware variant uses an exhaustive switch with no `default` branch:

```java
DOMINANCE("dominance", "Dominance", "High-D: direct, results-oriented", List.of("D")) {
    @Override
    public Optional<VocabularyTerm> axisExactMatch(Class<?> t, DispositionAxis axis) {
        if (t != ConscientiousnessTerm.class) return Optional.empty();
        return switch (axis) {
            case SOCIAL_ORIENTATION -> Optional.of(ConscientiousnessTerm.INDEPENDENT);
            case RULE_FOLLOWING     -> Optional.of(ConscientiousnessTerm.FLEXIBLE);
            case RISK_APPETITE      -> Optional.of(ConscientiousnessTerm.BOLD);
            case AUTONOMY           -> Optional.of(ConscientiousnessTerm.AUTONOMOUS);
        };
    }
}
```

The no-default exhaustive switch is the key design choice. When `CONFLICT_MODE` gets added to `DispositionAxis` (eidos#38 adds `conflictMode` as a fifth axis), the compiler identifies every site that handles disposition axes. Not a grep, not a code search — the type system does the impact analysis. Adding an enum value becomes a type-safe refactor.

## The spec review found things

The first draft had `axisExactMatches()` returning a `Map<Class<?>, Map<DispositionAxis, VocabularyTerm>>` — a map of maps rather than a method. The problem: it forces every implementation to provide a complete mapping or return empty silently. The exhaustive switch method is different. You only override it if you have something to say, and the switch forces you to be explicit about every axis — including the ones where you return `Optional.empty()`.

One subtlety that the review caught: wrapping the switch in `Optional.of(switch {...})` would forbid gaps. DISC.INFLUENCE might not have a clear AUTONOMY mapping. `Optional.empty()` has to be a valid branch, so the pattern needs to be `switch { case X -> Optional.of(...); case Y -> Optional.empty() }` — not the switch nested inside the Optional.

`targetVocab.cast()` was the right fix for the unchecked cast in the registry's typed methods. Instead of `(T) from.exactMatch(targetVocab)`, the cast becomes `from.exactMatch(targetVocab).map(targetVocab::cast)`. Fails immediately at the callsite if a vocabulary implementation returns the wrong type. Deferred ClassCastException somewhere downstream would have been harder to trace.

## A Java type system surprise

Registering vocabulary enums via CDI hit a type inference wall. `VocabularyRegistrar.vocabulary()` returns `Class<? extends Enum<? extends VocabularyTerm>>`. The registry's `register(Class<T>)` expects `T extends Enum<T> & VocabularyTerm`. Those are semantically equivalent — any concrete enum implementing `VocabularyTerm` satisfies both. But Java's type checker can't unify the wildcard form with the recursive self-bound; they use different capture variables.

The fix was a private `registerRaw()` bridge with a scoped `@SuppressWarnings("unchecked")` cast — safe because any real enum type satisfies the bound at its declaration site. Not particularly elegant, but honest about what's happening.

## What's done

`Vocabulary` record: deleted. `VocabularyTerm` record: replaced by the interface of the same name. `CdiVocabularyRegistry`: rewritten to discover `VocabularyRegistrar` CDI beans instead of `@Produces Vocabulary`. The three built-in vocabularies (SVO, Conscientiousness, CasehubSlot) are now enums with typed bidirectional `exactMatch()` overrides.

The registry has two parallel APIs: a typed path that bypasses registration entirely (enum constants carry the mapping data directly), and a string-based path for URIs and term values that come from agent descriptors at runtime. `AgentDisposition` got a `get(DispositionAxis)` method with its own exhaustive switch, so callers can resolve axis values programmatically without knowing field names as strings.

eidos#26 (DISC and Belbin vocabulary module) can now be built on a typed foundation. The question it faced — how do DISC types map to Conscientiousness values per axis — has an answer the compiler will enforce.
