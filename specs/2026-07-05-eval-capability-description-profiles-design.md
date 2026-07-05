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

**Domain grounding:** O*NET 15-2051.01 (Business Intelligence Analysts) — tasks include designing data pipelines, assessing data quality, managing schema evolution, and producing analytical outputs. Data engineering is a cross-cutting role without a dedicated O*NET code; 15-2051.01 covers the closest task cluster (pipeline management, data quality, schema design). The `sourceCitation` acknowledges this approximation.

**Shared profile metadata** (identical for both profiles unless noted):

| Field | Value |
|-------|-------|
| `domain` | `data-engineering` |
| `sourceUrl` | `https://www.onetonline.org/link/summary/15-2051.01` |
| `sourceCitation` | `O*NET Online, Business Intelligence Analysts (15-2051.01), 2024; synthesised data-engineer system prompt` |
| `sourceType` | `ONET_SYNTHESISED` |

**`evalGoal`** (shared):

- **description:** `"Manage a data pipeline failure affecting a production analytics dashboard"`
- **subGoals:** Diagnose root cause of pipeline failure · Assess impact on downstream consumers and SLAs · Implement or recommend a fix based on urgency and schema impact
- **caseRef:** `~`

#### `data-engineer-autonomous.yaml`

- **name:** data-engineer-autonomous
- **role:** Data Engineer — Pipeline Orchestration (Autonomous)
- **slot:** data-engineer
- **AUTONOMY:** autonomous
- **delegation:** false
- **Belbin:** plant (creative, independent problem-solver)
- **DISC:** D (dominant, decisive)
- **originalProse:**

  > You are a senior data engineer responsible for the organisation's analytical data
  > infrastructure. You independently design and operate data pipelines, choosing
  > extraction strategies, transformation logic, and load targets based on your
  > assessment of data volume, latency requirements, and downstream consumer needs.
  > When pipeline failures occur, you diagnose root causes and implement fixes
  > without waiting for approval, escalating only when changes affect cross-team
  > schema contracts. You evaluate dataset quality against business rules and
  > schema contracts, and you proactively initiate remediation when quality
  > thresholds are breached. Schema evolution is within your authority — you assess
  > backward compatibility, migration impact, and downstream contracts, then apply
  > changes when you judge the risk acceptable.

- **notes:** Derived from O*NET 15-2051.01 tasks. Autonomy-axis variant, autonomous pole. Stage 0 pair partner: data-engineer-directed (differs only on AUTONOMY). All other disposition axes identical: socialOrient=collaborative, ruleFollowing=adaptive, riskAppetite=moderate, conflictMode=null, delegation=false.
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
- **inputTypes / outputTypes / tags:**
  - pipeline-orchestration: [data-source, pipeline-config] → [pipeline-run, monitoring-report], tags: []
  - data-quality-assessment: [dataset, schema-contract] → [quality-report, remediation-plan], tags: []
  - schema-evolution: [schema-definition, consumer-contract] → [migration-plan, compatibility-report], tags: []
- **Other disposition axes:** socialOrient=collaborative, ruleFollowing=adaptive, riskAppetite=moderate, conflictMode=null (all shared with directed variant)
- **vocabularyGaps:**
  - `proactive-remediation-initiative`: Independently identifying and acting on data quality issues before being asked, based on personal judgment of urgency. No disposition axis captures the proactive-vs-reactive dimension. **loss: FULL**
  - `architectural-ownership`: Authority to make pipeline architecture and technology decisions without external approval. AUTONOMY=autonomous approximates independence but loses the domain-specific architectural scope. **loss: PARTIAL**

#### `data-engineer-directed.yaml`

Identical to autonomous except:
- **name:** data-engineer-directed
- **role:** Data Engineer — Pipeline Orchestration (Directed)
- **AUTONOMY:** directed
- **Belbin:** implementer (systematic, follows established patterns)
- **DISC:** C (conscientious, detail-oriented)
- **originalProse:**

  > You are a senior data engineer responsible for the organisation's analytical data
  > infrastructure. You operate data pipelines following documented runbooks and
  > established patterns. When pipeline failures occur, you follow the incident
  > response playbook and escalate design decisions to the team lead. You evaluate
  > dataset quality against predefined thresholds and report breaches through the
  > standard escalation process. Schema changes require approval from the data
  > architecture team — you prepare impact assessments and compatibility reports
  > but do not apply changes independently. Your strength is reliable execution
  > of well-defined processes and thorough documentation of pipeline behaviour.

- **notes:** Derived from O*NET 15-2051.01 tasks. Autonomy-axis variant, directed pole. Stage 0 pair partner: data-engineer-autonomous (differs only on AUTONOMY). All other disposition axes identical: socialOrient=collaborative, ruleFollowing=adaptive, riskAppetite=moderate, conflictMode=null, delegation=false.
- **vocabularyGaps:**
  - `scope-bounded-execution`: Operating within a defined decision mandate, where exceeding scope triggers escalation rather than independent judgment. This concept has surface overlap with ruleFollowing (both constrain action) but is fundamentally about decision authority, not rule adherence. The variant pair constrains ruleFollowing=adaptive, so this scope-bounding cannot be expressed through any axis. **loss: FULL**
  - `approval-gated-design`: Design decisions require explicit approval from designated authority (data architecture team), not just delayed action. AUTONOMY=directed captures "waits for instruction" but loses the "actively seeks approval" dimension. **loss: PARTIAL**

Same capabilities, same descriptions, same hints, same epistemic domains, same inputTypes/outputTypes/tags. Only AUTONOMY differs — required by `AgentProfileLoader.validateVariantPairs()` and protocol PP-20260602-64fde8.

#### Briefings (per protocol PP-20260617-bfc66f)

Briefings are derived from `loss: FULL` vocabulary gap entries (see `vocabularyGaps` in each profile above). Under 500 chars, second person, 2–3 sentences.

- **Autonomous** (from `proactive-remediation-initiative`): "You initiate remediation when you judge quality thresholds are at risk — you do not wait for a ticket or escalation to act. Pipeline design, schema migration timing, and technology selection are decisions you make based on your own assessment of the data."
- **Directed** (from `scope-bounded-execution`): "Design decisions and schema changes are outside your mandate — you prepare impact assessments and compatibility reports but defer final judgment to the data architecture team. When pipeline failures require changes beyond your established scope, you escalate rather than improvise."

Note: the directed briefing deliberately avoids rule-following language ("runbook adherence", "follows documented patterns") to prevent confounding the AUTONOMY effect measured by `PairContrastJudge`. The `scope-bounded-execution` vocabulary gap documents this tension — the persona wants to express process-adherence, but the briefing must stay in the autonomy lane because the variant pair constrains `ruleFollowing=adaptive` on both sides.

#### Expected Traits

| Axis | Autonomous | Directed |
|------|-----------|----------|
| AUTONOMY | HIGH | LOW |
| SOCIAL_ORIENTATION | NEUTRAL | NEUTRAL |
| RULE_FOLLOWING | NEUTRAL | NEUTRAL |
| RISK_APPETITE | NEUTRAL | NEUTRAL |
| CONFLICT_MODE | — | — |

CONFLICT_MODE is intentionally omitted from both profiles (null on both sides). No polarity is expected — the data-engineer persona does not exercise conflict-handling behaviour. This matches existing profiles (sw-engineer, security-analyst) where `conflictMode` is null.

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
- `computeA2aMissingDescriptions()` with declared descriptions present — validates the A2A JSON contains description entries for all capabilities
- COMPLETENESS judge dimension with capabilities that have declared descriptions (enrichment will produce the rendered descriptions; declared descriptions serve as fallback if enrichment fails)
- The A2A card rendering path with description-bearing capabilities — note that `EidosRenderPipeline` gives enriched descriptions priority over declared `cap.description()` (line ~644-649), so the fallback path is only exercised when enrichment is absent or fails

**Gap acknowledged:** The fallback path where enrichment is absent and `cap.description()` provides the sole A2A description is not isolated by this case or any existing case. Deferred to eidos#97.

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

- **PP-20260602-64fde8** (eval-variant-pair-single-axis-isolation): Variant pair differs on exactly one axis (AUTONOMY); all other disposition fields — socialOrient, ruleFollowing, riskAppetite, conflictMode, delegation — identical between profiles
- **PP-20260605-1000ad** (delegation-platform-semantic): `delegation=false` on both profiles; delegation is a platform capability, not correlated with autonomy axis
- **PP-20260617-bfc66f** (eval-profile-briefing-from-vocabulary-gaps): Briefings derived from `vocabularyGap: FULL` entries
- **PP-20260611-228599** (capability-metadata-rendering): Descriptions render in all formats; numeric routing signals (qualityHint, latencyHintP50Ms, etc.) appear in A2A_CARD only

## Verification

1. `mvn test -pl eval` — `AgentProfileLoaderTest` validates pair isolation (AUTONOMY differs, all other axes identical)
2. `mvn test -pl eval -Peval` — `evaluateAllScenarios()` runs new A2A_CARD case; `evaluateRealWorldScenarios()` runs new profiles in MARKDOWN + PROSE
3. All existing tests pass unchanged — existing profiles with null descriptions are unaffected
