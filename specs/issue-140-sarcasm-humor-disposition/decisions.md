# Design Decisions — #140 Structured Sarcasm/Humor Dimensions

## D1: Use case scope

**Choice:** Both generation-side (agent's own humor tone) and reception-side (understanding sarcasm directed at it)
**Alternatives:**
- Generation only — simpler, but misses comprehension awareness
- Reception only — doesn't shape agent personality output
**Rationale:** Autonomous characters need both to feel alive — produce humor in their style and comprehend sarcasm from others
**Trade-offs:** Wider scope means more surface area to implement and test
**Exploration:** quick
**Status:** captured

## D2: Model shape

**Choice:** Standalone vocabulary in casehub-eidos-vocab (no new DispositionAxis)
**Alternatives:**
- New DispositionAxis (6th axis) — full integration but heavy ripple through every axisExactMatch switch, AgentDisposition record, JPA schema, eval harness, and renderer
- Hybrid (vocab + lightweight float field on AgentDisposition) — partial integration, mixed concerns
**Rationale:** Humor/sarcasm is a communication style overlay, not a behavioral axis. It doesn't determine how an agent behaves — it determines how an agent communicates, independent of disposition. A new `styleProfile` field on AgentDisposition (with `styleVocabulary` on AgentDescriptor) provides a clean second profile slot alongside the behavioral `dispositionProfile`. Pre-release project — no migration cost.
**Trade-offs:** Two new fields on core API records (AgentDescriptor, AgentDisposition). Builder methods, JPA column, YAML support needed. Acceptable for a pre-release project with no deployed instances.
**Revised:** Spec review R1-01/R1-02/R1-03 — original "standalone vocab with axis mappings following DISC pattern" was a category error. Sarcasm style doesn't make meaningful behavioral axis claims. `styleProfile`/`styleVocabulary` is the correct abstraction.
**Exploration:** quick
**Status:** captured

## D3: Term source

**Choice:** Sarc7-grounded — 7 sarcasm types from the Sarc7 paper (self-deprecating, brooding, deadpan, polite, obnoxious, raging, manic)
**Alternatives:**
- Agent-oriented humor styles (dry wit, absurdist, etc.) — broader but no academic backing
- Layered (Sarc7 types as specializes() children of broader terms) — best of both but more terms to maintain
**Rationale:** Academic grounding from a published taxonomy. Avoids mixing systems (invented terms alongside research-backed ones), which risks being unsound.
**Trade-offs:** Narrow to sarcasm only — doesn't cover non-sarcastic humor (warmth, wit). A separate vocabulary could cover that later.
**Exploration:** quick
**Status:** captured

## D4: Reception-side modeling

**Choice:** Standalone AgentCapability `"sarcasm-awareness"` — independently queryable via find(), participates in subsumption matching, has own health probing
**Alternatives:**
- Epistemic domain on existing capability — lightweight but semantic mismatch (epistemicDomains is about knowledge domains like `{"java": 0.95}`, not processing capabilities)
- Defer reception entirely — simpler but leaves the "both" scope incomplete
**Rationale:** Sarcasm detection is a processing capability, not a knowledge domain. A standalone capability is independently discoverable via `AgentRegistry.find(capabilityName="sarcasm-awareness")`, can have its own epistemicDomains (e.g., cultural sarcasm detection confidence), and integrates with CapabilityHealth for independent probing.
**Trade-offs:** Heavier — requires a capability declaration on each agent that claims sarcasm awareness. Can be grounded in CasehubCapabilityTerm for subsumption.
**Exploration:** quick
**Revised:** R1-03 — epistemicDomains is about knowledge domains, not processing capabilities
**Status:** revised

## D5: Cross-vocabulary mappings

**Choice:** Sarc7Term implements axisExactMatch for reference and consistency checking. Cross-mappings exist on the enum and in personality-frameworks.md. DescriptorCollector does NOT auto-derive axes from styleProfile — the mappings are metadata, not axis population drivers.
**Alternatives:**
- Full axis derivation (DISC pattern) — category error; sarcasm type doesn't determine behavioral disposition
- No cross-maps at all — loses consistency validation opportunity
**Rationale:** Cross-mappings document genuine (if weak) correlations between sarcasm style and behavioral tendencies. Available for consumers who want consistency checking. But NOT used for axis auto-derivation because the correlations are editorial, not empirically grounded.
**Trade-offs:** Mappings exist in code and docs but don't drive axis values. Consumers who call axisExactMatch directly get the mappings; auto-derivation ignores them.
**Revised:** Spec review R1-01 — axis population from sarcasm type is a category error. Mappings preserved as reference, not as derivation drivers.
**Exploration:** quick
**Status:** captured

## D6: Prompt guidance methods

**Choice:** Include responseStyleGuidance() and antiPatternWarning() on each Sarc7 term
**Alternatives:**
- Terms only, no guidance — lighter but misses the generation-side value that motivated the work
**Rationale:** Directly shapes agent output tone in the system prompt. Note: the current render pipeline gates responseStyleGuidance/antiPatternWarning on `JUNGIAN_VOCAB_URI` — the pipeline must be generalized to support multiple vocabulary URIs, or a dedicated humor rendering section must be added for Sarc7 guidance to reach the prompt.
**Trade-offs:** Guidance text must be carefully written — bad prompt guidance could make LLM output worse than no guidance. Requires render pipeline changes (not just vocab-only work).
**Exploration:** quick
**Status:** captured

## D7: Vocabulary structure — dimensions as read-only enum fields

**Choice:** Sarc7Term with 7 types, each carrying the 4 Sarc7 evaluation dimensions (incongruity, shockValue, contextDependency, emotionalTone) as read-only fields with paper-derived values. No per-agent overrides. DispositionValue weight handles per-agent differentiation (DEADPAN at 0.85 vs 0.4). Experimentation with dimension values happens in the eval harness.
**Alternatives:**
- Per-agent dimension overrides via override map on descriptor — breaks vocabulary abstraction (no existing vocab has per-agent overrides), introduces undefined mechanism, carries experimental knobs into production
- Dimensions only on descriptor, enum is just a type tag — simpler enum but no sensible defaults
**Rationale:** The 4 dimensions characterize what each sarcasm type IS — deadpan IS high-incongruity, low-shock. They're intrinsic properties, not tuning knobs. DispositionValue weight already provides per-agent customization. The eval harness is the right place to experiment with different values; winners get baked into the enum constants.
**Trade-offs:** Can't tune dimensions per-agent in production. Must update enum constants and redeploy to change dimension values.
**Depends on:** D3 (Sarc7 terms), D6 (prompt guidance)
**Exploration:** quick
**Revised:** R1-04 — per-agent overrides break vocabulary abstraction; DispositionValue weight is the existing customization path
**Status:** revised

## D8: Cross-vocabulary mapping approach

**Choice:** Map each Sarc7 term to ConscientiousnessTerm (axes 1-4) and ThomasKilmannTerm (CONFLICT_MODE) via axisExactMatch, following the DISC/BigFive pattern. Document mappings in docs/personality-frameworks.md as the authoritative reference.
**Alternatives:**
- No cross-maps — simpler but no subsumption matching or consistency validation
- Minimal maps (strong correlations only) — safer but incomplete
**Rationale:** Follows established vocabulary pattern. personality-frameworks.md is the authoritative source for all framework mappings — Sarc7 should be documented there alongside DISC, BigFive, Belbin, and Jungian.
**Trade-offs:** Sarc7-to-Conscientiousness mappings are editorial (no psychometric research), unlike BigFive-to-Conscientiousness (Costa & McCrae, 1992). Must be reviewed carefully. personality-frameworks.md update is part of the deliverable.
**Depends on:** D3 (Sarc7 terms), D2 (standalone vocab)
**Exploration:** quick
**Status:** captured

## D9: Render pipeline architecture

**Choice:** Hybrid three-layer design. Layer 1: framework-specific structural rendering via dedicated private methods (assembleJungianCognitiveProfile, assembleSarc7HumorProfile). Layer 2: shared guidance helper (renderGuidanceBlock) that renders responseStyleGuidance/antiPatternWarning from any VocabularyTerm with parameterized section heading. Layer 3: generic fallback sweep (assembleGenericVocabularyGuidance) for future vocabularies that provide guidance but have no dedicated renderer.
**Alternatives:**
- Pure generalization (Option A) — abstracts Jungian-specific logic into a generic framework, weakening Jungian rendering quality
- Pure separation (Option B) — each vocabulary gets fully independent method, duplicating the guidance rendering mechanism
- SPI-based (VocabularyPromptContributor CDI beans) — fully open-closed but heavy engineering for 2 vocabularies
**Rationale:** Preserves full fidelity of framework-specific rendering (Jungian stays Jungian, Sarc7 stays Sarc7). Factors out the generic part (guidance text rendering) exactly once. Generic fallback means future guidance-only vocabularies need zero pipeline changes. Clear migration path to SPI if we reach 5+ custom renderers.
**Trade-offs:** Adding a new vocabulary with custom rendering still requires one private method + one if-check. Acceptable at 2-3 vocabularies; refactor to registry/SPI if it grows beyond that.
**Depends on:** D6 (prompt guidance), D2 (standalone vocab)
**Exploration:** deep-analysis
**Status:** captured
