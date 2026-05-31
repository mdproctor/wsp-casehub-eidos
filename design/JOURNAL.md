# Design Journal — issue-021-020-eval-coverage-validation

## RenderFormat: structure-named over provider-named

**Decision:** Rename `RenderFormat` from provider-named (`CLAUDE_MD`, `OPENAI_SYSTEM`, `GEMINI`) to structure-named (`MARKDOWN`, `PROSE`, `A2A_CARD`). Collapse `OPENAI_SYSTEM` + `GEMINI` into `PROSE` — they were structurally identical except for one space in resource citation formatting, which does not justify a separate format.

**Rationale:** Provider-named formats force a new enum member per LLM provider (Grok, Qwen, Mistral, Llama) even when those providers produce structurally identical output. The real distinction is output structure: markdown-capable vs. dense prose vs. JSON. Any LLM can be targeted with any structural format. Sub-labels for provider-specific specialisation (e.g. `PROSE_OPENAI`) can be added later only if concrete structural differences emerge.

**Impact:** `EidosRenderPipeline` loses `assembleGemini()` and `assembleGeminiStructural()` (merged into `assembleProse()`). `EidosSystemPromptRenderer`, `DefaultReactiveSystemPromptRenderer`, and all test files updated.

## AgentValidationException: generic name to cover all agent records

**Decision:** Rename `AgentDescriptorValidationException` → `AgentValidationException`.

**Rationale:** The exception is thrown from `AgentDescriptor`, `AgentCapability`, and `AgentDisposition` compact constructors. `AgentDescriptorValidationException` is misleading when thrown from a capability. Generic name covers all agent domain record validation without implying a specific record type.

## Validation at compact constructor: AgentCapability and AgentDisposition

**Decision (eidos#22):** `AgentCapability` validates all string fields at construction time: `name` (required, ≤100), `costHint` (optional, ≤200), `inputTypes`/`outputTypes`/`tags` items (≤200 each), `epistemicDomains` keys (≤200). Following the same principle as `AgentDescriptor` (PP-20260530-2d6dbd).

**Decision (eidos#20):** `AgentDisposition` validates its 4 open-string axes at construction time: null-permissive (axes are optional), blank-rejecting, ≤200 chars, no banned characters. Each value object validates its own invariants — it is self-consistent regardless of how it is constructed.

**Decision (eidos#20):** `AgentDescriptor` compact constructor extended to validate 10 optional String fields: version, provider, modelFamily, modelVersion (≤200), weightsFingerprint (≤255), domainVocabulary/slotVocabulary/dispositionVocabulary (≤500 — URI fields), jurisdiction/dataHandlingPolicy (≤1000 — compliance-text fields). Character-set rules (no C0/C1, BiDi controls, zero-width) apply uniformly. `AgentDescriptorValidator` extended with `validateOptional` (null-permissive, blank-rejecting), `validateItems`, `validateMapKeys` helpers.

**Rationale:** All string fields included in the LLM payload via `EidosRenderPipeline.buildDescriptorPayload()` or the A2A card JSON are injection surfaces. Validation at construction makes the guarantee system-wide — no invalid descriptor can exist in any registry, in any test, in any intermediate context.

## EvalReport: format-grouped, no cross-format summary

**Decision:** `EvalReport` redesigned from flat shape (`List<EvalResult>` + single `EvalSummary`) to format-grouped shape (`Map<RenderFormat, List<EvalResult>> resultsByFormat` + `Map<RenderFormat, EvalSummary> summaryByFormat`).

**Rationale:** With multiple output formats, a single cross-format `EvalSummary` is meaningless — `CONCISENESS` scored on markdown prose cannot be averaged with `CONCISENESS` scored on dense prose, and `COMPLETENESS` (A2A only) has no meaning for prose formats. Per-format summaries eliminate the category error. `LinkedHashMap` used for deterministic iteration order.

## EvalDimension: COMPLETENESS for A2A + applicableFor

**Decision:** Add `COMPLETENESS` as 5th `EvalDimension` (A2A_CARD only: all capabilities have non-empty quality descriptions). Add `applicableFor(RenderFormat)` static method returning the applicable dimension set per format: `{SECOND_PERSON, CONCISENESS, FACTUAL_FIDELITY, TONE}` for MARKDOWN/PROSE; `{COMPLETENESS, FACTUAL_FIDELITY}` for A2A_CARD.

**Rationale:** SECOND_PERSON and TONE are not meaningful dimensions for JSON output. COMPLETENESS (capability description quality) is the A2A-specific quality signal. Putting the format-to-dimension mapping on the enum means both `PromptJudge` and `EvalReport.build()` share the same source of truth without coupling to each other.

## A2A completeness check: JSON-aware, not substring

**Decision:** For A2A_CARD format, `completenessPass` is determined by parsing the rendered JSON and checking that each capability has a non-empty `description` field — not the existing substring-contains check. The substring check always passes for A2A (capability names always appear in the `name` field of JSON objects).

**Rationale:** The substring check was a regression waiting to happen — it would always report `completenessPass = true` for A2A regardless of whether descriptions were generated. The JSON-parse check is the correct semantic: a capability has been included iff its description is present and non-empty.
