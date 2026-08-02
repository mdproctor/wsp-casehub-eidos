# Jungian Personality Framework — Design Spec

> **Issue:** eidos#107 (epic)
> **Covers:** #108, #109, #110, #111, #112, #113, #114, #115, #116, #117
> **Date:** 2026-07-28
> **Status:** Approved

## Summary

Integrates Jungian cognitive functions into eidos as a vocabulary-grounded,
weighted disposition system. Three layers: vocabulary (8 functions, 16 MBTI
types, cross-vocabulary equivalences), API (weighted disposition profiles,
disposition profiles, DispositionHealth SPI, personality evolution),
and eval (MBTI questionnaire alignment, TAA/PSA metrics).

Based on the JPAF paper (arXiv:2601.10025) which demonstrates 100% MBTI
alignment across GPT-4, Llama, and Qwen using function-level specification.
Eidos becomes the first production system to integrate this into a structured
agent identity framework with vocabulary grounding, persistence, multi-format
rendering, and health probing.

## Research References

- [JPAF Paper](https://arxiv.org/abs/2601.10025) — Structured Personality Control and Adaptation for LLM Agents
- [JPAF GitHub](https://github.com/agent-topia/evolving_personality) — Python reference implementation
- [Geometry of Personality](https://arxiv.org/abs/2607.20803) — Independent validation via activation steering (July 2026)

---

## §1 Vocabulary Layer (#108, #109, #110)

### §1.1 JungianFunctionTerm (#108)

New enum in `casehub-eidos-vocab` — `urn:casehub:vocab:jungian`

Eight constants representing the Jungian cognitive functions:

| Constant | Value | Label | Category | Attitude | Description |
|----------|-------|-------|----------|----------|-------------|
| TI | `ti` | Introverted Thinking | JUDGING | INTROVERTED | Builds internal logical frameworks; analytical, precision-focused |
| TE | `te` | Extraverted Thinking | JUDGING | EXTRAVERTED | Applies logical organization externally; systematic, efficiency-oriented |
| FI | `fi` | Introverted Feeling | JUDGING | INTROVERTED | Evaluates through deeply held personal values; authentic, principled |
| FE | `fe` | Extraverted Feeling | JUDGING | EXTRAVERTED | Harmonizes group values and social dynamics; attentive to others |
| SI | `si` | Introverted Sensation | PERCEIVING | INTROVERTED | Draws on internalized sensory impressions and past experience |
| SE | `se` | Extraverted Sensation | PERCEIVING | EXTRAVERTED | Focuses on immediate sensory data; concrete, present-moment |
| NI | `ni` | Introverted Intuition | PERCEIVING | INTROVERTED | Synthesizes internal patterns into singular insights; foresight |
| NE | `ne` | Extraverted Intuition | PERCEIVING | EXTRAVERTED | Explores external patterns, possibilities, and connections |

#### Vocabulary methods beyond VocabularyTerm

```java
public JungianFunctionTerm shadow()
```
Returns the same-category, opposite-attitude function: Ti↔Te, Fi↔Fe, Si↔Se, Ni↔Ne.
Used by DispositionHealth for shadow takeover detection (JPAF Reflection 2/3).

```java
public FunctionCategory category()    // JUDGING or PERCEIVING
public FunctionAttitude attitude()    // INTROVERTED or EXTRAVERTED
```
Structural metadata for Jungian rules — auxiliary must be opposite category
from dominant, typically opposite attitude.

```java
public List<JungianFunctionTerm> compatibleAuxiliaries()
```
Valid auxiliary functions for this function as dominant. Encodes the Jungian
structural rule that JPAF hardcodes in `main_to_aux`.

#### Cross-vocabulary mapping via axisExactMatch

Each function maps to ConscientiousnessTerm and ThomasKilmannTerm per axis:

| Function | socialOrient | ruleFollowing | riskAppetite | autonomy | conflictMode |
|----------|-------------|---------------|--------------|----------|--------------|
| Ti | independent | principled | measured | autonomous | avoiding |
| Te | independent | strict | measured | semi-autonomous | competing |
| Fi | independent | principled | conservative | autonomous | accommodating |
| Fe | facilitative | flexible | conservative | semi-autonomous | collaborating |
| Si | collaborative | strict | conservative | directed | avoiding |
| Se | collaborative | flexible | bold | semi-autonomous | competing |
| Ni | independent | principled | bold | autonomous | avoiding |
| Ne | collaborative | flexible | bold | semi-autonomous | collaborating |

These mappings follow the pattern established by `DiscTerm.axisExactMatch`.

#### Weight tier constants

```java
public static final double DOMINANT_MIN = 0.31;
public static final double DOMINANT_MAX = 1.00;
public static final double AUXILIARY_MIN = 0.06;
public static final double AUXILIARY_MAX = 0.30;
public static final double UNDIFFERENTIATED_MAX = 0.06;
public static final double REINFORCEMENT_DELTA = 0.06;
public static final double DECAY_FACTOR = 0.20;
```

Derived from JPAF's parameters (B=0.06, A=0.30). These constants are
reference defaults. `DefaultDispositionHealth` reads thresholds from
`PreferenceProvider` at probe time, falling back to these constants:

| Constant | PreferenceProvider key | Default |
|----------|----------------------|---------|
| DOMINANT_MIN | `eidos.jungian.dominant.min` | 0.31 |
| DOMINANT_MAX | `eidos.jungian.dominant.max` | 1.00 |
| AUXILIARY_MIN | `eidos.jungian.auxiliary.min` | 0.06 |
| AUXILIARY_MAX | `eidos.jungian.auxiliary.max` | 0.30 |
| UNDIFFERENTIATED_MAX | `eidos.jungian.undifferentiated.max` | 0.06 |
| REINFORCEMENT_DELTA | `eidos.jungian.reinforcement.delta` | 0.06 |
| DECAY_FACTOR | `eidos.jungian.decay.factor` | 0.20 |

#### Enums for category and attitude

```java
public enum FunctionCategory { JUDGING, PERCEIVING }
public enum FunctionAttitude { INTROVERTED, EXTRAVERTED }
```

These live in `casehub-eidos-vocab` alongside `JungianFunctionTerm` —
they are Jungian-specific structural metadata, not referenced by the
`VocabularyTerm` interface.

### §1.2 MbtiTypeTerm (#110)

New enum in `casehub-eidos-vocab` — `urn:casehub:vocab:mbti`

Sixteen constants, each specializing its dominant + auxiliary JungianFunctionTerms:

| Type | Value | Dominant | Auxiliary | specializes() |
|------|-------|----------|-----------|---------------|
| INTJ | `intj` | Ni | Te | [NI, TE] |
| INTP | `intp` | Ti | Ne | [TI, NE] |
| ENTJ | `entj` | Te | Ni | [TE, NI] |
| ENTP | `entp` | Ne | Ti | [NE, TI] |
| INFJ | `infj` | Ni | Fe | [NI, FE] |
| INFP | `infp` | Fi | Ne | [FI, NE] |
| ENFJ | `enfj` | Fe | Ni | [FE, NI] |
| ENFP | `enfp` | Ne | Fi | [NE, FI] |
| ISTJ | `istj` | Si | Te | [SI, TE] |
| ISTP | `istp` | Ti | Se | [TI, SE] |
| ESTJ | `estj` | Te | Si | [TE, SI] |
| ESTP | `estp` | Se | Ti | [SE, TI] |
| ISFJ | `isfj` | Si | Fe | [SI, FE] |
| ISFP | `isfp` | Fi | Se | [FI, SE] |
| ESFJ | `esfj` | Fe | Si | [FE, SI] |
| ESFP | `esfp` | Se | Fi | [SE, FI] |

Each type provides a default weight profile method:

```java
public List<DispositionValue> defaultProfile()
```

Returns the 8-function weight distribution with JPAF parameters:
dominant at ~0.35, auxiliary at ~0.20, remaining 6 distributed
across ~0.45 with slight variations. Convenience for agents that
want to declare "I'm an INTP" without manually specifying 8 weights.

### §1.3 Cross-Vocabulary Equivalences (#109)

Built via `axisExactMatch` (proven pattern from DISC):

**Jung → Conscientiousness:** Per-axis mapping for all 8 functions (table in §1.1).

**Jung → Thomas-Kilmann:** CONFLICT_MODE axis mapping for all 8 functions.

**Jung ↔ DISC:** Approximate mappings (documented as soft, not exact):
- Ti ≈ Conscientiousness-DISC (analytical, systematic)
- Fe ≈ Influence (people-focused, collaborative)
- Se ≈ Dominance (action-oriented, results-driven)
- Si ≈ Steadiness (reliable, pattern-following)

The ≈ notation denotes shared behavioral space, not identical disposition
profiles. Ti and DISC-C both produce analytical, systematic behavior but
differ on specific axis mappings (e.g., Ti → principled vs. C → strict
on ruleFollowing) because Jungian functions describe cognitive orientation
while DISC describes behavioral quadrants. These are not implemented as
`axisExactMatch` entries — they are reference annotations only.

**MBTI → all:** Resolved through the cognitive profile mechanism, not
direct vocabulary resolution. `MbtiTypeTerm.defaultProfile()` returns
a full 8-function weight distribution; an agent declaring an MBTI type calls
`defaultProfile()` to populate `dispositionProfile`, which then auto-derives
axis values via §2.4. The `specializes()` relationship supports vocabulary
hierarchy queries (subsumption, matching) but `CdiVocabularyRegistry.equivalentValues()`
does not traverse `specializes()` edges — it uses `exactMatch()` only.
MBTI types are not usable as direct `dispositionVocabulary` values for
axis resolution; they require cognitive profile population.

**Jung → Belbin:** Softer relationship — function stacks suggest roles but
don't determine them. Ti-Ne (analytical + possibility) suggests Monitor
Evaluator; Fe-Ni (harmony + vision) suggests Co-ordinator. These are
documented as reference guidance in personality-frameworks.md, not
implemented as axisExactMatch (which would imply a precision this mapping
doesn't have).

---

## §2 API Evolution (#111, #112, #115, #116)

### §2.1 DispositionValue

New record in `casehub-eidos-api`:

```java
public record DispositionValue(String term, double weight) {
    public DispositionValue {
        if (term == null || term.isBlank()) throw new IllegalArgumentException("term required");
        if (Double.isNaN(weight) || weight < 0.0 || weight > 1.0)
            throw new IllegalArgumentException("weight must be 0.0–1.0");
    }
}
```

The universal primitive for weighted disposition — used for both axis values
and cognitive profile entries.

### §2.2 Evolved AgentDisposition (#111)

```java
public record AgentDisposition(
    List<DispositionValue> socialOrient,
    List<DispositionValue> ruleFollowing,
    List<DispositionValue> riskAppetite,
    List<DispositionValue> autonomy,
    List<DispositionValue> conflictMode,
    boolean delegation,
    List<DispositionValue> dispositionProfile
) {}
```

**Axes** — multi-valued weighted terms. Each axis carries one or more
vocabulary term values with weights. Single-value with weight 1.0 is
equivalent to the current String model.

**dispositionProfile** — optional list of weighted terms from a cognitive
vocabulary (JungianFunctionTerm, or future frameworks). When populated, the
profile is the authoritative disposition specification; axes can be
auto-derived from it via cross-vocabulary projection.

**Builder ergonomics:**
```java
// Simple (unchanged feel):
builder.socialOrient("independent")
    // → List.of(new DispositionValue("independent", 1.0))

// Weighted axis:
builder.socialOrient(dv("independent", 0.7), dv("collaborative", 0.3))

// Jungian profile (axes auto-derived at registration):
builder.dispositionProfile(
    dv("ti", 0.45), dv("ne", 0.20), dv("fi", 0.08), dv("fe", 0.04),
    dv("si", 0.06), dv("se", 0.05), dv("ni", 0.07), dv("te", 0.05)
)
```

#### Cognitive Profile Validation Rules

**Construction time** (API tier, vocabulary-agnostic):
- Each `DispositionValue`: weight must be 0.0–1.0 (NaN-guarded), term non-blank
- Empty `dispositionProfile` list is valid (no cognitive profile declared)
- No weight-sum constraint at construction — normalization happens at
  auto-derivation time (§2.4 step 5). Partial profiles (e.g., dominant
  and auxiliary only) are valid inputs.

**Probe time** (`DefaultDispositionHealth`, vocabulary-aware):
- Weight-sum deviation from 1.0 → `Drifted` status (not a construction error)
- Dominant-auxiliary structural rules (opposite categories per Jungian
  theory) validated via `JungianFunctionTerm.compatibleAuxiliaries()`
- Violations are health warnings, not construction errors — consistent
  with `CapabilityHealth`, where descriptors can be constructed freely
  and health probing detects issues

### §2.3 Dominant-Auxiliary Designation (#112)

Implicit from weights — no separate fields. The two highest-weighted terms
in `dispositionProfile` are dominant and auxiliary. The vocabulary metadata
(`shadow()`, `category()`, `attitude()`, `compatibleAuxiliaries()`) provides
the structural rules for validation and evolution.

### §2.4 Auto-Derivation: Cognitive Profile → Weighted Axes

At registration time (in `DescriptorCollector`/`AgentDescriptorBootstrap`),
when `dispositionProfile` is populated and axis fields are empty:

1. For each function in the profile, resolve `axisExactMatch` to get the
   implied ConscientiousnessTerm/ThomasKilmannTerm per axis
2. Weight that term by the function's weight
3. Aggregate weighted terms per axis (same term from multiple functions →
   sum weights)
4. Normalize per axis so weights sum to 1.0
5. Populate the axis fields with the derived values

Example — INTP (Ti:0.45, Ne:0.20, Fi:0.08, Fe:0.04, Si:0.06, Se:0.05, Ni:0.07, Te:0.05):

Ti(0.45)+Te(0.05)+Fi(0.08)+Ni(0.07) → `independent` on socialOrient
Ne(0.20)+Se(0.05)+Si(0.06) → `collaborative` on socialOrient
Fe(0.04) → `facilitative` on socialOrient

Result: `socialOrient: [("independent", 0.65), ("collaborative", 0.31), ("facilitative", 0.04)]`

The projection is the bridge between frameworks — queries by axis
("find agents with socialOrient near independent") work against
Jungian-profiled agents through the derived axes.

**Axis vocabulary tagging:** Auto-derivation also populates the
descriptor's `axisVocabularies` map for all derived axes, using the
vocabulary URIs of the target terms:

```
axisVocabularies = {
    SOCIAL_ORIENTATION: "urn:casehub:vocab:conscientiousness",
    RULE_FOLLOWING:     "urn:casehub:vocab:conscientiousness",
    RISK_APPETITE:      "urn:casehub:vocab:conscientiousness",
    AUTONOMY:           "urn:casehub:vocab:conscientiousness",
    CONFLICT_MODE:      "urn:casehub:vocab:thomas-kilmann"
}
```

This ensures the rendering pipeline resolves derived axis values against
the correct vocabulary (ConscientiousnessTerm / ThomasKilmannTerm), not
the agent's `dispositionVocabulary` (which is Jungian). This follows the
existing `axisVocabularies` override mechanism documented in
`personality-frameworks.md`.

**When both are specified:** Explicit axis values take precedence over
auto-derived values. This allows a Jungian-profiled agent to override
specific axes when the automatic projection doesn't capture the intended
nuance (e.g., a Ti-dominant agent that is more collaborative than Ti
typically implies).

### §2.5 Disposition Signal Store (#115)

New SPI in `casehub-eidos-api`, purpose-built for disposition function
activation tracking with its own lifecycle semantics:

```java
public interface DispositionSignalStore {
    void recordActivation(String agentId, String tenancyId,
                          String functionTerm);
    Map<String, Integer> activationCounts(String agentId,
                                          String tenancyId);
    void decay(String agentId, String tenancyId, double decayFactor);
    void clear(String agentId, String tenancyId);
}
```

**Why not BehavioralSignalStore?** The existing `BehavioralSignalStore`
contract requires `capabilityName` to be a declared capability name
(Javadoc on `record()`). Disposition activation signals are not
capability-scoped. Additionally, the lifecycle semantics differ:
capability signals are TTL-managed (windowed compliance monitoring);
activation counts accumulate without expiry until explicitly decayed
by evolution or cleared post-transition.

- `recordActivation` — records a single function activation event
- `activationCounts` — returns function → count mapping (cumulative, no TTL)
- `decay` — multiplies all activation counts by `(1 - decayFactor)`,
  truncated to integer. `DECAY_FACTOR = 0.20` means 20% of accumulated
  signal is removed (80% retained), matching JPAF's "80% decay" semantics.
  Formula: `newCount = (int)(count * (1 - decayFactor))`. Integer
  truncation is intentional — single activations (`count = 1`) decay to 0,
  providing a natural floor that prevents statistically insignificant
  signals from persisting indefinitely.
- `clear` — resets all activation data (post-evolution cleanup)

Accumulated activation counts ARE the JPAF change_history — durable,
queryable, with disposition-appropriate lifecycle management.

Effective weight at probe time:
```
effectiveWeight(fn) = baseWeight(fn) + (activationCount(fn) × REINFORCEMENT_DELTA)
// then normalize all effective weights to sum 1.0
```

### §2.6 DispositionHealth SPI (#116)

New SPI in `casehub-eidos-api`, paralleling CapabilityHealth:

```java
public interface DispositionHealth {
    DispositionStatus probe(AgentDescriptor descriptor, ProbeContext context);
}

public sealed interface DispositionStatus
    permits DispositionStatus.Aligned,
            DispositionStatus.Drifted,
            DispositionStatus.EvolutionPending {

    record Aligned(Map<String, Double> effectiveWeights)
        implements DispositionStatus {}

    record Drifted(
        Map<String, Double> effectiveWeights,
        String mostActivated,
        double driftMagnitude)
        implements DispositionStatus {}
    // driftMagnitude: L2 (Euclidean) distance between base and effective
    // weight vectors: sqrt(Σ(effective[i] - base[i])²). Consumers compare
    // against a threshold to decide whether to invoke evolution evaluation.

    record EvolutionPending(
        EvolutionType type,
        String candidateFunction,
        Map<String, Double> effectiveWeights)
        implements DispositionStatus {}
}

public interface EvolutionType {
    String name();
}
```

Framework-specific evolution types live in vocab:

```java
// In casehub-eidos-vocab:
public enum JungianEvolutionType implements EvolutionType {
    DOMINANT_AUXILIARY_SWAP,       // JPAF Reflection 1
    DOMINANT_REPLACEMENT,          // JPAF Reflection 2 (shadow takeover)
    AUXILIARY_REPLACEMENT,         // JPAF Reflection 3
    STRUCTURAL_REORGANIZATION     // JPAF Reflection 4
}
```

DefaultDispositionHealth probe logic:

| Condition | Status |
|-----------|--------|
| All effective weights within tier bounds | `Aligned` |
| Drift detected, no threshold crossed | `Drifted` |
| Auxiliary effective ≥ dominant base | `EvolutionPending(DOMINANT_AUXILIARY_SWAP)` |
| Shadow of dominant ≥ dominant base | `EvolutionPending(DOMINANT_REPLACEMENT)` |
| Shadow of auxiliary ≥ auxiliary base | `EvolutionPending(AUXILIARY_REPLACEMENT)` |
| Unrelated function ≥ dominant base | `EvolutionPending(STRUCTURAL_REORGANIZATION)` |

Over-reinforcement (dominant effective ≥ 0.5) produces `Drifted` status
with high `driftMagnitude`. The engine decides whether to normalize via
`DispositionSignalStore.decay()` — this is an orchestration concern, not
a probe concern. The probe detects; the engine acts.

### §2.7 Personality Evolution Service

Separates detection (probe) from decision (evolution):

```java
public interface DispositionEvolution {
    EvolutionResult evaluate(
        AgentDescriptor descriptor,
        DispositionStatus.EvolutionPending pending);
}

public sealed interface EvolutionResult
    permits EvolutionResult.Evolved, EvolutionResult.Dampened {

    record Evolved(
        List<DispositionValue> newProfile,
        String previousTypeLabel,
        String newTypeLabel)
        implements EvolutionResult {}

    record Dampened(double decayFactor)
        implements EvolutionResult {}
}
```

- **Evolved:** New weight profile + type transition. Consumer creates a
  new descriptor version. Evolution event recorded in signal store.
- **Dampened:** JPAF's "reflection says no" path. Accumulated signals
  decay by the decay factor (default 0.2). No structural change.

Default implementation uses `ChatModel` for LLM-adjudicated transitions
(matching JPAF's approach). Falls back to rule-based evolution when no
ChatModel is available.

Personality evolution creates a new descriptor version — the old version
is preserved in the registry's history. This connects to the existing
versioning and attestation story.

#### Evolution Orchestration Boundary

This spec designs the **detection** (`DispositionHealth.probe()`) and
**decision** (`DispositionEvolution.evaluate()`) SPIs. The **orchestration**
— who calls `evaluate()`, when, and how the result is applied — is the
engine's responsibility, defined in the companion engine epic for runtime
adaptation.

The boundary contract:
1. The engine calls `DispositionHealth.probe()` alongside
   `CapabilityHealth.probe()` during agent health checks
2. When `EvolutionPending` is returned, the engine decides whether to
   invoke `evaluate()` — not on every probe (debounced or scheduled)
3. The `Evolved` result is applied by the engine creating a new
   descriptor version via the registry
4. Concurrency is handled by the engine's existing descriptor versioning
   (optimistic locking on version number)
5. If `evaluate()` is already in flight for an agent, subsequent
   `EvolutionPending` detections are ignored until the current evaluation
   completes

### §2.8 Future Direction: Contextual Weights

Base weights are declared at registration time (static). Runtime
context-dependent weight modulation is a future concern that this design
accommodates: the `DispositionValue(term, weight)` structure is the right
primitive — contextual weights would be a runtime overlay computed at
render/dispatch time, combining base weights with situational context.
The signal store provides the observation layer; a future
context-aware renderer could adjust weights before prompting.

---

## §3 Rendering

### §3.1 Weighted Axis Rendering

When axes carry multiple weighted values:

**MARKDOWN:**
```markdown
## How You Operate
- Social orientation: primarily independent (0.7), with collaborative tendencies (0.3)
- Rule following: strict
```

Weight 1.0 or sole value renders without a number (identical to today).
Multi-valued or fractional weights use "primarily X, with Y tendencies".

**PROSE:** LLM enrichment receives weighted values, produces natural prose.

**A2A_CARD:** Raw weighted values in JSON:
```json
"disposition": {
  "socialOrient": [{"term": "independent", "weight": 0.7}, {"term": "collaborative", "weight": 0.3}]
}
```

### §3.2 Cognitive Profile Rendering (JPAF-Level Prompting)

When `dispositionProfile` is populated and `dispositionVocabulary` is
`urn:casehub:vocab:jungian`:

**MARKDOWN:**
```markdown
## Cognitive Style

Your personality is structured around Jungian cognitive functions:

**Dominant — Introverted Thinking (Ti):** You build internal logical
frameworks. Analytical, precision-focused, seeking internal consistency.
This is your primary mode of engagement.

**Auxiliary — Extraverted Intuition (Ne):** You explore external patterns,
possibilities, and connections. This complements your analytical core.

When your dominant and auxiliary functions cannot effectively address a
situation, draw on other cognitive functions. Recognize that compensatory
function use produces less controlled but potentially valuable responses.

## How You Operate
- Social orientation: primarily independent (0.65), collaborative (0.35)
- Rule following: principled (0.53), strict (0.25)
...
```

Structure mirrors JPAF's prompt: function identity → compensation
instructions → derived axes as behavioral summary.

**A2A_CARD:**
```json
"dispositionProfile": {
  "vocabulary": "urn:casehub:vocab:jungian",
  "functions": [
    {"term": "ti", "weight": 0.45, "role": "dominant"},
    {"term": "ne", "weight": 0.20, "role": "auxiliary"},
    ...
  ],
  "derivedMbtiType": "INTP"
}
```

`role` is derived at render time from weight ordering, not stored.

### §3.3 Rendering Dispatch

| Condition | Path |
|-----------|------|
| dispositionProfile empty, axes populated | Standard axis rendering + weight support |
| dispositionProfile populated, vocab is jungian | Function-level JPAF rendering + derived axes |
| dispositionProfile populated, other vocab | Generic profile rendering (future extensibility) |

No new RenderFormat values — vocabulary drives template selection within
each existing format.

---

## §4 Eval (#113, #117)

### §4.1 MbtiAlignmentJudge (#113)

Validates rendered agent responses align with expected MBTI type.

**Process:**
1. Render agent descriptor with cognitive profile via SystemPromptRenderer
2. Use rendered prompt as system prompt for test ChatModel
3. Administer MBTI-70 questionnaire items
4. Score per-dimension accuracy (E/I, S/N, T/F, J/P)

**Output:**
```json
{
  "expectedType": "INTP",
  "dimensions": {
    "EI": {"expected": "I", "accuracy": 0.92},
    "SN": {"expected": "N", "accuracy": 0.88},
    "TF": {"expected": "T", "accuracy": 0.95},
    "JP": {"expected": "P", "accuracy": 0.85}
  },
  "overallAlignment": true
}
```

Overall alignment = true when all four dimensions exceed 50% (JPAF criterion).
DAG metric compares against baseline (raw MBTI label injection).

**Eval profiles:** YAML agent definitions for all 16 MBTI types. Minimum
coverage: one type per dominant function (8 types). Full coverage: all 16.

### §4.2 FunctionActivationJudge (#117 — TAA)

Validates scenario contexts activate the correct cognitive function.

**Process:**
1. Present scenario designed to activate a specific function
2. Agent responds using rendered Jungian prompt (includes compensation instructions)
3. Parse response for function identification
4. Compare activated function against target

**Output:**
```json
{
  "agentType": "INTP",
  "targetFunction": "Fe",
  "scenarioCount": 15,
  "correctActivations": 14,
  "taa": 0.933
}
```

**Scenario corpus:** 8 sets × 3 scenarios × 5 questions = 120 items.
Adapted for software engineering contexts (eidos agents are code/analysis/team
agents). New YAML resources in `eval/src/test/resources/`.

### §4.3 PersonalityEvolutionJudge (#117 — PSA)

Validates personality transitions follow valid Jungian structural rules.

**Process:**
1. Run agent through repeated scenarios targeting a specific function
2. Record FUNCTION_ACTIVATED signals after each scenario
3. Invoke DispositionHealth.probe() for EvolutionPending detection
4. Invoke DispositionEvolution.evaluate() for transition judgment
5. Validate resulting type against Jungian rules

**Output:**
```json
{
  "initialType": "INTP",
  "targetFunction": "Fe",
  "evolutionType": "STRUCTURAL_REORGANIZATION",
  "resultingType": "ENFJ",
  "structurallyValid": true,
  "weightTiersValid": true,
  "psa": 1.0
}
```

**Validation checks:**
- Correct reflection type triggered (swap/replacement/reorganization)?
- Resulting dominant-auxiliary pair structurally valid (opposite categories)?
- Weight tiers respected (dominant [0.31,1.0], auxiliary [0.06,0.30])?

Core coverage: 8 dominant functions × 8 target functions = 64 scenarios.

**Multi-model:** All judges run across Claude, Ollama, and Jlama backends
(existing eval infrastructure).

---

## §5 Schema

Pre-release, no deployed instances — changes go into base migration files.

### New table: disposition_value

| Column | Type | Notes |
|--------|------|-------|
| id | BIGINT PK | |
| descriptor_id | BIGINT FK | → agent_descriptor |
| entry_type | VARCHAR(10) | `AXIS` or `COGNITIVE` |
| axis | VARCHAR(30) | Axis key (for AXIS entries) or `dispositionProfile` constant (for COGNITIVE entries) |
| term | VARCHAR(100) | Vocabulary term value |
| weight | DOUBLE | 0.0–1.0 |
| ordinal | INT | Ordering within axis |

**UNIQUE:** `(descriptor_id, axis, term)`

Single table for both axis values and cognitive profile entries,
discriminated by `entry_type` column.

Existing single-String disposition columns on the descriptor entity are
removed. `delegation` remains as a boolean column on the descriptor.

### New table: disposition_signal

| Column | Type | Notes |
|--------|------|-------|
| id | BIGINT PK | |
| agent_id | VARCHAR FK | |
| tenancy_id | VARCHAR FK | |
| function_term | VARCHAR(30) | Activated function term value |
| count | INT | Accumulated activation count |

**UNIQUE:** `(agent_id, tenancy_id, function_term)`

Separate from `behavioral_signal` — different lifecycle semantics
(cumulative vs. TTL-managed).

---

## §6 Documentation (#114)

Update `docs/personality-frameworks.md`:

### §6.1 New section: §2.4 Jungian Cognitive Functions

- The 8 functions, descriptions, dominant-auxiliary stack model
- JPAF findings: 100% alignment, TAA >90%, PSA 100% for capable models
- Why function-level specification succeeds: weighted profiles, compensation
  mechanism, continuous (not dichotomous) representation

### §6.2 Revise §2.3 (MBTI)

Keep the original rejection intact — the test-retest critique is
scientifically valid for human personality measurement.

Add subsection: "Jungian rehabilitation." The test-retest critique assumes
personality is *measured* from observed behavior. For LLM agents, personality
is *specified* — declared and injected via structured prompting. No
measurement error because no measurement. The instability is in the
assessment instrument, not the type system.

MBTI types are now supported via `MbtiTypeTerm`, grounded in
`JungianFunctionTerm` via `specializes()`. The type label is an emergent
property of the weighted function stack — not an injected identity. The
dichotomous scoring problem is resolved: weights are continuous.

**Update Anti-pattern 1** to distinguish between human-measured MBTI
(inadvisable) and agent-specified MBTI grounded through Jungian functions
(valid). Replace:

> "MBTI as vocabulary basis: MBTI types are unstable..."

With:

> "MBTI as vocabulary basis (human-measured): When MBTI types are derived
> from human personality assessment, ~50% type-change one month later makes
> any vocabulary built on MBTI terms unreliable. Use Big Five /
> Conscientiousness vocabulary instead for assessment-derived personality.
> **Exception:** For LLM agents, personality is *specified* not *measured*
> — `MbtiTypeTerm` provides MBTI types grounded through Jungian cognitive
> functions (`specializes()` → `JungianFunctionTerm`). See §2.4 Jungian
> Cognitive Functions."

### §6.3 Cross-reference table update

Add Jungian column. Maps to all 5 disposition axes (yes, via axisExactMatch),
slot (no), capabilities (no), delegation (no).

### §6.4 New combination pattern: Jungian Profile

```
dispositionVocabulary = "urn:casehub:vocab:jungian"
dispositionProfile      = [{ti, 0.45}, {ne, 0.20}, ...]
// axes auto-derived via cross-vocabulary projection
```

### §6.5 Framework compatibility update

| Pair | Rating | Reasoning |
|------|--------|-----------|
| Jungian + Belbin | Additive | Cognitive style + team role are orthogonal |
| Jungian + DISC | Redundant | Both describe behavioral style; Jungian is deeper |
| Jungian + Conscientiousness | Redundant | Jungian projects onto Conscientiousness axes |
| MBTI + Jungian | Hierarchical | MBTI emerges from Jungian function stacks |

**Update existing compatibility row:** Replace "MBTI + anything →
Inadvisable" with "MBTI (human-assessed) + anything → Inadvisable" and
add: "MBTI (agent-specified) + Jungian → Hierarchical (see §6.2)".

---

## JPAF Coverage Matrix

| JPAF mechanism | Eidos equivalent | Beyond JPAF |
|----------------|-----------------|-------------|
| Base weights | `AgentDisposition.dispositionProfile` | Vocabulary-grounded, JPA-persisted |
| Change history | `DispositionSignalStore` | Durable, cumulative, queryable |
| Status check | `DispositionHealth.probe()` | Sealed status types, tenancy-scoped thresholds |
| Reflection 1–4 | `DispositionEvolution.evaluate()` | LLM-adjudicated + rule-based fallback |
| LLM judgment | `ChatModel` in `DefaultDispositionEvolution` | Reuses existing LangChain4j integration |
| Decay (80%) | `Dampened(decayFactor)` | Configurable via PreferenceProvider |
| Normalization | Weight normalization in evolution result | — |
| Function→MBTI | VocabularyRegistry cross-vocab resolution | Generalizable to any framework |
| Shadow mapping | `JungianFunctionTerm.shadow()` | Vocabulary method, not hardcoded dict |
| Compatible aux | `JungianFunctionTerm.compatibleAuxiliaries()` | Vocabulary method, not hardcoded dict |
| Prompt injection | EidosSystemPromptRenderer | Multi-format (MD/PROSE/A2A_CARD) |
| — | Cross-vocabulary projection (→ DISC, Conscientiousness, Belbin) | Not in JPAF |
| — | Registry queryability (via auto-derived axes, not direct profile query) | Not in JPAF |
| — | Attestation integration (evolution → ledger) | Not in JPAF |
| — | Descriptor versioning (evolution creates new version) | Not in JPAF |
| — | Multi-model eval (Claude, Ollama, Jlama) | JPAF: GPT-4, Llama, Qwen |
| — | Contextual weights (future) | Not in JPAF |

---

## Module Placement

| Component | Module | Tier |
|-----------|--------|------|
| DispositionValue | casehub-eidos-api | 1 |
| Evolved AgentDisposition | casehub-eidos-api | 1 |
| DispositionHealth SPI | casehub-eidos-api | 1 |
| DispositionEvolution SPI | casehub-eidos-api | 1 |
| DispositionSignalStore SPI | casehub-eidos-api | 1 |
| EvolutionType interface | casehub-eidos-api | 1 |
| JungianFunctionTerm | casehub-eidos-vocab | — |
| MbtiTypeTerm | casehub-eidos-vocab | — |
| FunctionCategory, FunctionAttitude | casehub-eidos-vocab | — |
| JungianEvolutionType | casehub-eidos-vocab | — |
| JungianVocabRegistrar | casehub-eidos-vocab | — |
| MbtiVocabRegistrar | casehub-eidos-vocab | — |
| DefaultDispositionHealth | casehub-eidos (runtime) | 3 |
| DefaultDispositionEvolution | casehub-eidos (runtime) | 3 |
| NoOpDispositionHealth (@DefaultBean) | casehub-eidos (runtime) | 3 |
| NoOpDispositionSignalStore (@DefaultBean) | casehub-eidos (runtime) | 3 |
| Cognitive profile rendering | casehub-eidos (runtime) | 3 |
| Auto-derivation (profile → axes) | casehub-eidos (runtime) | 3 |
| InMemoryDispositionHealth | casehub-eidos-memory | — |
| InMemoryDispositionSignalStore | casehub-eidos-memory | — |
| MbtiAlignmentJudge | casehub-eidos-eval | — |
| FunctionActivationJudge | casehub-eidos-eval | — |
| PersonalityEvolutionJudge | casehub-eidos-eval | — |
| Schema migration | casehub-eidos (runtime) | 3 |

---

## Issue Mapping

| Issue | Scope | Section |
|-------|-------|---------|
| #108 | JungianFunctionTerm vocabulary | §1.1 |
| #109 | Cross-vocabulary equivalences | §1.3 |
| #110 | MbtiTypeTerm vocabulary | §1.2 |
| #111 | Weighted disposition profile | §2.1, §2.2 |
| #112 | Dominant-auxiliary designation | §2.3 |
| #113 | MBTI questionnaire alignment eval | §4.1 |
| #114 | Update personality-frameworks.md | §6 |
| #115 | Disposition function signal recording | §2.5 |
| #116 | DispositionHealth SPI | §2.6, §2.7 |
| #117 | Personality evolution testing (TAA/PSA) | §4.2, §4.3 |
