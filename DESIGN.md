# eidos — Design

This document records design decisions and architectural reasoning as the project evolves.
Entries are merged from branch design journals at epic close.

---

## §Phase3 — SystemPromptRenderer + AgentStateStore

**Date:** 2026-05-25

### Goals are dynamic, not a static string

The original SPI had `render(AgentDescriptor, String goal, RenderContext)`. Goals in casehub align with engine cases — an agent may be case-bound (the case is the goal, plan items are sub-goals) or generalised (no case yet). Goals evolve over time as sub-tasks are assigned.

Decision: replace `String goal` + `RenderContext` with `AgentPromptContext` — a single accumulation point carrying `Optional<GoalContext>`, `List<Resource>`, `situationalContext`, and `RenderFormat`. Re-renderable at any point as the context grows. Breaking change accepted; no callers existed.

### Rendering pipeline is two-phase

The renderer has two paths:
1. **Structural** — serializes AgentDescriptor + AgentPromptContext to single-quoted YAML, renders properties and lists. No LLM call. Valid output immediately usable by an LLM consumer.
2. **Semantic (optional)** — passes YAML to `ChatModel.chat(String)` (LangChain4j 1.14.1). The LLM generates an optimised system prompt for LLM consumption (not human prose). Returns the LLM response as content.

Prompt instruction: "Generate a system prompt optimised for LLM consumption — use whatever structure is clearest and most concise. Do not optimise for human readability." This deliberately avoids biasing toward paragraphs — the LLM chooses the right format.

Semantic pass is optional — the structural output is the valid fallback. Full semantic pipeline (canonical YAML schema, prompt templates, quality evaluation) is tracked in eidos#6.

### AgentStateStore follows persistence-backend CDI pattern

`DegradationReason` was nested inside `CapabilityHealth`. Promoted to top-level because both `AgentStateStore` and `CapabilityHealth` reference it — coupling the store to the probe SPI for no architectural benefit.

`AgentStateStore` SPI: `record(agentId, reason, expiresAt)`, `query(agentId)`, `clear(agentId)`. Operational SPI → `@DefaultBean` no-op. `InMemoryAgentStateStore @Alternative @Priority(1)` in `casehub-eidos-memory`. JPA persistence deferred.

`DefaultCapabilityHealth` probe order (fail-fast): degraded state → unavailable → epistemically weak → ready. State store check runs first — a known-degraded agent is not worth checking further.

### LangChain4j 1.x API changed from 0.x

In LangChain4j 1.14.1, the chat model interface is `ChatModel` (not `ChatLanguageModel`). The call method is `chat(String)` (default, convenient) which internally builds a `ChatRequest` and calls `doChat(ChatRequest)` — the method implementations override. `UserMessage.singleText()` (not `.text()`) is the accessor for the message content. Test mocks must implement `doChat(ChatRequest)` returning `ChatResponse.builder().aiMessage(...).build()`.

### Slot heading uses vocabulary label, body uses term description

Reviewer caught that the spec says: slot label → heading, slot description → body. Initial implementation used raw slot value for heading and term label for body. Fixed.

### GoalContext.subGoals requires compact constructor guard

Plain record constructor allows `new GoalContext("review", null, null)` — both YAML serializer and structural renderer called `subGoals.isEmpty()` without null check. Added compact constructor: `subGoals = subGoals != null ? subGoals : List.of()`.

### YAML values must be single-quoted

Raw value concatenation produces invalid YAML for values containing `:`, `#`, `|`, `>`. Single-quote scalar style handles all special characters; single quotes in values are doubled (`''`).

---

## §eidos21-20-22 — RenderFormat rename, validation surface, multi-format eval

**Date:** 2026-05-31 | **Branch:** issue-021-020-eval-coverage-validation

### RenderFormat: structure-named over provider-named

`RenderFormat` renamed from provider-named (`CLAUDE_MD`, `OPENAI_SYSTEM`, `GEMINI`) to structure-named (`MARKDOWN`, `PROSE`, `A2A_CARD`). `OPENAI_SYSTEM` and `GEMINI` collapsed into `PROSE` — they were structurally identical except for one space in a resource citation, which does not justify a separate format.

Provider-named values force a new enum member per LLM provider even when providers produce identical output. The real distinction is output structure: markdown-capable vs. dense prose vs. JSON. Sub-labels (e.g. `PROSE_OPENAI`) can be added later only if concrete structural differences emerge. Protocol: PP-20260531-60dc12.

### AgentValidationException: generic name for all agent records

`AgentDescriptorValidationException` → `AgentValidationException`. The exception is now thrown from `AgentDescriptor`, `AgentCapability`, and `AgentDisposition` compact constructors. The old name was misleading when thrown from a capability.

### Validation at compact constructor: AgentCapability and AgentDisposition

`AgentCapability` validates all string fields at construction time: `name` (required, ≤100), `costHint` (optional, ≤200), `inputTypes`/`outputTypes`/`tags` items (≤200 each), `epistemicDomains` keys (≤200). Same character-set rules as `AgentDescriptor` (PP-20260530-2d6dbd).

`AgentDisposition` validates its 4 open-string axes: null-permissive, blank-rejecting, ≤200 chars, no banned characters. Each value object validates its own invariants at construction.

`AgentDescriptor` compact constructor extended to validate 10 optional String fields: version/provider/modelFamily/modelVersion (≤200), weightsFingerprint (≤255), vocabulary URIs (≤500), jurisdiction/dataHandlingPolicy (≤1000). `AgentDescriptorValidator` extended with `validateOptional`, `validateItems`, `validateMapKeys`.

All string fields flowing into the LLM payload or the A2A card are now injection surfaces protected at construction — no invalid record can exist anywhere in the system.

### EvalReport: format-grouped, no cross-format summary

`EvalReport` redesigned from flat shape (`List<EvalResult>` + single `EvalSummary`) to format-grouped shape (`Map<RenderFormat, List<EvalResult>> resultsByFormat` + `Map<RenderFormat, EvalSummary> summaryByFormat`). A single cross-format summary is meaningless — `CONCISENESS` scored on markdown cannot be averaged with dense prose, and `COMPLETENESS` (A2A only) has no meaning for prose formats. `LinkedHashMap` for deterministic iteration order.

### EvalDimension: COMPLETENESS for A2A + applicableFor

`COMPLETENESS` added as 5th `EvalDimension` (A2A_CARD only: all capabilities have non-empty quality descriptions). `applicableFor(RenderFormat)` static method returns the applicable dimension set per format: `{SECOND_PERSON, CONCISENESS, FACTUAL_FIDELITY, TONE}` for MARKDOWN/PROSE; `{COMPLETENESS, FACTUAL_FIDELITY}` for A2A_CARD. Placing the mapping on the enum means both `PromptJudge` and `EvalReport.build()` share the same source of truth without coupling.

### A2A completeness check: JSON-aware, not substring

For A2A_CARD, `completenessPass` is determined by parsing the rendered JSON and checking that each capability has a non-empty `description` field. The old substring-contains check always passes for JSON — capability names are always present in the `name` field of each JSON object regardless of description presence.

## §eidos23 — Real-world profile library + personality preservation

### EvalCase sealed interface

`EvalCase` was a record. Changed to a sealed interface permitting `SyntheticEvalCase` (hand-crafted, no source profile) and `ProfiledEvalCase` (wraps `AgentProfile`, backed by a real-world YAML). The motivation: four new judges all require `AgentProfile` and `Optional<AgentProfile>` in a record component is a design smell. The sealed interface provides type safety at call sites without Optional guards.

`EvalDataset.all()` returns `List<EvalCase>` (the abstraction) — callers don't need to know the concrete type. `RealWorldEvalDataset.all()` returns `List<ProfiledEvalCase>` directly because callers need `profile()` access.

### AgentProfileLoader (test scope only)

Placed in `eval/src/test/java/` because it reads from test-classpath YAML resources and has no production use. Stage 0 validation runs at load time: each variant pair must differ on exactly the declared `primaryAxis` and be identical on all other `AgentDisposition` fields (including `delegation`). Throws `IllegalStateException` with descriptive message on violation.

Uses `ObjectMapper(YAMLFactory).findAndRegisterModules()` — required for Java record deserialization on Java 21+.

### Separate ProximityJudge (not a new EvalDimension)

`EvalDimension.applicableFor(format)` takes format only. Semantic proximity — how faithfully does the rendered prompt capture the original prose? — depends on profile availability, not format. Folding it into `EvalDimension` would add a second axis to `applicableFor()` and make `EvalResult.overall` incomparable across synthetic and real-world cases. `ProximityJudge` is a separate `@ApplicationScoped` bean with its own result type and floor.

### Three-stage personality preservation attribution

Stage 1 (`VocabularyExpressivenessJudge`): how precisely can a short phrase capture this personality axis from the prose? Four sequential calls, one per axis. Scores ≤ 2 identify vocabulary gaps that feed eidos#26.

Stage 2 (`TraitExpressionJudge`): blind-scores rendered text. User message is `rendered.content()` only — no descriptor, no profile. Direction match: HIGH ≥ 4, LOW ≤ 2, NEUTRAL always matches.

Stage 3 (`PairContrastJudge`): pairwise effect size measurement. Resolves renders by `evalCase.profile().name()`, not `evalCase.name()` (which carries a format suffix).

Attribution logic (in `PersonalityPreservationReport.build()`): s1 ≤ 2 → VOCABULARY_GAP; s1 ≥ 4 AND matchRate < 0.5 → RENDERER_FLATTENING; s1 ≥ 4 AND matchRate ≥ 0.5 AND s3 in [1,2] → PROFILE_DESIGN_GAP; s1 ≥ 4 AND matchRate ≥ 0.5 AND s3 > 2 → WORKING; else INSUFFICIENT_DATA. All diagnoses beyond VOCABULARY_GAP require `s1 >= 4`.

### Variant pair single-axis isolation

Profiles in a variant pair must differ on exactly one `AgentDisposition` axis (the declared `primaryAxis`); all other axes and `delegation` must be identical between pair members. Enforced by Stage 0 at `AgentProfileLoader.load()` time.

### Deferred

eidos#26 (Belbin/DISC/Big Five vocabulary), #27 (framework grounding in descriptor/renderer), #28 (Belbin phase routing in engine), #29 (personality framework mapping doc — prerequisite for #26), #30 (minor eval cleanup).
