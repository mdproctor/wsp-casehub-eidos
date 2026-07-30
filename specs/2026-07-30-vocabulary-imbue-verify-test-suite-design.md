# Vocabulary Imbue-and-Verify Test Suite — eidos#122

> **Status:** Design approved
> **Repo:** casehubio/eidos (eval module)
> **Issue:** eidos#122
> **Depends on:** eidos#113, eidos#117 (eval judges — landed)

## Goal

Prove that each personality framework (Jungian, Belbin, DISC, Conscientiousness)
produces expected behavioral signal when an LLM is imbued with it, both in
isolation and in pairwise composition. The suite splits into fast structural tests
(CI-safe, no LLM) and LLM eval tests (behind `-Peval`).

## Design Decisions

**Two-layer split.** Structural tests verify the deterministic machinery — axis
derivation, profile weights, prompt content. LLM eval tests verify behavioral
differentiation via judges. Rationale: structural regressions should be caught in
CI immediately; LLM eval is expensive and non-deterministic.

**New DispositionPresenceJudge.** MbtiAlignmentJudge is MBTI-specific (12-question
questionnaire for 4 dimensions). FunctionActivationJudge is Jungian-specific (8
cognitive functions). Neither applies to Belbin, DISC, or Conscientiousness. A
general-purpose judge that asks "does this prompt present as [vocabulary term]?"
fills the gap for all current and future vocabularies.

**All 6 pairwise combinations tested structurally.** Additive pairs (Jungian+Belbin,
Belbin+DISC, Belbin+Conscientiousness) assert both signals survive composition.
Redundant pairs (Jungian+DISC, Jungian+Conscientiousness, DISC+Conscientiousness)
assert detectable axis conflicts — proving the compatibility table is correct, not
just assumed.

**LLM eval tests only for additive pairs.** Redundant pairs are structurally proven
to conflict; burning LLM calls to confirm a contradictory prompt confuses a judge
adds noise, not signal.

## DispositionPresenceJudge

New judge in `casehub-eidos-eval`. Given a rendered system prompt and a vocabulary
term, asks an LLM whether the agent's personality description presents as the
declared term.

```java
DispositionPresenceJudge.evaluate(String systemPrompt, String termLabel, String termDescription)
  → DispositionPresenceResult(String termLabel, double score, String reasoning, boolean aligned)
```

- `score`: 0.0–1.0 alignment score
- `aligned`: true when score ≥ 0.7
- Uses the same CDI-injected ChatModel + ObjectMapper pattern as existing judges
- Response parsed via `PromptJudge.extractJson()`
- Unit-tested with stub ChatModel (no `@Tag("eval")`)

### Judge prompt

```
You are evaluating whether an AI agent's system prompt expresses a specific
personality or behavioral trait.

The trait: [termLabel] — [termDescription]

Read the system prompt carefully. Score how strongly this trait is expressed:
- 1.0: The trait is explicitly and clearly expressed
- 0.7: The trait is present but not dominant
- 0.4: Weakly present or only implied
- 0.0: Not present or contradicted

Return ONLY raw JSON — no markdown, no code blocks:
{ "score": number, "reasoning": string }
```

## Layer 1 — VocabularyImbueStructuralTest

`@QuarkusTest`, no `@Tag("eval")`. Runs in every build. Uses the real CDI container
(VocabularyRegistry, DescriptorCollector, SystemPromptRenderer) with a stub ChatModel
for the enrichment LLM call (returns empty narratives).

### Single-vocabulary assertions

**Jungian (via ENTJ mbtiType):**
- `dispositionProfile` has 8 entries, Te dominant (0.35), Ni auxiliary (0.20)
- All 5 axes derived: socialOrient=INDEPENDENT, ruleFollowing=STRICT,
  riskAppetite=MEASURED, autonomy=SEMI_AUTONOMOUS, conflictMode=COMPETING
- Rendered prompt contains profile function terms

**Belbin (SHAPER slot):**
- `dispositionProfile` is empty (Belbin has no defaultProfile)
- No axes derived (Belbin has no axisExactMatch)
- Rendered prompt contains slotLabel="Shaper" and slotDescription containing
  "Challenges the team"

**DISC (DOMINANCE):**
- `dispositionProfile` is empty (DISC has no defaultProfile)
- All 5 axes derived via axisExactMatch: socialOrient=INDEPENDENT,
  ruleFollowing=FLEXIBLE, riskAppetite=BOLD, autonomy=AUTONOMOUS,
  conflictMode=COMPETING
- Rendered prompt contains derived axis terms

**Conscientiousness (flat axes):**
- `dispositionProfile` is empty
- Axes set directly via flat strings (not derived)
- Rendered prompt contains axis values

### Pairwise structural assertions

| Pair | Type | Assertion |
|---|---|---|
| Jungian + Belbin | Additive | Jungian axes all derived + Belbin slot label present |
| Jungian + DISC | Redundant | Both derive axes — at least one axis value differs between Jungian-alone and DISC-alone derivation |
| Belbin + DISC | Additive | DISC axes derived + Belbin slot present, no interference |
| Belbin + Conscientiousness | Additive | Flat axes preserved + Belbin slot present |
| Jungian + Conscientiousness | Redundant | Jungian-derived axes differ from flat Conscientiousness values |
| DISC + Conscientiousness | Redundant | DISC-derived axes differ from flat Conscientiousness values |

### Garden gotcha encoded

`DescriptorCollector.deriveDispositionAxes()` silently no-ops without a registered
`dispositionVocabulary` (GE-20260730-d32015). All test fixtures enforce this:
Jungian descriptors always set `dispositionVocabulary: urn:casehub:vocab:jungian`,
Belbin descriptors always set `slotVocabulary: urn:casehub:vocab:belbin`, DISC
descriptors set `dispositionVocabulary: urn:casehub:vocab:disc`.

## Layer 2 — VocabularyImbueEvalTest

`@QuarkusTest` with `@Tag("eval")`. Runs under `-Peval` only. Uses live LLM for
both enrichment rendering and judge evaluation.

### Single-vocabulary eval

| Vocabulary | Judge | Assertion |
|---|---|---|
| Jungian (ENTJ) | MbtiAlignmentJudge | All 4 MBTI dimensions aligned |
| Jungian (ENTJ) | FunctionActivationJudge | Te activated under strategic-organizing scenario, Ni under long-range-planning scenario |
| Jungian (INTP) | PersonalityEvolutionJudge | 4x Ne activation produces dom/aux shift with valid weight tiers |
| Belbin (SHAPER) | DispositionPresenceJudge | aligned=true for "Shaper — Challenges the team to improve" |
| Belbin (TEAMWORKER) | DispositionPresenceJudge | aligned=true for "Teamworker — Cooperative, perceptive, diplomatic" |
| DISC (DOMINANCE) | DispositionPresenceJudge | aligned=true for "Dominance — Results-driven, direct, decisive" |
| Conscientiousness (STRICT) | DispositionPresenceJudge | aligned=true for strict rule-following |

### Pairwise composition eval (additive pairs only)

| Pair | Judges | Assertion |
|---|---|---|
| Jungian (ENTJ) + Belbin (SHAPER) | MbtiAlignmentJudge + DispositionPresenceJudge | Still ENTJ-aligned AND Shaper-aligned — both signals survive |
| Belbin (SHAPER) + DISC (DOMINANCE) | DispositionPresenceJudge (×2) | Both Shaper and Dominance aligned — no contradiction |

### Function activation scenarios

Domain-neutral scenarios for FunctionActivationJudge (reusable across single and
pairwise tests):

- **Te scenario:** "You must organize a team to complete a time-critical project.
  Describe your approach." → target: `te`
- **Ni scenario:** "You notice a pattern in recent events that others have missed.
  What do you see and what does it mean?" → target: `ni`

## File Layout

### New production classes

- `eval/src/main/java/io/casehub/eidos/eval/DispositionPresenceJudge.java`
- `eval/src/main/java/io/casehub/eidos/eval/DispositionPresenceResult.java` (record)

### New test classes

- `eval/src/test/java/io/casehub/eidos/eval/DispositionPresenceJudgeTest.java` — stub-model unit test, no tag
- `eval/src/test/java/io/casehub/eidos/eval/VocabularyImbueStructuralTest.java` — Layer 1, `@QuarkusTest`, no tag
- `eval/src/test/java/io/casehub/eidos/eval/VocabularyImbueEvalTest.java` — Layer 2, `@QuarkusTest`, `@Tag("eval")`
- `eval/src/test/java/io/casehub/eidos/eval/VocabularyImbueFixtures.java` — shared descriptor builders

### No new dependencies

DispositionPresenceJudge uses CDI-injected ChatModel, ObjectMapper, and
`PromptJudge.extractJson()` — all already available in the eval module.

## Test Fixtures — VocabularyImbueFixtures

Package-private class with static factory methods encoding correct vocabulary
configuration:

```java
jungianDescriptor(MbtiTypeTerm type)
    → AgentDescriptor with dispositionProfile from type.defaultProfile(),
      dispositionVocabulary = "urn:casehub:vocab:jungian"

belbinDescriptor(BelbinTerm role)
    → AgentDescriptor with slot = role.value(),
      slotVocabulary = "urn:casehub:vocab:belbin"

discDescriptor(DiscTerm style)
    → AgentDescriptor with dispositionVocabulary = "urn:casehub:vocab:disc",
      single-entry dispositionProfile with the DISC term value (e.g. "dominance")
      and weight 1.0. DescriptorCollector.deriveDispositionAxes() resolves the
      term via VocabularyRegistry, calls axisExactMatch for each axis, and
      populates the 5 derived axes.

conscientiousnessDescriptor(Map<DispositionAxis, String> axes)
    → AgentDescriptor with flat axis strings set directly

composite(AgentDescriptor base, AgentDescriptor overlay)
    → merges vocabulary fields from both: profile + vocab URI from base,
      slot + slotVocabulary from overlay (or vice versa)
```

## Compatibility Table (verified by tests)

| Pair | Relationship | Signal channels | Test type |
|---|---|---|---|
| Jungian + Belbin | Additive | Jungian: axis derivation + profile. Belbin: slot rendering. Orthogonal. | Structural + Eval |
| Jungian + DISC | Redundant | Both derive axes via axisExactMatch onto Conscientiousness. Conflicting values. | Structural only |
| Jungian + Conscientiousness | Redundant | Jungian derives axes that overwrite flat Conscientiousness values. | Structural only |
| Belbin + DISC | Additive | DISC: axis derivation. Belbin: slot rendering. Different channels. | Structural + Eval |
| Belbin + Conscientiousness | Additive | Conscientiousness: flat axes. Belbin: slot rendering. No interference. | Structural + Eval |
| DISC + Conscientiousness | Redundant | Both target the same axis values. | Structural only |
