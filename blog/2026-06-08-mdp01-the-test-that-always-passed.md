---
layout: post
title: "The Test That Always Passed"
date: 2026-06-08
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-eidos]
tags: [vocabulary, testing, assertj, builder-pattern]
---

Three issues this session — vocabulary registry gaps, a Builder migration, and three
factual errors in personality-frameworks.md. Small, all overdue.

Writing the tests for the registry was where it got interesting. Specifically the test
that proves `allTerms()` returns only primary constants, not aliases.

The registry stores two internal maps: one keyed by all values and aliases (for lookup),
one in declaration order (for `allTerms()`). A `SourceTerm` enum with `ALPHA` (aliases:
`"a"`, `"one"`) and `BETA` (alias: `"b"`) produces five lookup entries but two ordered
entries. The assertion I wanted:

```java
assertThat(terms).doesNotContainAnyElementsOf(List.of("a", "one", "b"));
```

`terms` is `List<? extends VocabularyTerm>`. That assertion is always true. AssertJ uses
`equals()` internally. A `VocabularyTerm` enum constant never equals a `String`. No
compile error, no runtime error — just a test that's green regardless of what `allTerms()`
actually returns.

Claude caught it in the code quality review. The fix:

```java
assertThat(terms).extracting(VocabularyTerm::value)
    .doesNotContain("a", "one", "b");
```

`.extracting()` materialises the string values. Now it fails if aliases appear in the output.

The blank URI guard was simpler: `if (uri.isBlank())` immediately after reading
`meta.uri()`. Java annotation attributes can't be null at runtime — `@VocabularyMetadata(uri = null)`
is a compile error — so the right check is `isBlank()`, not a null test. I updated the SPI
Javadoc to document all four `IllegalArgumentException` paths while touching it, since any
custom registry implementation would need to enforce the same contract.

The Builder migration was about a hundred call sites across nine modules. Mechanical, with
one decision worth noting. `AgentDescriptorMapper.toRecord()` uses the 17-parameter
positional constructor and should stay that way. A Builder call means any new
`AgentDescriptor` field silently defaults to null. The positional constructor fails to
compile instead — which is exactly right for a full-fidelity JPA-to-record mapping where
every field must be explicitly sourced from the entity. It's the only production call site
with that property, and it's deliberate.

Builder calls are better at the edges where you construct from partial data. Positional
constructors are better where the compiler should tell you something changed.

The docs: SFIA doesn't provide occupation codes (that's O*NET — SFIA provides skill codes
and responsibility levels), the KAI inventory scores on a 32–160 range rather than an
unspecified low-to-high scale, and §6 of personality-frameworks.md listed ratings that
weren't in the table.
