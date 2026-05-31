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

`OPENAI_SYSTEM` and `GEMINI` are not two formats. Their rendered output (both enriched and
structural fallback paths) differs by exactly two details:
- Resource citation: `label (uri)` (OpenAI, with space) vs `label(uri)` (Gemini, no space)
- Resource line terminator: `".\n"` (OpenAI) vs `"\n"` (Gemini, no trailing period)

That does not justify a format distinction.

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

- `SystemPromptRenderer.RenderFormat`: rename members. This is a **binary-breaking API
  change** for any caller compiled against the old constants. Acceptable under the clean-slate
  deployment policy — no existing installations.
- `EidosRenderPipeline`: collapse `assembleOpenAiSystem()` + `assembleGemini()` →
  `assembleProse()`. Resources standardise to `label (uri)` (natural English form).
  Rename `assembleClaudeMarkdown()` → `assembleMarkdown()`
- `EidosRenderPipeline.usesEnrichment()`: MARKDOWN and PROSE → true; A2A_CARD → false
- Cache keys: `format.name()` changes from `"CLAUDE_MD"` → `"MARKDOWN"` etc. — correct,
  old entries invalidate on deploy. On deploy there will be a brief cold-cache window for all
  formats simultaneously; acceptable for an internal cache.
- All call sites updated (EvalDataset, AgentPromptContext usages). **EvalDataset must be
  updated atomically with the enum rename** — it will fail to compile on the old constants.

`ClaudeMarkdownRenderer` class name: deferred. Track as a named issue — do not leave it as
an open-ended cosmetic deferral. After this change the class renders three formats, two of
which have no Claude relationship.

`usesEnrichment()` is a half-truth: it returns `false` for A2A, which the caller reads as
"no enrichment" — but A2A does use LLM enrichment, just a different kind. Track as a
named issue: rename to `usesProseLlmEnrichment()` or introduce `EnrichmentType { PROSE, A2A,
NONE }`. Not blocking this PR.

---

## 2. #21 — Multi-format eval coverage

### EvalDimension

Add `COMPLETENESS` (A2A_CARD only: LLM judges whether each declared capability has a
**meaningful** description — distinct from the programmatic `completenessPass` boolean, which
checks presence/non-blank).

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

**`COMPLETENESS` vs `completenessPass` — naming clarity:**
- `completenessPass` (programmatic, boolean): are all declared capability names present in the
  rendered output, and for A2A, does each have a non-blank `description` field?
- `COMPLETENESS` (LLM-judged, scored 0–5): are the descriptions **meaningful**? Not just
  present, but informative enough for agent-to-agent discovery.

These are deliberately distinct. `COMPLETENESS` as an enum member name is retained; its
docstring must call out this distinction explicitly.

### PromptJudge — format-aware evaluation

**Dispatch on `rendered.format()`, not `evalCase.context().format()`.**
The authoritative source is what format was actually produced. Both should agree, but the
rendered content is the artifact being evaluated.

**System prompts (3 constants):**

- `MARKDOWN_SYSTEM_PROMPT` — existing prompt, renamed. Scores all four prose dimensions
  against a markdown-structured prompt.
- `PROSE_SYSTEM_PROMPT` — dense prose rubric. CONCISENESS is stricter: any markdown
  header or bullet reduces the score to ≤2 (note: this instruction is LLM-probabilistic, not
  enforced programmatically). Otherwise same four dimensions.
- `A2A_SYSTEM_PROMPT` — JSON evaluation. COMPLETENESS: every capability has a meaningful,
  non-empty description. FACTUAL_FIDELITY: no hallucinated capabilities absent from the
  descriptor. SECOND_PERSON and TONE not assessed — not meaningful for JSON output.

  **Zero-capabilities edge case:** include the sentence "If no capabilities are declared,
  COMPLETENESS score is 5" in `A2A_SYSTEM_PROMPT`. Without this, the judge LLM may
  hallucinate reasoning for a vacuously true condition.

**JSON schemas (2 constants):**

- `PROSE_JUDGE_RESPONSE_FORMAT` — existing schema (4 dimensions + issues), renamed
- `A2A_JUDGE_RESPONSE_FORMAT` — 2-dimension schema (COMPLETENESS + FACTUAL_FIDELITY + issues)

**Test constants (for PromptJudgeTest):**

- `VALID_MARKDOWN_JUDGE_JSON` — 4-dimension response (rename existing `VALID_JUDGE_JSON`)
- `VALID_A2A_JUDGE_JSON` — 2-dimension response for use in A2A stub tests:

```json
{
  "COMPLETENESS":      { "score": 4, "reasoning": "All capabilities have descriptions." },
  "FACTUAL_FIDELITY":  { "score": 5, "reasoning": "No hallucinated capabilities." },
  "issues": []
}
```

**`evaluate()` dispatch:**

```
1. applicable = EvalDimension.applicableFor(rendered.format())   ← rendered.format(), not evalCase
2. select system prompt and schema based on rendered.format()
3. compute completenessPass (format-aware — see below)
4. call judge LLM
5. parseResponse(json, applicable) — iterates applicable set only
6. overall = mean of applicable dimension scores
```

**Format-aware `completenessPass`:**

- MARKDOWN / PROSE: existing `rendered.content().contains(capabilityName)` substring check
- A2A_CARD: parse rendered JSON, find `capabilities` array, check each entry has a
  non-blank `description` field. `missingCapabilities` for A2A = capability names whose
  description is absent or blank. If JSON parse fails, throw `IllegalStateException` with
  context: `"A2A completeness check: rendered content is not valid JSON — this is a
  renderer bug"`.

Rationale: for A2A, capability names always appear in the `name` field of the JSON —
the substring check is trivially always true and provides no regression value.

**JSON coupling note:** `PromptJudge` parses A2A card JSON and expects `capabilities[].description`.
This couples `PromptJudge` to the A2A card structure in `EidosRenderPipeline.assembleA2aCard()`.
The JSON field names `"capabilities"` and `"description"` must be extracted as package-private
constants in `PromptJudge` to make the coupling explicit and locate-able.

**`parseResponse(String json, Set<EvalDimension> applicable)`:**

Iterates `applicable`, not `EvalDimension.values()`. Throws `IllegalStateException` if
any applicable dimension key is absent in the judge response. Extra keys in the response
that are not in `applicable` are silently ignored — the judge prompt controls what the model
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

### EvalSummary — add `applicable`

```java
public record EvalSummary(
    boolean allCasesComplete,
    Set<EvalDimension> applicable,           // ← new: which dimensions this format tracks
    Map<EvalDimension, Double> meanByDimension,
    EvalDimension lowestScoringDimension,
    double meanOverall
)
```

`applicable` is derived from `EvalDimension.applicableFor(format)` in `EvalReport.build()`.
This makes the partial-map contract self-documenting: callers know which keys are present in
`meanByDimension` without consulting a separate method. `meanByDimension` contains exactly the
dimensions in `applicable` — no more, no fewer.

### EvalReport — clean break

```java
public record EvalReport(
    Instant timestamp,
    String judgeModel,
    Map<RenderFormat, List<EvalResult>> resultsByFormat,
    Map<RenderFormat, EvalSummary> summaryByFormat
)
```

No cross-format summary. `build(List<EvalResult>, String judgeModel)` groups by
`result.rendered().format()`, computes `EvalSummary` per format using
`EvalDimension.applicableFor()`. `EvalSummary` operates over the applicable dimension set for
its format (4 dimensions for MARKDOWN/PROSE, 2 for A2A_CARD).

**Atomicity constraint:** Adding `EvalDimension.COMPLETENESS` and rewriting `EvalReport.build()`
must be in the same commit. The old `build()` iterates `EvalDimension.values()` and calls
`r.scores().get(d).score()`. After `COMPLETENESS` is added, A2A results have no entry for
`SECOND_PERSON`; non-A2A results have no entry for `COMPLETENESS`. Both NPE. There is no safe
intermediate state.

### EvalResult

```java
public record EvalResult(
    EvalCase evalCase,
    RenderedPrompt rendered,
    boolean completenessPass,
    List<String> missingCapabilities,
    Map<EvalDimension, EvalScore> scores,    // sparse: only applicable dimensions present
    double overall,    // 0.0–5.0; mean of applicable dimension scores (2 for A2A, 4 for others)
    List<String> issues
)
```

`scores` is a **sparse map**: it contains entries only for the applicable dimensions of its
format. Callers must not access `scores.get(d)` without knowing the format.

### EvalReportWriter

`summaryTable()` iterates `summaryByFormat`, printing one labelled section per format with
case count and applicable dimension rows. Format-section headers must be present in output so
`EvalReportWriterTest` can assert on them.

`writeJson()` serialises the new shape.

### PromptEvalTest

Per-format floor constants (all 3.5 initially — calibrate after first run):

```java
private static final Map<RenderFormat, Double> SCORE_FLOORS = Map.of(
    MARKDOWN, 3.5, PROSE, 3.5, A2A_CARD, 3.5
);
```

Assert per-format entry in `summaryByFormat` — call `report.summaryByFormat().get(FORMAT)`:

```java
SCORE_FLOORS.forEach((format, floor) -> {
    final EvalSummary s = report.summaryByFormat().get(format);
    assertThat(s.allCasesComplete())
        .as("All %s cases complete", format)
        .isTrue();
    assertThat(s.meanOverall())
        .as("Mean judge score for %s", format)
        .isGreaterThanOrEqualTo(floor);
});
```

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

**Message prefix update (same commit as rename):** The existing constructor produces
`"AgentDescriptor field 'name': ..."`. After rename, change the message prefix to
`"Agent field 'name': ..."` — correct for descriptors, capabilities, and dispositions alike.
The old prefix is misleading when thrown from `AgentCapability` or `AgentDisposition`.

### AgentDescriptorValidator extensions

Four new package-private methods:

**`validateRequired(fieldName, value, maxLength)`** — existing `validateField` logic
exposed under a clear name. Non-null, non-blank, within length, no banned chars.

**`validateOptional(fieldName, value, maxLength)`** — null-permissive, blank-rejecting.
If null: skip. If provided: must not be blank, within length, no banned chars.

**`validateItems(fieldName, items, maxLength)`** — null/empty-safe. If `items` is null or
empty, skip. If non-null and non-empty, iterates. **A null element within a non-null list
throws** — a null item is not "absent", it is a bug at the call site. Error message uses
index-qualified name: `inputTypes[0]`, `outputTypes[2]`, etc.

**`validateMapKeys(fieldName, keys, maxLength)`** — null/empty-safe. Iterates key set.
Error message uses bracket-qualified name: `epistemicDomains['reinforcement-learning']`.

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
| `MAX_WEIGHTS_FINGERPRINT` | 255 | `weightsFingerprint` (open-format field; SHA-256 hex is 64 chars, 255 is a generous upper bound for any hash/fingerprint encoding) |
| `MAX_VOCABULARY_URI` | 500 | `domainVocabulary`, `slotVocabulary`, `dispositionVocabulary` |
| `MAX_JURISDICTION` | 1000 | `jurisdiction`, `dataHandlingPolicy` (both can be open-ended prose) |
| `MAX_DISPOSITION_AXIS` | 200 | All four `AgentDisposition` string axes |

URI format (scheme, authority, path structure) is **not validated** — only injection
characters and length are checked. IRI handling is out of scope.

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

### Existing tests — disposition

**`EvalResultCompletenessTest` — retire.**
This test duplicates the string-contains completeness logic from `PromptJudge.evaluate()`.
After the change, A2A completeness uses JSON parsing, not string contains. The file becomes
stale and misleading. The string-contains path is covered by `PromptJudgeTest` (existing
`evaluate_detects_completeness_when_cap_present` and `evaluate_detects_missing_cap`). The
JSON path is covered by the new A2A tests below. Delete `EvalResultCompletenessTest.java`.

**`EvalDatasetTest.all_cases_use_claude_md_format` — delete.**
After the enum rename (`CLAUDE_MD` → `MARKDOWN`) and the addition of PROSE/A2A cases, this
test fails to compile then fails as an assertion. Delete it. Replace with
`all_cases_cover_all_three_formats` (see below).

### #21 — new/updated tests

**`PromptJudgeTest` (unit, no CDI):**

Rename `VALID_JUDGE_JSON` → `VALID_MARKDOWN_JUDGE_JSON`. Add `VALID_A2A_JUDGE_JSON`
(see §2 above for the constant definition).

- Update existing MARKDOWN tests: rename the constant references
- `evaluate_a2a_scores_only_completeness_and_factual_fidelity` — uses `VALID_A2A_JUDGE_JSON`
  stub; asserts scores map has exactly those two keys
- `evaluate_a2a_completeness_pass_checks_json_description` — rendered content is a valid
  A2A JSON with non-blank capability descriptions; asserts `completenessPass = true`
- `evaluate_a2a_completeness_fail_when_description_absent` — rendered A2A JSON has a
  capability with no `description` key; asserts `completenessPass = false`, `missingCapabilities`
  contains that capability name
- `evaluate_a2a_completeness_fail_when_capabilities_array_empty` — descriptor has capabilities
  but rendered A2A JSON has empty `capabilities` array; asserts `completenessPass = false`
- `evaluate_prose_selects_prose_system_prompt` — stub captures which system message was sent;
  asserts it contains a phrase unique to `PROSE_SYSTEM_PROMPT`
- `parseResponse_throws_when_applicable_dimension_missing` — applicable = {COMPLETENESS,
  FACTUAL_FIDELITY}; judge JSON omits COMPLETENESS; asserts `IllegalStateException`
- `parseResponse_ignores_extra_dimensions_not_in_applicable` — applicable = {COMPLETENESS,
  FACTUAL_FIDELITY}; judge JSON contains all 4 dimensions; asserts result scores map has
  exactly 2 entries and does not throw

**`EvalDatasetTest`:**

- Update case count assertion: 9, not 5
- Delete `all_cases_use_claude_md_format`
- `all_cases_cover_all_three_formats` — asserts at least 2 cases per format (not just 1):
  MARKDOWN ≥ 5, PROSE ≥ 2, A2A_CARD ≥ 2
- `a2a_cases_have_capabilities_for_completeness_coverage` — asserts every A2A_CARD case
  whose descriptor has capabilities has at least one capability in the descriptor

**`EvalReportTest` (new):**

- `build_groups_results_by_format` — mixed results list; asserts `resultsByFormat` keys
  match the formats represented in the input
- `build_computes_per_format_summary_with_correct_dimensions` — MARKDOWN summary has 4-key
  `meanByDimension`; A2A summary has 2-key `meanByDimension`
- `a2a_summary_has_exactly_two_dimensions` — `applicable` field equals {COMPLETENESS,
  FACTUAL_FIDELITY}; `meanByDimension` has exactly those two keys

**`EvalReportWriterTest` — rewrite:**

`sampleReport()` must produce the new `EvalReport` shape. Build two results: one MARKDOWN
(scores for all 4 dimensions), one A2A_CARD (scores for COMPLETENESS and FACTUAL_FIDELITY
only). Call `EvalReport.build(List.of(markdownResult, a2aResult), "test-judge")`.

- `summaryTable_returns_non_empty_string` — unchanged assertion, update fixture
- `summaryTable_contains_format_section_headers` — assert table contains "MARKDOWN" and
  "A2A_CARD" section labels (replaces `summaryTable_contains_dimension_names` which iterated
  all `EvalDimension.values()` — invalid after sparse-map change)
- `summaryTable_markdown_section_contains_four_dimensions` — assert MARKDOWN section
  contains all 4 dimension names
- `summaryTable_a2a_section_contains_two_dimensions` — assert A2A section contains
  COMPLETENESS and FACTUAL_FIDELITY, does not contain SECOND_PERSON or TONE
- `writeJson_creates_valid_json_file` — unchanged assertion, update fixture
- `writeJson_round_trips_judge_model` — unchanged assertion, update fixture

**`PromptEvalTest`:** per-format floor assertions as specified in §2.

### #22 — new tests

**`AgentCapabilityTest` (new, api/):**
- null/blank/overlength/banned-char for `name` (required)
- null allowed for `costHint`; blank/banned-char throws
- list item and map key injection variants
- null item within non-null list throws for `inputTypes`
- valid construction succeeds

### #20 — new/updated tests

**`AgentDispositionTest` (new, api/):**
- null axes allowed; blank throws; injection char throws; overlength throws

**`AgentDescriptorValidatorTest`:** extend with `validateOptional`, `validateItems`,
`validateMapKeys` coverage; include null-item-in-list rejection test for `validateItems`

**`AgentDescriptorTest`:** one representative test per optional field category

**All existing tests:** update `AgentDescriptorValidationException` → `AgentValidationException`;
update error message assertions from `"AgentDescriptor field"` → `"Agent field"`

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
- `ClaudeMarkdownRenderer` class rename — **open a named issue**, do not leave as informal
  deferral. After this change the class renders MARKDOWN, PROSE, and A2A paths with no
  Claude-specific coupling in the non-markdown paths.
- `usesEnrichment()` rename to `usesProseLlmEnrichment()` or replacement with
  `EnrichmentType { PROSE, A2A, NONE }` — **open a named issue**. Current method is a
  half-truth: returns false for A2A, which the caller reads as "no enrichment", but A2A
  does call an LLM via `A2A_PROMPT_TEMPLATE`.
- `EvalConfig` consolidation — if a fourth format is added, consolidate format→dimensions,
  system prompt, and response schema into a single `EvalConfig` record rather than three
  parallel switch statements. Revisit before adding any new format.
- `AgentCapability` null-to-empty list normalization — compact constructor is the right
  place to do this; deferred as a separate cleanup.
- `AgentCapability` value field validation (`qualityHint` range, `latencyHintP50Ms`
  positive) — numeric, not an injection surface; deferred
