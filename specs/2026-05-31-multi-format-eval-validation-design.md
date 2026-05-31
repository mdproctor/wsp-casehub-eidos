# Multi-Format Eval Coverage + String Field Validation Design

**Date:** 2026-05-31
**Issues:** eidos#21 (multi-format eval), eidos#20 (optional field validation), eidos#22 (capability validation)
**Branch:** issue-021-020-eval-coverage-validation

---

## Context

Three issues on one branch, sequenced by size. They share a common theme: the existing
implementation is too narrow — the eval harness covers only one render format, and the
validator covers only the four required AgentDescriptor fields. Both gaps leave injection
surfaces open and quality coverage incomplete.

---

## 1. RenderFormat redesign (prerequisite to #21)

### Problem

`RenderFormat` is provider-named (`CLAUDE_MD`, `OPENAI_SYSTEM`, `GEMINI`), which conflates
output structure with target LLM. This forces a format proliferation question: where is
Grok? Qwen? Mistral? The answer can't be "add a format per provider" — those formats would
be structurally identical.

`OPENAI_SYSTEM` and `GEMINI` are not two formats. Their rendered output differs by exactly
one space in resource citation (`label (uri)` vs `label(uri)`). That does not justify a
format distinction.

### Design

Three structure-named formats:

```java
enum RenderFormat {
    MARKDOWN,  // rich markdown — Claude and other markdown-capable models
    PROSE,     // flowing paragraphs — OpenAI, Gemini, Grok, Qwen, Mistral, Llama, ...
    A2A_CARD   // JSON — machine-readable agent identity card
}
```

Provider sub-labels (e.g. `PROSE_OPENAI`) can be added later only if concrete structural
differences emerge. The current evidence does not justify them.

### Changes

- `SystemPromptRenderer.RenderFormat`: rename members
- `EidosRenderPipeline`: collapse `assembleOpenAiSystem()` + `assembleGemini()` →
  `assembleProse()`. Resources standardise to `label (uri)` (natural English form).
  Rename `assembleClaudeMarkdown()` → `assembleMarkdown()`
- `EidosRenderPipeline.usesEnrichment()`: MARKDOWN and PROSE → true; A2A_CARD → false
- Cache keys: `format.name()` changes from `"CLAUDE_MD"` → `"MARKDOWN"` etc. — correct,
  old entries invalidate on deploy
- All call sites updated (EvalDataset, AgentPromptContext usages)

`ClaudeMarkdownRenderer` class name: left as-is. The LLM enrichment step is Claude-specific;
the name is accurate for the implementation. Rename is deferred.

---

## 2. #21 — Multi-format eval coverage

### EvalDimension

Add `COMPLETENESS` (A2A_CARD only: all capabilities present with quality descriptions).
Add `applicableFor(RenderFormat)` static method — shared by `PromptJudge` and
`EvalReport.build()` to avoid coupling those two:

```java
public static Set<EvalDimension> applicableFor(RenderFormat format) {
    return switch (format) {
        case MARKDOWN, PROSE -> EnumSet.of(SECOND_PERSON, CONCISENESS, FACTUAL_FIDELITY, TONE);
        case A2A_CARD        -> EnumSet.of(COMPLETENESS, FACTUAL_FIDELITY);
    };
}
```

Dimension semantics are unchanged for existing members. `COMPLETENESS` assesses whether
each declared capability has a meaningful, non-empty description in the rendered card —
distinct from the deterministic `completenessPass` boolean, which checks presence/name
inclusion.

### PromptJudge — format-aware evaluation

**System prompts (3 constants):**

- `MARKDOWN_SYSTEM_PROMPT` — existing prompt, renamed. Scores all four prose dimensions
  against a markdown-structured prompt.
- `PROSE_SYSTEM_PROMPT` — dense prose rubric. CONCISENESS is stricter: any markdown
  header or bullet reduces the score to ≤2. Otherwise same four dimensions.
- `A2A_SYSTEM_PROMPT` — JSON evaluation. COMPLETENESS: every capability has a non-empty
  description. FACTUAL_FIDELITY: no hallucinated capabilities absent from the descriptor.
  SECOND_PERSON and TONE are not assessed — they are not meaningful for JSON output.

**JSON schemas (2 constants):**

- `PROSE_JUDGE_RESPONSE_FORMAT` — existing schema (4 dimensions + issues), renamed
- `A2A_JUDGE_RESPONSE_FORMAT` — 2-dimension schema (COMPLETENESS + FACTUAL_FIDELITY + issues)

**`evaluate()` dispatch:**

```
1. applicable = EvalDimension.applicableFor(format)
2. select system prompt and schema based on format
3. compute completenessPass (format-aware — see below)
4. call judge LLM
5. parseResponse(json, applicable) — iterates applicable set only
6. overall = mean of applicable dimension scores
```

**Format-aware `completenessPass`:**

- MARKDOWN / PROSE: existing `rendered.content().contains(capabilityName)` substring check
- A2A_CARD: parse rendered JSON, find `capabilities` array, check each entry has a
  non-empty `description` field. `missingCapabilities` for A2A = capability names whose
  description is absent or blank.

Rationale: for A2A, capability names always appear in the `name` field of the JSON —
the substring check is trivially always true and provides no regression value.

**`parseResponse(String json, Set<EvalDimension> applicable)`:**

Iterates `applicable`, not `EvalDimension.values()`. Throws `IllegalStateException` if
any applicable dimension key is absent in the judge response. Extra keys in the response
that are not in `applicable` are ignored — the judge prompt controls what the model
returns, but defensive parsing is correct.

### EvalDataset — new cases

Reuse existing descriptors — the renderer is under test, not the descriptor.

| Case | Descriptor | Format | Stresses |
|---|---|---|---|
| `devtownPlannerProse` | devtown-planner | PROSE | CONCISENESS with goal + disposition in dense prose |
| `maximalProse` | maximal | PROSE | Resources as `label (uri)` in prose |
| `devtownPlannerA2a` | devtown-planner | A2A_CARD | COMPLETENESS: 2 capabilities need descriptions |
| `minimalA2a` | minimal | A2A_CARD | Edge: no capabilities → empty array, trivially complete |

Total: 9 cases (5 existing MARKDOWN + 2 PROSE + 2 A2A_CARD).

### EvalReport — clean break

```java
public record EvalReport(
    Instant timestamp,
    String judgeModel,
    Map<RenderFormat, List<EvalResult>> resultsByFormat,
    Map<RenderFormat, EvalSummary> summaryByFormat
)
```

No cross-format summary. `build(List<EvalResult>, String judgeModel)` groups by format,
computes `EvalSummary` per format using `EvalDimension.applicableFor()`. `EvalSummary`
is structurally unchanged — it operates over the applicable dimension set for its format
(4 dimensions for MARKDOWN/PROSE, 2 for A2A_CARD).

### EvalReportWriter

`summaryTable()` iterates `summaryByFormat`, printing one labelled section per format
with case count. `writeJson()` serialises the new shape.

### PromptEvalTest

Per-format floor constants (all 3.5 initially — calibrate after first run):

```java
private static final Map<RenderFormat, Double> SCORE_FLOORS = Map.of(
    MARKDOWN, 3.5, PROSE, 3.5, A2A_CARD, 3.5
);
```

Assert `allCasesComplete` and `meanOverall >= floor` per format entry in
`summaryByFormat`.

---

## 3. #22 — AgentCapability string field validation

### Motivation

`AgentCapability` string fields (`name`, `costHint`, `inputTypes` items, `outputTypes`
items, `tags` items, `epistemicDomains` keys) all flow into the LLM payload via
`EidosRenderPipeline.buildDescriptorPayload()` and into the A2A card JSON. None are
validated. A capability name with a BiDi control character (e.g. U+202E RLO) would
inject into the LLM payload undetected.

### AgentDescriptorValidationException → AgentValidationException

The old name is misleading once `AgentCapability` throws it. Rename before extending.
All catch sites updated mechanically.

### AgentDescriptorValidator extensions

Four new package-private methods:

**`validateRequired(fieldName, value, maxLength)`** — existing `validateField` logic
exposed under a clear name. Non-null, non-blank, within length, no banned chars.

**`validateOptional(fieldName, value, maxLength)`** — null-permissive, blank-rejecting.
If null: skip. If provided: must not be blank, within length, no banned chars.

**`validateItems(fieldName, items, maxLength)`** — null/empty-safe. Iterates,
calls `validateOptional` per item with index-qualified name (`inputTypes[0]`, etc.).

**`validateMapKeys(fieldName, keys, maxLength)`** — null/empty-safe. Iterates key set,
calls `validateOptional` per key.

New length constants (package-private):

| Constant | Value | Applies to |
|---|---|---|
| `MAX_CAPABILITY_NAME` | 100 | `AgentCapability.name` |
| `MAX_CAPABILITY_STRING` | 200 | `costHint`, list items, map keys |

### AgentCapability compact constructor

```
name        → validateRequired  (MAX_CAPABILITY_NAME)
costHint    → validateOptional  (MAX_CAPABILITY_STRING)
inputTypes  → validateItems     (MAX_CAPABILITY_STRING)
outputTypes → validateItems     (MAX_CAPABILITY_STRING)
tags        → validateItems     (MAX_CAPABILITY_STRING)
epistemicDomains keys → validateMapKeys (MAX_CAPABILITY_STRING)
```

`epistemicDomains` values are `Double` — not a string injection surface, no validation.

---

## 4. #20 — AgentDescriptor optional field validation

### AgentDescriptorValidator — additional length constants

| Constant | Value | Applies to |
|---|---|---|
| `MAX_VERSION` | 200 | `version` |
| `MAX_PROVIDER` | 200 | `provider`, `modelFamily`, `modelVersion` |
| `MAX_WEIGHTS_FINGERPRINT` | 255 | `weightsFingerprint` |
| `MAX_VOCABULARY_URI` | 500 | `domainVocabulary`, `slotVocabulary`, `dispositionVocabulary` |
| `MAX_JURISDICTION` | 1000 | `jurisdiction`, `dataHandlingPolicy` |
| `MAX_DISPOSITION_AXIS` | 200 | All four `AgentDisposition` string axes |

### AgentDescriptor compact constructor

Appends `validateOptional` calls for all 10 optional String fields after the existing
`validate(agentId, name, slot, tenancyId)` call. Compact constructor signature unchanged.

### AgentDisposition compact constructor (new)

```
socialOrient   → validateOptional (MAX_DISPOSITION_AXIS)
ruleFollowing  → validateOptional (MAX_DISPOSITION_AXIS)
riskAppetite   → validateOptional (MAX_DISPOSITION_AXIS)
autonomy       → validateOptional (MAX_DISPOSITION_AXIS)
```

`delegation` is boolean — no validation needed. All four axes may be null (fully
optional); if provided, must not be blank and must not carry banned characters.

`AgentDisposition` validates its own invariants at construction, following the same
principle as `AgentDescriptor` (PP-20260530-2d6dbd). An `AgentDisposition` is
self-consistent regardless of how it is constructed.

---

## 5. Testing

### #21 — new/updated tests

**`PromptJudgeTest` (unit, no CDI):**
- Update existing MARKDOWN tests (format reference rename)
- `evaluate_a2a_scores_only_completeness_and_factual_fidelity`
- `evaluate_a2a_completeness_pass_checks_json_description`
- `evaluate_a2a_completeness_fail_when_description_absent`
- `evaluate_prose_selects_prose_system_prompt`
- `parseResponse_throws_when_applicable_dimension_missing`

**`EvalDatasetTest`:**
- Update case count assertion (9 not 5)
- `all_cases_cover_all_three_formats`
- `a2a_cases_have_capabilities_for_completeness_coverage`

**`EvalReportTest` (new):**
- `build_groups_results_by_format`
- `build_computes_per_format_summary_with_correct_dimensions`
- `a2a_summary_has_exactly_two_dimensions`

**`PromptEvalTest`:** per-format floor assertions

### #22 — new tests

**`AgentCapabilityTest` (new, api/):**
- null/blank/overlength/banned-char for `name` (required)
- null allowed for `costHint`; blank/banned-char throws
- list item and map key injection variants
- valid construction succeeds

### #20 — new/updated tests

**`AgentDispositionTest` (new, api/):**
- null axes allowed; blank throws; injection char throws; overlength throws

**`AgentDescriptorValidatorTest`:** extend with `validateOptional`, `validateItems`,
`validateMapKeys` coverage

**`AgentDescriptorTest`:** one representative test per optional field category

**All existing tests:** update `AgentDescriptorValidationException` → `AgentValidationException`

---

## Protocol coherence

| Protocol | Status |
|---|---|
| PP-20260530-2d6dbd — validation in compact constructor | Upheld for AgentDescriptor, AgentDisposition, AgentCapability |
| PP-20260529-368527 — format-specific eval concerns in dedicated step | Upheld: A2A judge uses separate system prompt and schema |
| PP-20260529-35f3bd — structural fallback per format | Upheld: assembleProse() provides structural fallback; assembleMarkdown() unchanged |
| PP-20260529-5c883f — cache key includes format | Upheld: format.name() in key; names change correctly on rename |
| PP-20260529-b83bf1 — enrichment constants in EidosRenderPipeline | Upheld: judge constants are in PromptJudge (eval module), not renderer |

---

## Out of scope (tracked)

- eidos#23: real-world agent profile library — blocked by #21, #20, #22
- `ClaudeMarkdownRenderer` class rename — cosmetic, deferred
- `AgentCapability` value field validation (`qualityHint` range, `latencyHintP50Ms` positive) — numeric, not an injection surface; deferred
