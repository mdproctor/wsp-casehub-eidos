# Design Decisions — annotation parity (#146)

## D1: Full parity — no fields excluded from annotation surface

**Choice:** Every field on `AgentDescriptor`, `AgentDisposition`, and `AgentCapability` that is expressible via Builder or YAML will also be expressible via annotations. No progressive-disclosure carve-outs.
**Alternatives:**
- Exclude "power fields" (epistemicDomains, axisVocabularies, templates) — leaves annotation surface incomplete, forces generators to special-case surface selection
**Rationale:** Issue-139 Known Limitations §1-3 excluded weighted profiles, rich capabilities, and templates as an intentional scope boundary for initial delivery — not as a permanent architectural constraint. Issue #146 reopens these gaps. Annotations are the natural declarative target for generator-produced agent definitions: they produce compilable, type-checked, IDE-navigable, version-controllable code that humans subsequently maintain. Builder code is imperative; YAML lacks type safety. Progressive disclosure is preserved at the surface level (simple agents use `@Discoverable` + `@Identity`; complex agents add `@AgentCapabilityDef`, `@AgentTemplateRef`, etc.), not at the field level.
**Trade-offs:** Verbose annotations for complex agents. Acceptable — the builder/YAML paths exist for that case, and formatted annotations (one attribute per line) are no more verbose than the equivalent builder calls.
**Sources:** casehubio/eidos#146 issue body ("full parity means any descriptor expressible in YAML or the Builder DSL can also be expressed declaratively via annotations"), issue-139 Known Limitations §1-3 (scope boundaries, not architectural constraints)
**Exploration:** quick
**Status:** captured

## D2: Capability metadata via @Repeatable @AgentCapabilityDef with merge semantics

**Choice:** New `@AgentCapabilityDef` with `@Repeatable(AgentCapabilities.class)` and `@Target(ElementType.TYPE)` in `casehub-eidos-annotations`. `@Discoverable` stays name-only in `casehub-eidos-api`. Both present on the same class → merge semantics: name-only capabilities from `@Discoverable` and rich capabilities from `@AgentCapabilityDef` form a union. Name collision between the two → build-time error.
**Alternatives:**
- Expand `@Discoverable` with nested annotation array — breaks module layering: `@Discoverable` is Tier 1 API, rich metadata types belong in the annotations extension
- Inline all fields into `@Discoverable` — changes its character from pure marker to complex annotation
- Mutual exclusion (original D2) — forces all-or-nothing rewrite when one capability needs rich metadata; unnecessary verbosity for mixed-granularity agents
**Rationale:** Module layering preserved (`@Discoverable` in `casehub-eidos-api`, `@AgentCapabilityDef` in `casehub-eidos-annotations`). Merge semantics avoids forced migration: agents can start with `@Discoverable`, add `@AgentCapabilityDef` for rich capabilities incrementally, and remove the corresponding name from `@Discoverable`. `@Repeatable` follows the modern platform idiom (`@Supervises` in org-annotations already uses it). `@AgentGoalDef`/`@AgentConstraintDef` use the older `@Target({})` pattern — retrofitting them is a separate concern.
**Trade-offs:** Two annotation types for capabilities. Processor must merge from two sources — straightforward via Jandex `getAnnotations()`.
**Sources:** AgentCapability.java (11 fields), Discoverable.java (String[] only), Supervises.java (@Repeatable pattern), issue-139 spec Known Limitations §2
**Exploration:** quick
**Status:** revised — R1-03 (merge semantics), R1-04 (@Repeatable)

## D3: Epistemic domains via nested @EpistemicDomain annotation, not string encoding

**Choice:** `@EpistemicDomain(value = "legal", score = 0.95)` nested inside `@AgentCapabilityDef.epistemicDomains()`. Type-safe double for score.
**Alternatives:**
- String encoding `"legal:0.95"` — concise but requires parsing, no compile-time score validation, no IDE autocompletion
**Rationale:** Type safety. Build extension catches malformed values at compile time. Consistent with the annotation style of using typed fields (enums for priority/severity, boolean for delegation). Epistemic domains are typically 2-4 entries so verbosity is minimal.
**Trade-offs:** Slightly more verbose than string encoding. Worth it for type safety.
**Depends on:** D2 (@AgentCapabilityDef)
**Sources:** AgentCapability.java:40 (epistemicDomains Map<String, Double>), AgentGoalDef pattern (typed fields)
**Exploration:** quick
**Status:** captured

## D4: Weighted disposition profiles — single DispositionWeight[] field

**Choice:** Replace `String[] dispositionProfile` with `DispositionWeight[] dispositionProfile` on `@Disposition`. `@DispositionWeight` has `String value()` (term) and `double weight() default 1.0`. Same for `styleProfile`. No dual fields, no mutual-exclusion check.
**Alternatives:**
- Dual fields (original D4) — backward-compatible but contradicts D1's full-parity principle and creates a progressive-disclosure carve-out
- Union types — annotation type system doesn't support this
**Rationale:** Consistent with D1: no progressive-disclosure carve-outs. Simple agents use `@DispositionWeight("collaborative")` (weight defaults to 1.0); weighted agents use `@DispositionWeight(value = "analytical", weight = 0.3)`. Single field, no ambiguity, no ad-hoc naming prefix convention. Migration from `dispositionProfile = {"collaborative"}` to `dispositionProfile = {@DispositionWeight("collaborative")}` is mechanical.
**Trade-offs:** Breaking change for existing `@Disposition` users (must update `String[]` → `DispositionWeight[]`). Per design philosophy: migration is mechanical and the breakage forces every user to be explicit about weight.
**Sources:** DispositionValue.java (term + weight), Disposition.java (existing String[] fields)
**Exploration:** quick
**Status:** revised — R1-05 (internal consistency with D1)

## D5: mbtiType/enneagramType derivation — shared PersonalityTypeDeriver utility

**Choice:** Extract a `PersonalityTypeDeriver` utility that centralizes mbtiType/enneagramType → disposition derivation logic. Both `DispositionDeserializer` (YAML surface) and the annotation recorder call this utility. `VocabularyRegistry` injection into the recorder uses `SyntheticBeanBuildItem.createWith()` as before.
**Alternatives:**
- Duplicate derivation logic in recorder (original D5) — maintenance hazard when vocabulary mapping rules change
- Defer to `DescriptorCollector` — adds personality-type-convenience awareness to the registration pipeline, which currently knows nothing about it; violates separation of concerns
**Rationale:** The derivation logic (~40 lines in `DispositionDeserializer:51-95`) includes mbtiType → vocab resolution → defaultProfile, and enneagramType → per-axis equivalentValues resolution with precedence rules (mbtiType only when dispositionProfile is empty; enneagramType only when explicit axis value is empty). Duplicating this creates lockstep maintenance. A shared utility eliminates the duplication without changing the injection architecture — the recorder still uses `createWith()` for `VocabularyRegistry` access.
**Trade-offs:** New utility class. Minor — worth it for DRY principle on non-trivial derivation logic.
**Sources:** DispositionDeserializer.java:51-95 (mbtiType/enneagramType derivation), EidosAnnotationsRecorder.java:25 (current Supplier pattern), Quarkus SyntheticBeanBuildItem.createWith() API
**Exploration:** quick
**Status:** revised — R1-06 (centralized derivation)

## D6: Template refs via @Repeatable @AgentTemplateRef, not string encoding

**Choice:** `@AgentTemplateRef` with `@Repeatable(AgentTemplates.class)` and `@Target(ElementType.TYPE)`. Each ref has `String id()` and `@TemplateArg[] args()`. Two annotation types on the class surface: `@AgentTemplateRef` (repeatable) and `@TemplateArg` (nested within refs). `@AgentTemplates` is the implicit container.
**Alternatives:**
- String encoding `"key=value"` — requires parsing, values containing `=` need escaping, no IDE support
- Three explicit annotation types (original D6) — unnecessarily deep user-facing nesting
- Omit templates from annotation surface — leaves gap that contradicts D1 parity principle
**Rationale:** `@Repeatable` eliminates the need for explicit `@AgentTemplates` container wrapping — users place `@AgentTemplateRef` directly on the class. Two user-facing annotation types instead of three. Type-safe, consistent with D2 (`@AgentCapabilityDef` also uses `@Repeatable`). Build extension validates template refs and args against `TemplateRegistry`.
**Trade-offs:** `@TemplateArg` nesting is the deepest in eidos annotations (`@AgentTemplateRef.args.@TemplateArg`). Acceptable — template args are typically 1-3 entries.
**Sources:** TemplateRef.java (templateId + Map<String, String> args), Supervises.java (@Repeatable pattern), D2 precedent
**Exploration:** quick
**Status:** revised — R1-11 (@Repeatable reduces annotation count)

## D7: axisVocabularies — nested @AxisVocabulary annotation, not per-axis named fields

**Choice:** `@AxisVocabulary(axis = DispositionAxis.X, uri = "urn:...")[]` field on `@Disposition`. Extensible for future axis additions without annotation changes.
**Alternatives:**
- Five named `String` fields (original D7: `socialOrientVocabulary`, etc.) — readable and IDE-autocompletable, but assumes axes won't grow; adds an annotation field for every new axis
**Rationale:** `DispositionAxis` is not architecturally frozen. ARC42STORIES §1 describes "four open-string disposition axes" but the enum has five (CONFLICT_MODE was added or the description was always imprecise). Issue #140 D2 explicitly evaluated a 6th axis — rejected on domain grounds ("humor is style, not behavior"), not structural grounds. Active personality framework research (issue-107 Jungian, issue-133 personality-aware rendering) continues to explore new behavioral dimensions. The nested annotation makes the annotation surface automatically extensible when a new axis is added — no annotation, processor, recorder, or config changes needed. The asymmetry with named axis-value fields on `@Disposition` is a pre-existing design constraint (pre-#146), not introduced by this decision.
**Trade-offs:** Less IDE-discoverable than named fields for the 5 existing axes. Acceptable — axis vocabularies are power-user configuration; most agents won't set per-axis vocabularies at all.
**Sources:** DispositionAxis.java (5 values), ARC42STORIES §1 ("four" vs 5 actual), issue-140 D2 (6th axis evaluation), AgentDescriptor.java:19 (axisVocabularies Map)
**Exploration:** quick
**Status:** revised — R1-08 (axis growth evidence)

## D8: Build-time validation for new annotation fields

**Choice:** Extend `EidosAnnotationsProcessor` build-time validation to cover: (1) `qualityHint` 0.0-1.0 range, (2) `epistemicDomains` scores 0.0-1.0 range, (3) `excludedDomains ∩ epistemicDomains.keys() = ∅`, (4) `@AgentGoalDef.capabilities()` validated against both `@Discoverable` and `@AgentCapabilityDef` capability names.
**Alternatives:**
- Runtime-only validation via compact constructors — catches errors at startup, not at build time
**Rationale:** Issue-139 D4 established hybrid build-time validation as a principle (Quarkus philosophy: fail-fast at build time). The `AgentCapability` compact constructor already validates these constraints at runtime. Extending the build extension to catch them at compile time is consistent and catches typos/misconfigurations before deployment.
**Trade-offs:** Build extension complexity increases. Acceptable — the validation logic mirrors what compact constructors already enforce.
**Depends on:** D2 (@AgentCapabilityDef)
**Sources:** EidosAnnotationsProcessor.java (existing vocabulary validation), AgentCapability.java (compact constructor validation), issue-139 D4 (hybrid validation principle)
**Exploration:** quick (implicit decision surfaced by reviewer)
**Status:** captured

## D9: weightsFingerprint and modelVersion on @Identity

**Choice:** Add `String weightsFingerprint() default ""` and `String modelVersion() default ""` to `@Identity`. Both are trivial string fields with empty defaults.
**Alternatives:**
- Omit — leaves two fields as annotation gaps, contradicting D1 full-parity
**Rationale:** Issue #146 explicitly lists `weightsFingerprint` as a gap. `modelVersion` is similarly absent from `@Identity` but present on `AgentDescriptor`. Both are simple string fields that complete the `@Identity` → `AgentDescriptor` field mapping.
**Trade-offs:** None — trivial additions.
**Sources:** AgentDescriptor.java (modelVersion, weightsFingerprint fields), Identity.java (missing both), issue #146 gaps table
**Exploration:** quick (implicit decision surfaced by reviewer)
**Status:** captured

## D10: mbtiType and enneagramType as explicit fields on @Disposition

**Choice:** Add `String mbtiType() default ""` and `String enneagramType() default ""` to `@Disposition`. Derivation follows the same precedence rules as YAML: mbtiType applies only when dispositionProfile is empty; enneagramType derives an axis only when the explicit axis value is empty. Precedence rules documented in annotation Javadoc and enforced by `PersonalityTypeDeriver` (D5).
**Alternatives:**
- Fields on @Identity — these influence disposition, not identity metadata
- Separate @PersonalityType annotation — over-annotates for two convenience fields
**Rationale:** Issue #146 lists both as gaps. These are convenience fields that auto-derive disposition profile and axis values via VocabularyRegistry. Placement on `@Disposition` is natural — they influence disposition state. Precedence rules must be explicit: the annotation Javadoc documents that explicit axis values and dispositionProfile take priority over derived values.
**Depends on:** D5 (PersonalityTypeDeriver handles derivation logic)
**Sources:** DispositionDeserializer.java:51-95 (mbtiType/enneagramType precedence rules), issue #146 gaps table
**Exploration:** quick (implicit decision surfaced by reviewer)
**Status:** captured
