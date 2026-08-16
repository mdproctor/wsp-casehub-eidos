# Structured Sarcasm/Humor Dimensions in Disposition Model

**Issue:** casehubio/eidos#140
**Date:** 2026-08-16
**Status:** Draft

## Summary

Extend the Eidos disposition model with structured sarcasm dimensions per agent, grounded in the Sarc7 paper (Xiong et al., 2025). A new `Sarc7Term` vocabulary enum in `casehub-eidos-vocab` provides 7 sarcasm types with cross-vocabulary mappings, prompt guidance methods, and read-only evaluation dimension metadata. The render pipeline gains a hybrid three-layer architecture to surface vocabulary-provided guidance generically while preserving framework-specific rendering fidelity. Reception-side sarcasm awareness is modeled as a standalone `AgentCapability`.

## Academic Reference

**Sarc7: Evaluating Sarcasm Detection and Generation with Seven Types and Emotion-Informed Techniques**
Xiong, Gao, Jeong, Fu, O'Brien, Sharma, Zhu — ACL WiNLP 2025, NeurIPS COLM SoLaR 2025.
arXiv: 2506.00658

Seven pragmatically defined sarcasm types (self-deprecating, brooding, deadpan, polite, obnoxious, raging, manic) with four evaluation dimensions (incongruity, shock value, context dependency, emotional tone). Emotion-based prompting yields best macro-F1 (0.3664, Gemini 2.5) for classification; structured prompts improve subtype alignment by 38.5% for generation (Claude 3.5 Sonnet).

## Scope

**In scope:**
- `Sarc7Term` vocabulary enum with 7 constants, cross-vocabulary mappings, responseStyleGuidance(), antiPatternWarning(), and read-only Sarc7 dimension fields
- `Sarc7VocabularyRegistrar` CDI bean
- Render pipeline hybrid refactor (Layer 1: framework-specific, Layer 2: generic guidance helper, Layer 3: generic fallback sweep)
- `sarcasm-awareness` capability declaration pattern
- `docs/personality-frameworks.md` update with Sarc7 section, cross-reference table, compatibility matrix
- Eval YAML profiles and scenarios for sarcasm differentiation testing

**Out of scope:**
- New DispositionAxis — humor is a personality framework, not a behavioral axis (D2)
- Per-agent dimension overrides — dimensions are read-only on the enum; DispositionValue weight handles per-agent customization (D7)
- Non-sarcastic humor (warmth, wit) — separate vocabulary if needed later (D3)
- Disposition evolution for sarcasm — no theory of sarcasm type evolution exists (no structural transition rules like Jungian shadow/compensation)
- `CasehubCapabilityTerm.SARCASM_AWARENESS` — nice-to-have, not required for the capability declaration pattern

---

## 1. Vocabulary: `Sarc7Term`

### 1.1 Enum Structure

New enum `Sarc7Term` in `casehub-eidos-vocab`, implementing `VocabularyTerm`.

```
@VocabularyMetadata(
    uri = "urn:casehub:vocab:sarc7",
    name = "Sarc7 Sarcasm Types",
    version = "1.0",
    description = "Seven pragmatically defined sarcasm types from the Sarc7 paper
                   (Xiong et al., 2025). Each type carries evaluation dimensions
                   (incongruity, shock value, context dependency, emotional tone)
                   and prompt guidance for LLM rendering."
)
```

URI: `urn:casehub:vocab:sarc7`

### 1.2 Constants

| Constant | value | label | Description |
|---|---|---|---|
| `SELF_DEPRECATING` | `"self-deprecating"` | Self-Deprecating | Humor at one's own expense; disarming, builds rapport through vulnerability |
| `BROODING` | `"brooding"` | Brooding | Dark, moody sarcasm with pessimistic undertone; inward-directed frustration |
| `DEADPAN` | `"deadpan"` | Deadpan | Dry, expressionless delivery; no signal that humor is intended |
| `POLITE` | `"polite"` | Polite | Veiled sarcasm disguised as courtesy; surface-level agreeable |
| `OBNOXIOUS` | `"obnoxious"` | Obnoxious | Aggressive, in-your-face sarcasm designed to provoke |
| `RAGING` | `"raging"` | Raging | Angry, intense sarcasm driven by frustration |
| `MANIC` | `"manic"` | Manic | Frenzied, over-the-top sarcasm with chaotic energy |

### 1.3 Evaluation Dimension Fields

Each constant carries 4 read-only dimension fields (0.0–1.0), derived from the Sarc7 paper's characterization of each type. These are intrinsic properties of the type, not per-agent tuning knobs. Experimentation with different values happens in the eval harness; winning values are baked back into the enum constants.

| Constant | incongruity | shockValue | contextDependency | emotionalTone |
|---|---|---|---|---|
| SELF_DEPRECATING | 0.5 | 0.2 | 0.6 | 0.7 (positive) |
| BROODING | 0.6 | 0.3 | 0.7 | 0.2 (negative) |
| DEADPAN | 0.8 | 0.2 | 0.6 | 0.4 (neutral) |
| POLITE | 0.7 | 0.3 | 0.8 | 0.6 (positive surface) |
| OBNOXIOUS | 0.6 | 0.8 | 0.4 | 0.2 (negative) |
| RAGING | 0.5 | 0.7 | 0.3 | 0.1 (negative) |
| MANIC | 0.7 | 0.6 | 0.4 | 0.5 (variable) |

Fields:
- `incongruity()` — gap between literal and intended meaning
- `shockValue()` — surprise/provocation intensity
- `contextDependency()` — how much context is needed to detect the sarcasm
- `emotionalTone()` — negative (0.0) to positive (1.0)

Note: These values are initial estimates from the paper's qualitative descriptions. The eval harness (§5) will test whether these values, when surfaced in prompt guidance, produce measurably different LLM output. Adjust based on eval results.

### 1.4 Cross-Vocabulary Mappings

Each Sarc7 term maps to ConscientiousnessTerm (axes 1–4) and ThomasKilmannTerm (CONFLICT_MODE) via `axisExactMatch()`. These are editorial mappings — no psychometric research backs them, unlike BigFive-to-Conscientiousness (Costa & McCrae, 1992). They represent reasonable behavioral correlations.

| Sarc7 Term | SOCIAL_ORIENT | RULE_FOLLOWING | RISK_APPETITE | AUTONOMY | CONFLICT_MODE (TK) |
|---|---|---|---|---|---|
| SELF_DEPRECATING | FACILITATIVE | FLEXIBLE | CONSERVATIVE | SEMI_AUTONOMOUS | ACCOMMODATING |
| BROODING | INDEPENDENT | PRINCIPLED | CONSERVATIVE | AUTONOMOUS | AVOIDING |
| DEADPAN | INDEPENDENT | FLEXIBLE | MEASURED | AUTONOMOUS | AVOIDING |
| POLITE | FACILITATIVE | PRINCIPLED | CONSERVATIVE | DIRECTED | ACCOMMODATING |
| OBNOXIOUS | INDEPENDENT | FLEXIBLE | BOLD | AUTONOMOUS | COMPETING |
| RAGING | INDEPENDENT | FLEXIBLE | BOLD | AUTONOMOUS | COMPETING |
| MANIC | COLLABORATIVE | FLEXIBLE | BOLD | AUTONOMOUS | COLLABORATING |

Mapping rationale:
- **Self-deprecating → FACILITATIVE / ACCOMMODATING** — humor at one's own expense is socially lubricating, conflict-avoidant
- **Brooding → INDEPENDENT / AVOIDING** — dark inward sarcasm is socially withdrawn, sidesteps conflict
- **Deadpan → INDEPENDENT / AVOIDING** — expressionless delivery is socially detached, conflict-neutral
- **Polite → FACILITATIVE / DIRECTED / ACCOMMODATING** — veiled sarcasm implies deference to social norms and authority
- **Obnoxious → INDEPENDENT / BOLD / COMPETING** — provocative sarcasm is assertive, risk-tolerant, confrontational
- **Raging → INDEPENDENT / BOLD / COMPETING** — frustration-driven sarcasm is aggressive, assertive
- **Manic → COLLABORATIVE / BOLD / COLLABORATING** — chaotic energy is socially engaging, high-participation, risk-tolerant

### 1.5 Prompt Guidance Methods

Each constant implements `responseStyleGuidance()` and `antiPatternWarning()`. These feed into the render pipeline (§3) to produce tone instructions in the agent's system prompt.

Example for DEADPAN:
- `responseStyleGuidance()`: "Deliver humor through understatement and matter-of-fact delivery. Never signal that you are being humorous. Let the incongruity between your flat tone and the content do the work. State absurd things as though they are obvious facts."
- `antiPatternWarning()`: "Do not use exclamation marks, emoji, 'lol', or any tonal marker that telegraphs humor. Do not explain the joke. Do not break character by acknowledging the humor."

Guidance text for all 7 types will be drafted during implementation and refined by eval results.

### 1.6 Registrar

`Sarc7VocabularyRegistrar` — `@ApplicationScoped` bean implementing `VocabularyRegistrar`, returning `Sarc7Term.class`. Discovered by `CdiVocabularyRegistry` at startup. Follows existing pattern (e.g., `DiscVocabRegistrar`).

---

## 2. Render Pipeline: Hybrid Three-Layer Architecture

### 2.1 Problem

`EidosRenderPipeline.assembleMarkdownCognitiveProfile()` gates on `JUNGIAN_VOCAB_URI`. Sarc7's `responseStyleGuidance()` and `antiPatternWarning()` would never be called without pipeline changes.

### 2.2 Design

Decompose into three layers:

**Layer 1 — Framework-specific structural rendering:**
Private methods on `EidosRenderPipeline`, one per vocabulary that needs custom structural content.

- `assembleJungianCognitiveProfile()` — existing method, unchanged. Handles function stack, dominant/auxiliary pairing, perception style, MBTI derivation, compensation mechanism. Calls the Layer 2 helper for guidance rendering instead of rendering guidance directly.
- `assembleSarc7HumorProfile()` — new method. Renders sarcasm type identification, dimension context (incongruity/shockValue/contextDependency/emotionalTone), and calls the Layer 2 helper with "Your Humor Style" heading.

**Layer 2 — Generic guidance helper:**
`renderGuidanceBlock(StringBuilder sb, VocabularyTerm term, String heading)` — shared helper that renders `responseStyleGuidance()` and `antiPatternWarning()` for any VocabularyTerm. Parameterized section heading ("Your Response Style" for Jungian, "Your Humor Style" for Sarc7).

Refactoring: the current Jungian method's inline guidance rendering is extracted into this helper. The Jungian method calls `renderGuidanceBlock(sb, term, "Your Response Style")` instead of duplicating the pattern.

**Layer 3 — Generic fallback sweep:**
`assembleGenericVocabularyGuidance(StringBuilder sb, descriptor, Set<String> renderedUris)` — after all Layer 1 methods have run, iterates any remaining vocabularies on the descriptor. For each vocabulary not already rendered, checks if its terms implement `responseStyleGuidance()`. If so, renders a plain guidance block with a generic heading. This means a future vocabulary that implements `responseStyleGuidance()` gets its guidance surfaced automatically — zero pipeline changes needed.

### 2.3 Dispatch

In the main render orchestration:

```
if (hasJungianProfile(descriptor))  → assembleJungianCognitiveProfile(sb, ...)
if (hasSarc7Profile(descriptor))    → assembleSarc7HumorProfile(sb, ...)
assembleGenericVocabularyGuidance(sb, descriptor, alreadyRendered)
```

Two if-checks for two vocabularies. A third vocabulary with custom rendering adds one method and one if-check. Migration to a registry or SPI is warranted if we reach 5+ custom renderers.

### 2.4 A2A_CARD Format

The A2A_CARD path (`assembleA2aCard()`) adds a `"humorProfile"` JSON block alongside the existing cognitive profile:

```json
{
  "humorProfile": {
    "framework": "sarc7",
    "type": "deadpan",
    "dimensions": {
      "incongruity": 0.8,
      "shockValue": 0.2,
      "contextDependency": 0.6,
      "emotionalTone": 0.4
    },
    "responseStyle": "...",
    "antiPattern": "..."
  }
}
```

### 2.5 PROSE Format

PROSE format renders the sarcasm profile as a natural-language paragraph integrated into the agent description, using the same `responseStyleGuidance()` and `antiPatternWarning()` content but without structural headings.

---

## 3. Reception: `sarcasm-awareness` Capability

Sarcasm comprehension is modeled as a standalone `AgentCapability` declaration on the agent's descriptor. No new types or SPIs.

```java
AgentCapability.builder()
    .name("sarcasm-awareness")
    .description("Detects and correctly interprets sarcastic intent in user messages")
    .epistemicDomains(Map.of(
        "western-cultural", 0.9,
        "cross-cultural", 0.5
    ))
    .build()
```

The `epistemicDomains` here are genuine knowledge domains — cultural contexts for sarcasm detection confidence. This is semantically correct: "how well does this agent detect sarcasm in Western vs. cross-cultural contexts" is a knowledge-domain question.

Discoverable via `AgentRegistry.find(AgentQuery.builder().capabilityName("sarcasm-awareness").tenancyId(...).build())`.

Optionally, `CasehubCapabilityTerm` can add a `SARCASM_AWARENESS` term for subsumption matching. Not required for the declaration pattern to work.

---

## 4. Documentation: `personality-frameworks.md` Update

The authoritative mapping reference at `docs/personality-frameworks.md` gains:

### 4.1 New Section: §2.5 Sarc7 Sarcasm Types

Following the existing section format (What it models, Scientific validity, Workplace adoption, Vocabulary role, mapping table). Scientific validity: Medium — published at ACL WiNLP 2025 and NeurIPS COLM SoLaR 2025. Annotation grounded in Qasim (2021) linguistic taxonomy.

### 4.2 Cross-Reference Summary Table Update (§5)

New column `Sarc7` added. Rows:
- `socialOrient` → **disposition** (via axisExactMatch)
- `ruleFollowing` → **disposition**
- `riskAppetite` → **disposition**
- `autonomy` → **disposition**
- `conflictMode` → **disposition** (via TK axisExactMatch)
- All other rows → `—`

### 4.3 Framework Compatibility Update (§6)

New entries:

| Pair | Rating | Reasoning |
|---|---|---|
| Sarc7 + Jungian | Additive | Sarcasm style (Sarc7) and cognitive style (Jungian) are orthogonal — both contribute independent signal to agent personality |
| Sarc7 + DISC | Additive | Sarcasm style and behavioral quadrant are orthogonal |
| Sarc7 + Belbin | Additive | Sarcasm style and team role are orthogonal |
| Sarc7 + Conscientiousness | Redundant (partial) | Sarc7 maps to Conscientiousness terms via axisExactMatch — using both creates overlapping encodings on the mapped axes |

### 4.4 New Combination Pattern: Sarc7 Profile

```
dispositionVocabulary = "urn:casehub:vocab:sarc7"  // or alongside Jungian
disposition.socialOrient = "deadpan"                // Sarc7 type; axis-resolved
```

Guidance on combining Sarc7 with Jungian profiles — both can be active simultaneously since they render in separate pipeline sections.

---

## 5. Eval Integration

### 5.1 Approach

Use existing eval judges — no new judge types needed initially.

- `TraitExpressionJudge` — assess whether sarcasm style traits are expressed in rendered system prompts
- `PairContrastJudge` — verify that different Sarc7 types produce distinguishable agent output (e.g., DEADPAN vs. OBNOXIOUS given the same scenario)

### 5.2 YAML Profiles

Add Sarc7-specific eval YAML agent profiles in `eval/src/test/resources/`:
- One profile per Sarc7 type (7 profiles)
- At least 2 combined profiles (Jungian + Sarc7) to test interaction effects

### 5.3 Protocol Compliance

The `disposition-axis-string-boundary` protocol applies: if eval judges iterate Sarc7 dimensions or types, the constants must match exactly. Use `Sarc7Term.values()` or explicit lists — never hardcoded strings.

### 5.4 Key Experimental Questions

The eval harness should answer:
1. Do LLMs produce detectably different output for different Sarc7 types?
2. Does the structured vocabulary add differentiation beyond "be sarcastic" in plain English?
3. Do the Sarc7 evaluation dimensions (when surfaced in guidance text) produce finer-grained control?
4. Does combining Jungian + Sarc7 produce coherent personality expression or conflicting signals?

---

## 6. Disposition Evolution

Sarc7 terms do **not** participate in `DispositionEvolution` or `DispositionHealth` initially. The Jungian framework has structural rules for valid personality transitions (shadow activation, dominant-auxiliary swap, over-reinforcement). Sarc7 has no equivalent theory — there is no academic model of sarcasm type evolution (e.g., "what does it mean for an agent's sarcasm to shift from POLITE to RAGING?").

If a theory of sarcasm evolution emerges from eval results or future research, the `DispositionSignalStore` and `DispositionEvolution` SPIs are available. No structural changes needed.

---

## 7. Deliverables Summary

| Deliverable | Module | New/Modified |
|---|---|---|
| `Sarc7Term` enum | casehub-eidos-vocab | New |
| `Sarc7VocabularyRegistrar` | casehub-eidos-vocab | New |
| `assembleSarc7HumorProfile()` | casehub-eidos (runtime, renderer) | New |
| `renderGuidanceBlock()` helper | casehub-eidos (runtime, renderer) | New (extracted from Jungian method) |
| `assembleGenericVocabularyGuidance()` | casehub-eidos (runtime, renderer) | New |
| Refactor `assembleMarkdownCognitiveProfile()` | casehub-eidos (runtime, renderer) | Modified (extract guidance to helper) |
| A2A_CARD `humorProfile` block | casehub-eidos (runtime, renderer) | Modified |
| `personality-frameworks.md` | docs | Modified |
| Eval YAML profiles (7+ Sarc7 profiles) | casehub-eidos-eval | New |
| Eval scenarios for sarcasm differentiation | casehub-eidos-eval | New |

---

## Decisions Reference

See `decisions.md` in this directory for all 9 captured decisions with alternatives, rationale, trade-offs, and review revisions.
