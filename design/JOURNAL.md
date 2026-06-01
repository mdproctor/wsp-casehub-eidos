# Design Journal — issue-023-real-world-profile-library

## 2026-06-01/02 — Real-world profile library + personality preservation (eidos#23)

### EvalCase sealed interface

`EvalCase` was a record. Changed to a sealed interface permitting `SyntheticEvalCase` (hand-crafted, no source profile) and `ProfiledEvalCase` (wraps `AgentProfile`, backed by a real-world YAML). The motivation: four new judges all require `AgentProfile` and `Optional<AgentProfile>` in a record component is a design smell. The sealed interface provides type safety at call sites without Optional guards.

`EvalDataset.all()` returns `List<EvalCase>` (the abstraction, not `List<SyntheticEvalCase>`) — callers don't need to know the concrete type. `RealWorldEvalDataset.all()` returns `List<ProfiledEvalCase>` directly because callers need `profile()` access.

### AgentProfileLoader (test scope only)

Placed in `eval/src/test/java/` because it reads from test-classpath YAML resources and has no production use. Stage 0 validation runs at load time: each variant pair must differ on exactly the declared `primaryAxis` and be identical on all other `AgentDisposition` fields (including `delegation`). Throws `IllegalStateException` with descriptive message on violation.

Uses `ObjectMapper(YAMLFactory).findAndRegisterModules()` — `findAndRegisterModules()` is required for Java record deserialization on Java 21+.

### Separate ProximityJudge (not a new EvalDimension)

Semantic proximity measures rendered-vs-original-prose fidelity — a different reference than descriptor-vs-rendered (what `EvalDimension` measures). Folding it into `EvalDimension` would corrupt `applicableFor(format)` (mixing format and profile-availability as axes) and make `EvalResult.overall` incomparable across synthetic and real-world cases. `ProximityJudge` is a separate `@ApplicationScoped` bean with its own result type and floor.

### Three-stage personality preservation attribution

Stage 1 (`VocabularyExpressivenessJudge`): asks the LLM "how precisely can a short phrase capture this personality axis from the prose?" — four sequential calls, one per axis (`socialOrient`, `ruleFollowing`, `riskAppetite`, `autonomy`). Scores ≤ 2 identify vocabulary gaps that feed eidos#26.

Stage 2 (`TraitExpressionJudge`): blind-scores the rendered text. User message is `rendered.content()` only — no descriptor, no profile. Direction match: HIGH ≥ 4, LOW ≤ 2, NEUTRAL always matches. Score 3 (neutral anchor) matches neither directional declaration.

Stage 3 (`PairContrastJudge`): pairwise effect size measurement. Resolves renders by `evalCase.profile().name()` (not `evalCase.name()` which has a format suffix). "A" in judge response = `pair.higher()` profile.

Attribution logic (in `PersonalityPreservationReport.build()`):
- s1 ≤ 2 → VOCABULARY_GAP
- s1 ≥ 4 AND matchRate < 0.5 → RENDERER_FLATTENING
- s1 ≥ 4 AND matchRate ≥ 0.5 AND s3 in [1,2] → PROFILE_DESIGN_GAP
- s1 ≥ 4 AND matchRate ≥ 0.5 AND s3 > 2 → WORKING
- else → INSUFFICIENT_DATA

Critical: all diagnoses beyond VOCABULARY_GAP require `s1 >= 4`. Without this guard, borderline vocabulary (s1=3) generates false RENDERER_FLATTENING or WORKING diagnoses.

### Stage 1 scope boundary

Stage 1 is valid for vocabularies with common-language semantics (Conscientiousness, SVO). Not valid for domain-specific technical vocabularies where terms carry non-standard meanings — the LLM's general understanding of words is a valid proxy for the former but not the latter.

### Variant pair single-axis isolation

Profiles in a variant pair must differ on exactly one `AgentDisposition` axis. The pairs in this library: `sw-engineer-{careful,bold}` on `riskAppetite`; `security-analyst-{defensive,proactive}` on `ruleFollowing`. All other axes (`socialOrient`, `ruleFollowing`/`riskAppetite`, `autonomy`, `delegation`) are identical within each pair.

### LLM call budget

84 calls per steady-state eval run (16 quality + 16 proximity + 32 Stage-1 + 16 Stage-2 + 4 Stage-3). Renderer runs in structural-only mode to keep the budget bounded.

### Deferred

- eidos#26: Belbin/DISC/Big Five vocabulary module (awaits eidos#29 framework mapping doc)
- eidos#27: Theoretical framework grounding in AgentDescriptor/renderer
- eidos#28: Belbin-based agent composition for project phases in casehub-engine
- eidos#29: Docs: Mapping Personality and Role Frameworks to AgentDescriptor
- eidos#30: Minor eval cleanup (attribution guard clarity, test redundancy, emoji in output)
