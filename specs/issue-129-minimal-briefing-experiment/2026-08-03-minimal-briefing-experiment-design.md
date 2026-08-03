# Minimal Briefing Experiment — Isolate Framework Contribution

**Issue:** casehubio/eidos#129
**Date:** 2026-08-03

## Problem

The structured personality composition paper (Section 9, item 2) identifies a confound: every Jungian profile has briefing text that describes cognitive functions in natural language. The current eval cannot distinguish whether function activation accuracy (TAA) comes from the framework's `dispositionProfile` weights or from the briefing text that the LLM reads directly.

## Experimental Design

2×2 factorial: **framework** (present/absent) × **briefing** (rich/minimal).

| | Minimal briefing | Rich briefing |
|---|---|---|
| **Baseline** (no framework) | Floor — no personality signal | Briefing-only contribution |
| **Jungian** (framework active) | Framework-only contribution | Current state (combined) |

**Profiles:** All 8 Jungian profiles (INTP, ENTJ, INFP, ENFJ, ISTJ, ESTP, INTJ, ENTP) — covers all 8 dominant functions.

**Scenarios:** For each profile, 6 function activation scenarios — 3 targeting the dominant function + 3 targeting the auxiliary. Loaded from `function-scenarios/scenarios.yaml`.

**Render format:** MARKDOWN (the behavioral prompt channel).

**Key comparisons:**
- `Jung+Min − Base+Min` = framework independent contribution
- `Base+Rich − Base+Min` = briefing independent contribution
- `Jung+Rich − Jung+Min` = briefing contribution on top of framework

**Expected outcomes (from issue #129):**
- If TAA shifts significantly with minimal briefings → framework works but rich briefings override it
- If TAA does NOT shift → framework needs stronger integration regardless of briefing

## Data Model

### JungianProfile (test scope)

New record in `eval/src/test/java/`:

```java
record JungianProfile(
    String name,
    String role,
    String domain,
    String sourceType,
    String mbtiType,
    String dominantFunction,
    String auxiliaryFunction,
    AgentDescriptor descriptor
) {}
```

Rich briefing accessed via `descriptor.briefing()`. The `dispositionProfile` is on the descriptor's disposition; `dispositionVocabulary` is on the descriptor itself. `sourceType` is a plain String field (not the `SourceType` enum from `AgentProfile` — Jungian profiles use `"JPAF_DERIVED"` which is not in that enum).

### JungianProfileLoader (test scope)

Reads `jungian-profiles/index.yaml` for the profile list, loads each YAML into `JungianProfile`. Uses Jackson YAML with `DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES` disabled (the Jungian YAML has fields not in the record that should be silently ignored).

### FunctionScenarioLoader (test scope)

Reads `function-scenarios/scenarios.yaml`. Returns `Map<String, List<FunctionScenario>>` keyed by function code (`ti`, `te`, `fi`, `fe`, `si`, `se`, `ni`, `ne`). Each entry maps to `FunctionActivationJudge.FunctionScenario`.

### BriefingCondition (enum)

Four conditions with `apply(JungianProfile) → AgentDescriptor`:

- **BASELINE_MINIMAL** — strips `dispositionProfile`, `dispositionVocabulary`, and all derived disposition axes; replaces briefing with `"You are an agent named {role}"` (uses `role` not `name` to avoid MBTI type leaking into the minimal condition — e.g. "Systems Analyst" not "Systems Analyst (INTP)")
- **BASELINE_RICH** — strips `dispositionProfile`, `dispositionVocabulary`, and all derived disposition axes; keeps original briefing
- **JUNGIAN_MINIMAL** — keeps `dispositionProfile` AND `dispositionVocabulary` (both required for `deriveDispositionAxes` to produce axes); replaces briefing with `"You are an agent named {role}"`
- **JUNGIAN_RICH** — returns original descriptor unchanged

Each `apply()` copies the original descriptor's fields (agentId, name, slot, tenancyId, capabilities, etc.) and replaces only the varied fields. Baseline conditions build an empty `AgentDisposition` (delegation only, no profile, no vocabulary, no axes). `deriveDispositionAxes()` is a no-op for baseline conditions (no dispositionProfile to project) — this is intentional; the baseline must have zero structured personality signal.

## Test Structure

`MinimalBriefingEvalTest` — `@QuarkusTest @Tag("eval")`

```
@BeforeAll: load JungianProfiles + function scenarios

compareBriefingContribution():
  for each profile (8):
    scenarios = dominant function scenarios (3) + auxiliary function scenarios (3)
    for each BriefingCondition (4):
      descriptor = condition.apply(profile)
      derived = DescriptorCollector.deriveDispositionAxes(descriptor, vocabRegistry)
      prompt = renderer.render(derived, MARKDOWN context)
      result = functionJudge.evaluate(prompt, profile.mbtiType(), scenarios)
      collect(profile.name(), condition, result)
  
  write report
```

**Scale:** 8 profiles × 4 conditions × 6 scenarios = 192 evaluations (384 LLM calls — agent response + judge each).

**Assertions:** Diagnostic only — no hard pass/fail. One sanity check: mean JUNGIAN_RICH TAA >= mean BASELINE_MINIMAL TAA.

## Report

### Console output

```
=== Minimal Briefing Experiment ===

Profile          | Base+Min | Base+Rich | Jung+Min | Jung+Rich
─────────────────┼──────────┼───────────┼──────────┼──────────
intp-analyst     |   0.17   |   0.50    |  0.67    |   0.83
entj-architect   |   0.17   |   0.33    |  0.50    |   0.67
...
─────────────────┼──────────┼───────────┼──────────┼──────────
Mean             |   0.15   |   0.42    |  0.58    |   0.75

Framework contribution (Jung+Min − Base+Min): +0.43
Briefing contribution (Base+Rich − Base+Min): +0.27
```

### JSON report

Written to `target/briefing-experiment-report.json`:

```json
{
  "timestamp": "2026-08-03T...",
  "modelLabel": "claude",
  "profiles": [
    {
      "name": "intp-analyst",
      "mbtiType": "INTP",
      "dominantFunction": "ti",
      "auxiliaryFunction": "ne",
      "conditions": {
        "BASELINE_MINIMAL": { "taa": 0.17, "correct": 1, "total": 6, "activations": [...] },
        "BASELINE_RICH": { "taa": 0.50, "correct": 3, "total": 6, "activations": [...] },
        "JUNGIAN_MINIMAL": { "taa": 0.67, "correct": 4, "total": 6, "activations": [...] },
        "JUNGIAN_RICH": { "taa": 0.83, "correct": 5, "total": 6, "activations": [...] }
      }
    }
  ],
  "aggregated": {
    "BASELINE_MINIMAL": { "meanTaa": 0.15 },
    "BASELINE_RICH": { "meanTaa": 0.42 },
    "JUNGIAN_MINIMAL": { "meanTaa": 0.58 },
    "JUNGIAN_RICH": { "meanTaa": 0.75 }
  },
  "frameworkContribution": 0.43,
  "briefingContribution": 0.27
}
```

## Run Command

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -Peval \
  -Dtest=MinimalBriefingEvalTest#compareBriefingContribution
```

## New Files

| File | Scope | Purpose |
|------|-------|---------|
| `eval/src/test/java/.../JungianProfile.java` | test | Profile record |
| `eval/src/test/java/.../JungianProfileLoader.java` | test | YAML loader for jungian-profiles/ |
| `eval/src/test/java/.../FunctionScenarioLoader.java` | test | YAML loader for function-scenarios/ |
| `eval/src/test/java/.../BriefingCondition.java` | test | 2×2 condition enum with apply() |
| `eval/src/test/java/.../BriefingExperimentReport.java` | test | Report record |
| `eval/src/test/java/.../MinimalBriefingEvalTest.java` | test | Experiment test |

## Non-Goals

- No Belbin or composite layers — the 2×2 answers the confound question; additional layers can be added as a follow-up if needed
- No repeat runs for statistical averaging — the 3 scenarios per function provide within-condition variance; full experiment repetition is a separate concern
- No modifications to existing eval infrastructure — this is additive
