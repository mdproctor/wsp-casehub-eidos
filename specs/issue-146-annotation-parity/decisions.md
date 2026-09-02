# Design Decisions — annotation parity (#146)

## D1: Full parity — no fields excluded from annotation surface

**Choice:** Every field on `AgentDescriptor`, `AgentDisposition`, and `AgentCapability` that is expressible via Builder or YAML will also be expressible via annotations. No progressive-disclosure carve-outs.
**Alternatives:**
- Exclude "power fields" (epistemicDomains, axisVocabularies, templates) — leaves annotation surface incomplete, forces generators to special-case surface selection
**Rationale:** A generator should be able to target any of the three surfaces with equal fidelity. Humans writing complex agents will prefer Builder/YAML naturally — that's progressive disclosure at the surface level, not at the field level.
**Trade-offs:** Verbose annotations for complex agents. Acceptable — the builder/YAML paths exist for that case.
**Sources:** casehubio/eidos#146 issue body ("full parity means any descriptor expressible in YAML or the Builder DSL can also be expressed declaratively via annotations")
**Exploration:** quick
**Status:** captured

## D2: Capability metadata via new @AgentCapabilities container, not @Discoverable expansion

**Choice:** New `@AgentCapabilities(@AgentCapabilityDef[])` in `casehub-eidos-annotations`. `@Discoverable` stays name-only in `casehub-eidos-api`. Both present on the same class → build-time error.
**Alternatives:**
- Expand `@Discoverable` with nested annotation array — breaks existing consumers, bloats the API module with rich metadata types
- Inline all fields into `@Discoverable` — changes its character from pure marker to complex annotation
**Rationale:** Module layering (`@Discoverable` is Tier 1 API, rich metadata belongs in the annotations extension). Follows established container+def pattern (`@AgentGoals/@AgentGoalDef`, `@OrgRelationships/@OrgRelationshipDef`). Progressive disclosure preserved: simple agents use `@Discoverable`, rich agents use `@AgentCapabilities`.
**Trade-offs:** Two annotations for capabilities. Mitigated by build-time mutual-exclusion check — no ambiguity.
**Sources:** AgentCapability.java (11 fields), Discoverable.java (String[] only), OrgRelationships/OrgRelationshipDef pattern, issue-139 spec Known Limitations §2
**Exploration:** quick
**Status:** captured

## D3: Epistemic domains via nested @EpistemicDomain annotation, not string encoding

**Choice:** `@EpistemicDomain(value = "legal", score = 0.95)` nested inside `@AgentCapabilityDef.epistemicDomains()`. Type-safe double for score.
**Alternatives:**
- String encoding `"legal:0.95"` — concise but requires parsing, no compile-time score validation, no IDE autocompletion
**Rationale:** Type safety. Build extension catches malformed values at compile time. Consistent with the annotation style of using typed fields (enums for priority/severity, boolean for delegation). Epistemic domains are typically 2-4 entries so verbosity is minimal.
**Trade-offs:** Slightly more verbose than string encoding. Worth it for type safety.
**Depends on:** D2 (@AgentCapabilities container)
**Sources:** AgentCapability.java:40 (epistemicDomains Map<String, Double>), AgentGoalDef pattern (typed fields)
**Exploration:** quick
**Status:** captured

## D4: Weighted disposition profiles — dual fields, not replacement

**Choice:** Keep existing `String[] dispositionProfile` for equal-weight case. Add `DispositionWeight[] weightedDispositionProfile` for weighted case. Both present → build-time error. Same pattern for `styleProfile` / `weightedStyleProfile`.
**Alternatives:**
- Replace `String[]` with nested `@DispositionWeight[]` only — breaks backward compatibility, makes simple agents verbose
- Single field that accepts both formats — annotation type system doesn't support union types
**Rationale:** Backward compatible. Simple agents keep `dispositionProfile = {"collaborative", "analytical"}`. Weighted agents use `weightedDispositionProfile = {@DispositionWeight(...)}`. Mutual exclusion eliminates ambiguity.
**Trade-offs:** Two fields for the same concept. Mitigated by build-time mutual-exclusion check and clear naming convention (prefixed with `weighted`).
**Sources:** Disposition.java:17-18 (existing String[] fields), DispositionValue.java (term + weight record), D2 mutual-exclusion precedent
**Exploration:** quick
**Status:** captured

## D5: mbtiType/enneagramType derivation — recorder with VocabularyRegistry injection

**Choice:** Inject `VocabularyRegistry` into the recorder's runtime supplier via `SyntheticBeanBuildItem.createWith()`. Recorder performs derivation at runtime, mirroring `DispositionDeserializer`'s logic. Falls back to warning when vocab module not on classpath.
**Alternatives:**
- Defer to `DescriptorCollector` — would add personality-type-convenience awareness to the registration pipeline, which currently knows nothing about it; violates separation of concerns
**Rationale:** Derivation logic already exists in `DispositionDeserializer` (YAML surface). Recorder should produce a fully formed descriptor, not one requiring post-processing. `VocabularyRegistry` is `@ApplicationScoped` — available for injection into synthetic beans.
**Trade-offs:** Recorder becomes CDI-aware (moves from `Supplier` to `BeanCreator` pattern). Minor complexity increase, but the Quarkus synthetic bean API supports this natively.
**Sources:** DispositionDeserializer.java:51-95 (mbtiType/enneagramType derivation), EidosAnnotationsRecorder.java:25 (current Supplier pattern), Quarkus SyntheticBeanBuildItem.createWith() API
**Exploration:** quick
**Status:** captured

## D6: Template refs via nested @TemplateArg annotation, not string encoding

**Choice:** `@AgentTemplates(@AgentTemplateRef[])` container. Each ref has `String id()` and `@TemplateArg[] args()`. `@TemplateArg` has `String key()` and `String value()`.
**Alternatives:**
- String encoding `"key=value"` — requires parsing, values containing `=` need escaping, no IDE support
**Rationale:** Type-safe, consistent with D3 (nested annotations for map-like data). Template args are typically 1-3 entries. Build extension validates key/value directly.
**Trade-offs:** Three annotation types for templates (`@AgentTemplates`, `@AgentTemplateRef`, `@TemplateArg`). Acceptable — each has a clear single purpose.
**Sources:** TemplateRef.java (templateId + Map<String, String> args), DescriptorTemplate.java (template definition), D3 precedent
**Exploration:** quick
**Status:** captured

## D7: axisVocabularies — dedicated per-axis fields on @Disposition, not nested annotation

**Choice:** Five new `String` fields on `@Disposition`: `socialOrientVocabulary`, `ruleFollowingVocabulary`, `riskAppetiteVocabulary`, `autonomyVocabulary`, `conflictModeVocabulary`. Empty string = not set.
**Alternatives:**
- Nested `@AxisVocabulary(axis, uri)[]` array — unnecessary indirection for a fixed 5-element enum
**Rationale:** Axes are a closed enum — they won't grow. Dedicated fields are more readable, IDE-autocompletable, and parallel the existing axis value fields (`socialOrient`, `ruleFollowing`, etc.). Naming convention mirrors the relationship: `socialOrient` (value) / `socialOrientVocabulary` (vocabulary URI).
**Trade-offs:** Five fields instead of one array. Acceptable — fixed cardinality, cleaner than nested annotation.
**Sources:** DispositionAxis.java (5 values, closed enum), AgentDescriptor.java:19 (axisVocabularies Map), Identity.java vocabulary field pattern
**Exploration:** quick
**Status:** captured
