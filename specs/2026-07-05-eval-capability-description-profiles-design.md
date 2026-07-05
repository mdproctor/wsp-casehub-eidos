# Eval Profiles Exercising Capability Description Field

**Issue:** casehubio/eidos#95
**Deferred from:** #77 (capability description spec)
**Date:** 2026-07-05

## Problem

The `AgentCapability.description` field (nullable String, ≤500 chars) was added in #77. All rendering paths handle it:

- **MARKDOWN**: `"- **name** — description"` (EidosRenderPipeline:362)
- **PROSE**: `"name (description)"` (EidosRenderPipeline:487)
- **A2A_CARD**: enriched description wins when non-blank; declared `cap.description()` is fallback (EidosRenderPipeline:644-649)

Zero of the 8 existing eval profiles populate `description` on any capability. This means:

1. YAML → `AgentCapability` deserialization of `description` is untested in the profile pipeline
2. MARKDOWN/PROSE structural rendering of descriptions is untested for real-world profiles
3. A2A_CARD declared-description fallback has no meaningful data to fall back to
4. The `PromptJudge` COMPLETENESS dimension is only exercised via LLM-generated descriptions, never declared ones

## Design

### New YAML Profiles

Two new profiles in `eval/src/test/resources/profiles/`, forming a variant pair on the **AUTONOMY** axis in the **data engineering** domain.

**Domain grounding:** O*NET 15-2051.01 (Business Intelligence Analysts) — tasks include designing data pipelines, assessing data quality, managing schema evolution, and producing analytical outputs.

#### `data-engineer-autonomous.yaml`

- **slot:** data-engineer
- **AUTONOMY:** autonomous
- **delegation:** true
- **Belbin:** plant (creative, independent problem-solver)
- **DISC:** D (dominant, decisive)
- **Capabilities (all with descriptions):**

| Name | Description | qualityHint | latencyHintP50Ms | costHint |
|------|-------------|-------------|------------------|----------|
| `pipeline-orchestration` | Designs and operates end-to-end data pipelines, selecting extraction strategies, transformation logic, and load targets based on data volume, latency requirements, and downstream consumer needs | 0.90 | 60000 | medium |
| `data-quality-assessment` | Evaluates dataset completeness, consistency, and accuracy against defined schema contracts and business rules, producing remediation recommendations when quality thresholds are breached | 0.88 | 30000 | low |
| `schema-evolution` | Manages schema changes across versioned datasets, assessing backward compatibility, migration impact, and downstream consumer contracts before applying changes | 0.85 | 45000 | medium |

- **epistemicDomains:**
  - pipeline-orchestration: `{data-engineering: 0.92, streaming: 0.78, batch-processing: 0.88}`
  - data-quality-assessment: `{data-profiling: 0.90, schema-validation: 0.85}`
  - schema-evolution: `{data-modeling: 0.88, migration-tooling: 0.80, api-versioning: 0.72}`
- **Other disposition axes:** socialOrient=collaborative, ruleFollowing=adaptive, riskAppetite=moderate (shared with directed variant)

#### `data-engineer-directed.yaml`

Identical to autonomous except:
- **AUTONOMY:** directed
- **delegation:** false
- **Belbin:** implementer (systematic, follows established patterns)
- **DISC:** C (conscientious, detail-oriented)

Same capabilities, same descriptions, same hints, same epistemic domains. Only AUTONOMY and delegation differ — required by `AgentProfileLoader.validateVariantPairs()`.

#### Briefings (per protocol PP-20260617-bfc66f)

Both profiles have `vocabularyGaps` entries. Briefings derive from `loss: FULL` entries:

- **Autonomous:** Emphasises architectural decision-making authority — pipeline design choices, schema migration timing, and technology selection are within scope without escalation.
- **Directed:** Emphasises runbook adherence and escalation discipline — design decisions and schema changes require approval; execution follows documented patterns.

#### Expected Traits

| Axis | Autonomous | Directed |
|------|-----------|----------|
| AUTONOMY | HIGH | LOW |
| SOCIAL_ORIENTATION | NEUTRAL | NEUTRAL |
| RULE_FOLLOWING | NEUTRAL | NEUTRAL |
| RISK_APPETITE | NEUTRAL | NEUTRAL |

### index.yaml Updates

```yaml
profiles:
  # ... existing 8 ...
  - data-engineer-autonomous.yaml
  - data-engineer-directed.yaml

variants:
  # ... existing 2 pairs ...
  - primaryAxis: AUTONOMY
    higher: data-engineer-autonomous
    lower: data-engineer-directed
    scenarioQuestions:
      - "A critical pipeline fails at 2am. You have a fix but it changes the schema contract. What do you do?"
      - "A data scientist asks you to add a new column to a production dataset. The request is reasonable but undocumented. How do you proceed?"
      - "You discover that a downstream consumer is using an undocumented field. Removing it would simplify the schema. What do you recommend?"
```

### Synthetic A2A_CARD Case in EvalDataset

New method `dataEngineerA2a()` added to `EvalDataset.java`. Builds an `AgentDescriptor` with the same three capabilities and descriptions as the YAML profiles, format `A2A_CARD`. Added to `EvalDataset.all()`.

This exercises:
- `computeA2aMissingDescriptions()` with declared descriptions present
- COMPLETENESS judge dimension with declared descriptions as the data source
- The A2A card rendering path where `cap.description()` provides fallback content

### What We Don't Change

- **Existing 8 profiles** — untouched; continue testing the null-description path
- **`RealWorldEvalDataset`** — no code changes; new profiles picked up automatically via `index.yaml`
- **`PromptJudge`**, **`EidosRenderPipeline`**, **`AgentProfileLoader`** — no changes; all already handle descriptions
- **Eval score floors** — unchanged; new profiles should meet existing floors
- **Proximity, trait expression, pair contrast judges** — no changes; AUTONOMY axis is new coverage but requires no modifications

## Change Set

| File | Action |
|------|--------|
| `eval/src/test/resources/profiles/data-engineer-autonomous.yaml` | New |
| `eval/src/test/resources/profiles/data-engineer-directed.yaml` | New |
| `eval/src/test/resources/profiles/index.yaml` | Add 2 profile entries + 1 variant pair |
| `eval/src/main/java/io/casehub/eidos/eval/EvalDataset.java` | Add `dataEngineerA2a()` + list entry |

## Protocols Checked

- **PP-20260617-bfc66f** (eval-profile-briefing-from-vocabulary-gaps): Briefings derived from `vocabularyGap: FULL` entries
- **PP-20260611-228599** (capability-metadata-rendering): Descriptions render in all formats; numeric routing signals (qualityHint, latencyHintP50Ms, etc.) appear in A2A_CARD only

## Verification

1. `mvn test -pl eval` — `AgentProfileLoaderTest` validates pair isolation (AUTONOMY differs, all other axes identical)
2. `mvn test -pl eval -Peval` — `evaluateAllScenarios()` runs new A2A_CARD case; `evaluateRealWorldScenarios()` runs new profiles in MARKDOWN + PROSE
3. All existing tests pass unchanged — existing profiles with null descriptions are unaffected
