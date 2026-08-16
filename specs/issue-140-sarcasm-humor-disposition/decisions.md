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
**Rationale:** Zero changes to core API. Follows established DISC/BigFive/Enneagram pattern. If vocab turns out unused, it just sits there. No schema migrations needed.
**Trade-offs:** No first-class axis means humor doesn't participate in disposition axis resolution directly — only through cross-vocabulary mappings
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

**Choice:** Epistemic domain on existing capability — add `"sarcasm-detection"` as an epistemicDomain entry on whatever NLU/comprehension capability the agent declares
**Alternatives:**
- Standalone AgentCapability `"sarcasm-awareness"` — independently discoverable via find() but heavier, possibly premature
- Defer reception entirely — simpler but leaves the "both" scope incomplete
**Rationale:** Lightweight, no new capability declaration needed. Epistemic domains already exist for qualifying capability competence by domain.
**Trade-offs:** Not independently queryable via AgentRegistry.find(capabilityName="sarcasm-awareness") — requires knowing the parent capability name
**Exploration:** quick
**Status:** captured

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
**Rationale:** Directly shapes agent output tone via the existing render pipeline (EidosRenderPipeline already reads these from JungianFunctionTerm). Each sarcasm type generates specific tone instructions in the system prompt.
**Trade-offs:** Guidance text must be carefully written — bad prompt guidance could make LLM output worse than no guidance
**Exploration:** quick
**Status:** captured

## D7: Vocabulary structure — dimensions and overridability

**Choice:** Sarc7Term with 7 types, each carrying the 4 Sarc7 evaluation dimensions (incongruity, shockValue, contextDependency, emotionalTone) as fields with paper-derived defaults. Per-agent dimension overrides via an override map on the descriptor.
**Alternatives:**
- Dimensions on enum only, no per-agent override — simpler but can't experiment with tuning
- Dimensions only on descriptor, enum is just a type tag — simpler enum but no sensible defaults
**Rationale:** Defaults from the paper give sensible out-of-the-box behavior; most agents won't override. Per-agent overrides enable experimentation ("do LLMs respond differently to incongruity=0.3 vs 0.8?") without losing the academic grounding.
**Trade-offs:** Requires a new mechanism for per-agent dimension overrides on the descriptor. Adds complexity to the descriptor → renderer pipeline.
**Depends on:** D3 (Sarc7 terms), D6 (prompt guidance)
**Exploration:** quick
**Status:** captured
