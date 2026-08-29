# Decision Review — Light Pass

**Reviewer:** fork (single-pass adversarial)
**Date:** 2026-08-29

## D1: Generate builders — No issues

Rationale holds. Builders are mechanical. The only nuance: `AgentDisposition.Builder`
has convenience overloads (`socialOrient(String)` creates a single-element
`List<DispositionValue>` via `DispositionValue.of()`). The generator must handle
these overloads — but D1 already acknowledges this in Trade-offs. No contradiction.

## D2: Eliminate DescriptorConfig — One material concern

**Concern: record immutability vs engine's mutable POJOs.**

Engine's `CaseDefinition` is a mutable class with setters — Jackson can construct it
via no-arg constructor + setters, which is the simplest deserialization path.

Eidos's `AgentDescriptor` is a Java record. Jackson can deserialize into records
via `@JsonCreator` on the canonical constructor, BUT:

1. The compact constructor runs validation (capability uniqueness, goal-capability
   cross-validation, size limits). If YAML has invalid data, the error will come
   from the compact constructor — not from a validation phase the deserializer
   controls. This is actually **fine** — fail-fast is good. But it means the
   deserializer can't partially construct and validate later.

2. `AgentDisposition` axes are `List<DispositionValue>` in the record, but YAML
   users write `socialOrient: "collaborative"` (a plain string). The custom
   deserializer must convert `String → List.of(DispositionValue.of(s))` before
   passing to the record constructor. This **requires a custom deserializer for
   AgentDisposition** — Jackson can't do this type adaptation with mixins alone.

3. `tenancyId` is required on `AgentDescriptor` but absent from YAML (resolved
   from config at runtime). The deserializer must inject it.

**Assessment:** D2 is viable, but the rationale understates the complexity. The
custom deserializer for eidos will be more involved than engine's because of
type adaptations (String → List<DispositionValue>) and injected fields (tenancyId).
This doesn't change the decision — DescriptorConfig should still be eliminated —
but the spec should explicitly address these three points.

**Status:** No revision needed. Flag for spec.

## D3: Shared schema generator in platform — No issues

Rationale holds. D3 now confirmed as a separate platform module (spec filed).

## D4: Generate AnnotatedAgentConfig — One minor concern

**Concern: D4 says "Depends on: D3 (shared generator provides the reflection
infrastructure)" but AnnotatedAgentConfig generation is a different kind of
codegen than JSON Schema generation.** JSON Schema generation (victools) reflects
on Java types to produce a schema document. AnnotatedAgentConfig generation
reflects on Java types to produce a Java source file (a POJO). These are
different tools. victools doesn't generate Java source.

**Assessment:** The dependency link is wrong. D4 needs its own reflection-based
source generator (or an annotation processor), not victools. Remove the D3
dependency — D4 is independently implementable.

**Status:** Revise dependency link.

## D5: Convenience fields in Jackson Module — No issues

Rationale holds. Keeps api/ pure. CDI injection into the Jackson Module for
VocabularyRegistry access is standard Quarkus practice.

## D6: Publish JSON Schema as classpath resource — Minor note

**Note:** D6 says "Adds victools as a build dependency of eidos runtime." This
is inaccurate — victools is a build-time/test-time dependency only. The generated
schema file is packaged as a resource; victools itself is not a runtime dependency.

**Status:** Clarify in spec (build-time only, not runtime dependency).

## Cross-decision consistency

No contradictions found. D2 and D5 are complementary (Jackson Module handles
both the DescriptorConfig elimination and convenience field resolution). D1
(builder generation) doesn't conflict with D2 (the custom deserializer will
use the builder to construct records).

## Summary

| Decision | Verdict | Action |
|---|---|---|
| D1 | Clean | None |
| D2 | Viable, complexity understated | Flag 3 points for spec |
| D3 | Clean | None |
| D4 | Wrong dependency link | Remove D3 dependency |
| D5 | Clean | None |
| D6 | Minor inaccuracy | Clarify build-time only |
