# Spec Review — Light Pass

**Reviewer:** fork (single-pass)
**Date:** 2026-08-29

## 1. Gaps

### Gap 1: `tenancyId` is already in YAML — spec says it isn't

The spec says:
> "tenancyId injection — not in YAML, resolved from MicroProfile Config at runtime"

But `DescriptorConfig` line 187 declares `public String tenancyId` — meaning tenancyId IS
already a YAML field. The current `toDescriptor()` line 39 passes `cfg.tenancyId` directly
to the builder. The spec's claim that tenancyId is "absent from YAML" is wrong.

**Fix:** The deserializer should read `tenancyId` from YAML if present (as today), and fall
back to MicroProfile Config only when absent. The spec partially acknowledges this in
"reads an optional tenancyId field from YAML" but the opening statement contradicts it.
Clean up the contradiction.

### Gap 2: `axisVocabularies` mapping not addressed

The current `toDescriptor()` handles `axisVocabularies` (lines 51-55) — converting
`Map<String, String>` to `Map<DispositionAxis, String>` via `DispositionAxis.valueOf(key)`.
This mapping is on `AgentDescriptor`, not `AgentDisposition`, so it belongs in
`AgentDescriptorDeserializer`, not `DispositionDeserializer`. The spec doesn't mention it.

**Fix:** Add `axisVocabularies` handling to the `AgentDescriptorDeserializer` design section.

### Gap 3: `loadFrom(InputStream)` and `loadFrom(InputStream, VocabularyRegistry)` API

The current registrar exposes two `loadFrom` overloads used by tests (and potentially by
other callers) to parse YAML without CDI. The spec doesn't address whether these survive.
If `ClasspathYamlDescriptorRegistrar` now depends on CDI-injected `EidosDescriptorModule`,
the test-friendly `loadFrom(InputStream)` path needs a non-CDI ObjectMapper factory.

**Fix:** Document that a static factory method (or test-only constructor) creates an
`ObjectMapper` with the module configured without CDI — passing `null` VocabularyRegistry
(matching current `loadFrom(yaml, null)` behaviour). This is important for test isolation.

### Gap 4: `DescriptorFile` as a record won't deserialize with Jackson YAML by default

The spec proposes `record DescriptorFile(List<AgentDescriptor> descriptors) {}`. Jackson
requires either `@JsonCreator` or parameter-names module for record deserialization. The
current `DescriptorFile` is a POJO with public fields. Switching to a record without
addressing this will cause deserialization failures.

**Fix:** Either keep `DescriptorFile` as a POJO (trivial, no drift risk) or add
`jackson-module-parameter-names` / `@JsonProperty` annotations. The simplest path:
keep it as a static inner class with a public field, matching current behaviour.

## 2. Test Strategy Coverage

Test strategy is solid for the new deserializer. Two gaps:

- **Missing:** Test for `axisVocabularies` deserialization (Map<String, String> → Map<DispositionAxis, String>)
- **Missing:** Test for `loadFrom(InputStream)` without CDI context (null VocabularyRegistry)

## 3. Risks — Additions

### Risk 4: Static ObjectMapper is now CDI-dependent

Current code uses a `static final ObjectMapper` (line 34). The new design injects the module
via CDI. This changes the ObjectMapper lifecycle — it's no longer a static singleton. If any
code path caches or shares the ObjectMapper outside CDI scope, it won't have the module
registered.

### Risk 5: `enneagramType` logic is non-trivial

The enneagram → axis resolution (lines 75-99) involves cross-vocabulary equivalentValues
lookups across 5 axes plus a special case for CONFLICT_MODE (Thomas-Kilmann). The spec says
"port existing logic" — this is correct, but the complexity should be flagged. The
`resolveEnneagramAxes()` method will be ~25 lines of axis-specific logic.

## 4. Scope Clarity

Scope is clear. An implementer knows:
- **Build:** EidosDescriptorModule, AgentDescriptorDeserializer, DispositionDeserializer
- **Delete:** 7 inner classes + toDescriptor()
- **Simplify:** ClasspathYamlDescriptorRegistrar
- **Defer:** D1, D3, D4, D6

## Summary

| Finding | Severity | Action |
|---|---|---|
| tenancyId contradiction | Minor | Fix spec wording |
| axisVocabularies missing | Gap | Add to spec |
| loadFrom test API | Gap | Document non-CDI path |
| DescriptorFile record | Gap | Keep as POJO or add @JsonCreator |
| Static ObjectMapper lifecycle | Risk | Add Risk 4 |
| enneagramType complexity | Note | Flag in spec |
