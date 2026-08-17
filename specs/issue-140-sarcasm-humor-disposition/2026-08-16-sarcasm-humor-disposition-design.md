# Structured Sarcasm/Humor Dimensions in Disposition Model

**Issue:** casehubio/eidos#140
**Date:** 2026-08-16
**Status:** Draft

## Summary

Extend the Eidos disposition model with structured sarcasm dimensions per agent, grounded in the Sarc7 paper (Xiong et al., 2025). Two new fields on core API records — `styleVocabulary` on `AgentDescriptor` and `styleProfile` on `AgentDisposition` — provide a clean second personality profile slot for communication style vocabularies alongside the behavioral `dispositionProfile`. A new `Sarc7Term` vocabulary enum in `casehub-eidos-vocab` provides 7 sarcasm types with prompt guidance methods, read-only evaluation dimension metadata, and reference cross-vocabulary mappings. The render pipeline gains a hybrid three-layer architecture to surface vocabulary-provided guidance generically while preserving framework-specific rendering fidelity. Reception-side sarcasm awareness is modeled as a standalone `AgentCapability`.

## Academic Reference

**Sarc7: Evaluating Sarcasm Detection and Generation with Seven Types and Emotion-Informed Techniques**
Xiong, Gao, Jeong, Fu, O'Brien, Sharma, Zhu — ACL WiNLP 2025, NeurIPS COLM SoLaR 2025.
arXiv: 2506.00658

Seven pragmatically defined sarcasm types (self-deprecating, brooding, deadpan, polite, obnoxious, raging, manic) with four evaluation dimensions (incongruity, shock value, context dependency, emotional tone). Emotion-based prompting yields best macro-F1 (0.3664, Gemini 2.5) for classification; structured prompts improve subtype alignment by 38.5% for generation (Claude 3.5 Sonnet).

## Scope

**In scope:**
- `styleVocabulary` field on `AgentDescriptor` + `styleProfile` field on `AgentDisposition` — new core API fields for communication style vocabularies
- `Sarc7Term` vocabulary enum with 7 constants, reference cross-vocabulary mappings, responseStyleGuidance(), antiPatternWarning(), and read-only Sarc7 dimension fields
- `Sarc7VocabularyRegistrar` CDI bean
- Render pipeline hybrid refactor (Layer 1: framework-specific, Layer 2: generic guidance helper, Layer 3: generic fallback sweep)
- `sarcasm-awareness` capability declaration pattern
- `docs/personality-frameworks.md` update with Sarc7 section, cross-reference table, compatibility matrix
- Eval YAML profiles and scenarios for sarcasm differentiation testing

**Out of scope:**
- New DispositionAxis — humor is a communication style, not a behavioral axis (D2)
- Axis auto-derivation from `styleProfile` — DescriptorCollector ignores styleProfile; cross-mappings exist on the enum for reference only (D5 revised)
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

Sarc7 types live in `styleProfile` on `AgentDisposition`, NOT on the 5 behavioral axes. An agent declares `styleVocabulary="urn:casehub:vocab:sarc7"` and populates `styleProfile` with one or more weighted Sarc7 terms (e.g., `[DispositionValue("deadpan", 0.8), DispositionValue("brooding", 0.2)]`). The behavioral disposition axes (`socialOrient`, `ruleFollowing`, etc.) are populated independently — typically from a Jungian or DISC profile. `DescriptorCollector` does NOT auto-derive axes from `styleProfile`.

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

Note: These values are initial estimates from the paper's qualitative descriptions. Dimension values are **eval-only metadata** — they are NOT rendered directly in the system prompt. The `responseStyleGuidance()` text (§1.5) already encodes the dimensional profile narratively (e.g., DEADPAN guidance emphasizes incongruity without stating "incongruity=0.8"). The numeric values serve the eval harness (§6) for scoring and comparison, and the A2A_CARD format for machine-readable metadata. Adjust based on eval results.

### 1.4 Cross-Vocabulary Mappings

Each Sarc7 term implements `axisExactMatch()` to ConscientiousnessTerm (axes 1–4) and ThomasKilmannTerm (CONFLICT_MODE). These are **reference mappings** — they document correlations between sarcasm style and behavioral tendencies but are NOT used for axis auto-derivation. `DescriptorCollector` ignores `styleProfile` entirely; these mappings are available for explicit consumer use (consistency validation, personality-frameworks.md documentation) only.

The mappings are editorial — no psychometric research backs them, unlike BigFive-to-Conscientiousness (Costa & McCrae, 1992). They represent reasonable but debatable behavioral correlations.

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

## 2. Core API: `styleProfile` / `styleVocabulary`

### 2.1 New Fields

Two new fields introduce a second personality profile slot for communication style vocabularies:

```
AgentDescriptor:
  + styleVocabulary: String    // vocabulary URI for the style profile (e.g., "urn:casehub:vocab:sarc7")

AgentDisposition:
  + styleProfile: List<DispositionValue>   // communication style terms with weights
```

This parallels the existing behavioral disposition structure:

| Concern | Vocabulary URI field | Profile data field | Auto-derives axes? |
|---|---|---|---|
| Behavioral disposition | `dispositionVocabulary` | `dispositionProfile` | Yes |
| Communication style | `styleVocabulary` | `styleProfile` | **No** |

### 3.2 Design Properties

- `styleProfile` uses `List<DispositionValue>` for consistency with `dispositionProfile`. Enables multi-term blending: `DEADPAN(0.7) + BROODING(0.3)` — "primarily deadpan with brooding undertones."
- `DescriptorCollector` ignores `styleProfile` — no axis auto-derivation. Cross-mappings on the enum exist for reference/consistency only.
- Jungian + Sarc7 coexist naturally: `dispositionVocabulary="urn:casehub:vocab:jungian"` + `styleVocabulary="urn:casehub:vocab:sarc7"`.
- `styleVocabulary` resolution: `styleVocabulary` → `domainVocabulary` (fallback). No `slotVocabulary`/`dispositionVocabulary` fallback — communication style is a distinct concern.
- `impliesSupervision()`: no Sarc7 term implies supervision. Sarcasm style doesn't determine autonomy.

### 2.3 Builder / YAML / JPA

- `AgentDisposition.Builder` gains `styleProfile(DispositionValue...)` and `styleProfile(List<DispositionValue>)` methods.
- `AgentDescriptor` builder gains `styleVocabulary(String)`.
- YAML descriptor format gains `styleVocabulary` and `styleProfile` fields on the disposition config.
- JPA: new `style_profile` column on the disposition entity (JSON array of term+weight pairs). No Flyway migration needed — no deployed instances.

### 2.4 Worked Example

```
// Jungian cognitive profile + Sarc7 communication style
dispositionVocabulary = "urn:casehub:vocab:jungian"
dispositionProfile    = [{ti, 0.45}, {ne, 0.20}, {si, 0.10}, {fe, 0.08}]
styleVocabulary       = "urn:casehub:vocab:sarc7"
styleProfile          = [{deadpan, 0.8}]
```

Both render in the system prompt: Jungian via `assembleJungianCognitiveProfile()`, Sarc7 via `assembleSarc7HumorProfile()`. No conflict — separate profile fields, separate vocabulary URIs, separate pipeline sections.

---

## 3. Render Pipeline: Hybrid Three-Layer Architecture

### 3.1 Problem

`EidosRenderPipeline.assembleMarkdownCognitiveProfile()` gates on `JUNGIAN_VOCAB_URI`. Sarc7's `responseStyleGuidance()` and `antiPatternWarning()` would never be called without pipeline changes.

### 3.2 Design

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

### 3.3 Dispatch

In the main render orchestration:

```
if (hasJungianProfile(descriptor))  → assembleJungianCognitiveProfile(sb, ...)
if (hasSarc7Profile(descriptor))    → assembleSarc7HumorProfile(sb, ...)
assembleGenericVocabularyGuidance(sb, descriptor, alreadyRendered)
```

Detection: `hasSarc7Profile(descriptor)` returns true when `descriptor.styleVocabulary()` equals `urn:casehub:vocab:sarc7` and `descriptor.disposition().styleProfile()` is non-empty. Clean and unambiguous — no overlap with Jungian detection.

Two if-checks for two vocabularies. A third vocabulary with custom rendering adds one method and one if-check. Migration to a registry or SPI is warranted if we reach 5+ custom renderers.

### 3.4 A2A_CARD Format

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

### 3.5 PROSE Format

PROSE format renders the sarcasm profile as a natural-language paragraph integrated into the agent description, using the same `responseStyleGuidance()` and `antiPatternWarning()` content but without structural headings.

---

## 4. Reception: `sarcasm-awareness` Capability

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

## 5. Documentation: `personality-frameworks.md` Update

The authoritative mapping reference at `docs/personality-frameworks.md` gains:

### 6.1 New Section: §2.5 Sarc7 Sarcasm Types

Following the existing section format (What it models, Scientific validity, Workplace adoption, Vocabulary role, mapping table). Scientific validity: Medium — published at ACL WiNLP 2025 and NeurIPS COLM SoLaR 2025. Annotation grounded in Qasim (2021) linguistic taxonomy.

### 6.2 Cross-Reference Summary Table Update (§5)

New column `Sarc7` added. Rows:
- `socialOrient` → **disposition** (via axisExactMatch)
- `ruleFollowing` → **disposition**
- `riskAppetite` → **disposition**
- `autonomy` → **disposition**
- `conflictMode` → **disposition** (via TK axisExactMatch)
- All other rows → `—`

### 6.3 Framework Compatibility Update (§6)

New entries:

| Pair | Rating | Reasoning |
|---|---|---|
| Sarc7 + Jungian | Additive | Sarcasm style (Sarc7) and cognitive style (Jungian) are orthogonal — both contribute independent signal to agent personality |
| Sarc7 + DISC | Additive | Sarcasm style and behavioral quadrant are orthogonal |
| Sarc7 + Belbin | Additive | Sarcasm style and team role are orthogonal |
| Sarc7 + Conscientiousness | Redundant (partial) | Sarc7 maps to Conscientiousness terms via axisExactMatch — using both creates overlapping encodings on the mapped axes |

### 6.4 New Combination Pattern: Sarc7 Profile

```
dispositionVocabulary = "urn:casehub:vocab:sarc7"  // or alongside Jungian
disposition.socialOrient = "deadpan"                // Sarc7 type; axis-resolved
```

Guidance on combining Sarc7 with Jungian profiles — both can be active simultaneously since they render in separate pipeline sections.

---

## 6. Eval Integration

### 6.1 Approach

Use existing eval judges — no new judge types needed initially.

- `TraitExpressionJudge` — assess whether sarcasm style traits are expressed in rendered system prompts
- `PairContrastJudge` — verify that different Sarc7 types produce distinguishable agent output (e.g., DEADPAN vs. OBNOXIOUS given the same scenario)

### 6.2 YAML Profiles

Add Sarc7-specific eval YAML agent profiles in `eval/src/test/resources/`:
- One profile per Sarc7 type (7 profiles)
- At least 2 combined profiles (Jungian + Sarc7) to test interaction effects

### 6.3 Protocol Compliance

The `disposition-axis-string-boundary` protocol applies: if eval judges iterate Sarc7 dimensions or types, the constants must match exactly. Use `Sarc7Term.values()` or explicit lists — never hardcoded strings.

### 6.4 Key Experimental Questions

The eval harness should answer:
1. Do LLMs produce detectably different output for different Sarc7 types?
2. Does the structured vocabulary add differentiation beyond "be sarcastic" in plain English?
3. Do the Sarc7 evaluation dimensions (when surfaced in guidance text) produce finer-grained control?
4. Does combining Jungian + Sarc7 produce coherent personality expression or conflicting signals?

---

## 7. Disposition Evolution

Sarc7 terms do **not** participate in `DispositionEvolution` or `DispositionHealth` initially. The Jungian framework has structural rules for valid personality transitions (shadow activation, dominant-auxiliary swap, over-reinforcement). Sarc7 has no equivalent theory — there is no academic model of sarcasm type evolution (e.g., "what does it mean for an agent's sarcasm to shift from POLITE to RAGING?").

If a theory of sarcasm evolution emerges from eval results or future research, the `DispositionSignalStore` and `DispositionEvolution` SPIs are available. No structural changes needed.

---

## 8. Deliverables Summary

| Deliverable | Module | New/Modified |
|---|---|---|
| `styleVocabulary` field on `AgentDescriptor` | casehub-eidos-api | Modified |
| `styleProfile` field on `AgentDisposition` + Builder | casehub-eidos-api | Modified |
| `Sarc7Term` enum | casehub-eidos-vocab | New |
| `Sarc7VocabularyRegistrar` | casehub-eidos-vocab | New |
| JPA entity updates for `styleProfile` | casehub-eidos (runtime, registry/jpa) | Modified |
| YAML descriptor support for `styleVocabulary`/`styleProfile` | casehub-eidos (runtime, registrar) | Modified |
| `DescriptorCollector` — ignore `styleProfile` for axis derivation | casehub-eidos (runtime, registrar) | Verified (no change needed if styleProfile isn't in dispositionProfile) |
| `assembleSarc7HumorProfile()` | casehub-eidos (runtime, renderer) | New |
| `renderGuidanceBlock()` helper | casehub-eidos (runtime, renderer) | New (extracted from Jungian method) |
| `assembleGenericVocabularyGuidance()` | casehub-eidos (runtime, renderer) | New |
| Refactor `assembleMarkdownCognitiveProfile()` | casehub-eidos (runtime, renderer) | Modified (extract guidance to helper) |
| A2A_CARD `styleProfile` block | casehub-eidos (runtime, renderer) | Modified |
| `personality-frameworks.md` | docs | Modified |
| Eval YAML profiles (7+ Sarc7 profiles) | casehub-eidos-eval | New |
| Eval scenarios for sarcasm differentiation | casehub-eidos-eval | New |

---

## Decisions Reference

See `decisions.md` in this directory for all 9 captured decisions with alternatives, rationale, trade-offs, and review revisions.
