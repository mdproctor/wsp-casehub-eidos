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
**Rationale:** Humor/sarcasm is a personality framework (like Jungian, MBTI, Enneagram), not a behavioral axis (like SOCIAL_ORIENTATION or RISK_APPETITE). Follows established vocab pattern. No schema migrations needed. Note: the render pipeline requires changes to surface Sarc7 responseStyleGuidance (currently gated on Jungian URI — see D6).
**Trade-offs:** No first-class axis means humor doesn't participate in disposition axis resolution directly — only through cross-vocabulary mappings. Render pipeline changes needed (not "zero changes" — D6 requires pipeline generalization).
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

**Choice:** Cross-map each Sarc7 term to existing vocabularies (ConscientiousnessTerm, ThomasKilmannTerm) via axisExactMatch
**Alternatives:**
- Self-contained, no cross-maps — simpler but no integration with disposition axis resolution or subsumption matching
- Minimal cross-maps (only strong correlations) — safer but loses weaker but useful correlations
**Rationale:** Follows DISC/BigFive pattern. Enables subsumption matching and consistency validation (e.g., a DISC-S agent with obnoxious sarcasm would flag as inconsistent).
**Trade-offs:** Cross-mappings are subjective — "brooding maps to CONSERVATIVE on RISK_APPETITE" is defensible but debatable. Must be reviewed carefully.
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
