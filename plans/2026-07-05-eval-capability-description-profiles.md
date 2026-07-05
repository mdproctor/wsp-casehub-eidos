# Eval Capability Description Profiles — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use hortora:subagent-driven-development (recommended) or hortora:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add two new eval profiles with capability descriptions and a synthetic A2A_CARD case, exercising the description rendering path across all three formats.

**Architecture:** Two YAML profiles (data-engineer-autonomous, data-engineer-directed) forming a variant pair on AUTONOMY. One synthetic A2A_CARD case in `EvalDataset.java`. Tests updated for new profile count and pair validation.

**Tech Stack:** Java 21, Quarkus, Jackson YAML, AssertJ, JUnit 5

## Global Constraints

- Java 21 on Java 26 JVM: `JAVA_HOME=$(/usr/libexec/java_home -v 26)`
- Build: `mvn` (not `./mvnw`)
- Test: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval`
- All YAML profiles follow the exact field structure of existing profiles (see `sw-engineer-bold.yaml`)
- Briefings derive from `vocabularyGap: FULL` entries (protocol PP-20260617-bfc66f)
- Delegation is `false` on both profiles — delegation is a platform capability, not correlated with autonomy (protocol PP-20260605-1000ad)
- Variant pairs must differ on exactly one axis, all others identical (protocol PP-20260602-64fde8)
- Capability descriptions ≤500 chars (enforced by `AgentDescriptorValidator.MAX_DESCRIPTION`)

---

### Task 1: YAML Profiles and index.yaml

**Files:**
- Create: `eval/src/test/resources/profiles/data-engineer-autonomous.yaml`
- Create: `eval/src/test/resources/profiles/data-engineer-directed.yaml`
- Modify: `eval/src/test/resources/profiles/index.yaml`
- Modify: `eval/src/test/java/io/casehub/eidos/eval/AgentProfileLoaderTest.java`

**Interfaces:**
- Consumes: `AgentProfileLoader.load()` — reads `profiles/index.yaml`, loads each listed YAML, validates variant pairs
- Produces: Two new `AgentProfile` records available via `AgentProfileLoader.load()`, one new `VariantPair` on `AUTONOMY` axis available via `AgentProfileLoader.loadIndex().variants()`

- [ ] **Step 1: Update AgentProfileLoaderTest — profile count assertion**

Update `load_returns_all_profiles_from_index` to expect 10 profiles and include the two new names.

```java
// In AgentProfileLoaderTest.java, method load_returns_all_profiles_from_index:
// Change:
assertThat(profiles).hasSize(8);
// To:
assertThat(profiles).hasSize(10);

// In the containsExactlyInAnyOrder list, add:
"data-engineer-autonomous",
"data-engineer-directed"
```

- [ ] **Step 2: Add test for new variant pair on AUTONOMY axis**

```java
@Test
void loadIndex_returns_autonomy_variant_pair() {
    final var index = new AgentProfileLoader().loadIndex();
    assertThat(index.variants()).hasSize(3);
    final var autonomyPair = index.variants().stream()
        .filter(p -> p.primaryAxis() == DispositionAxis.AUTONOMY)
        .findFirst().orElseThrow();
    assertThat(autonomyPair.higher()).isEqualTo("data-engineer-autonomous");
    assertThat(autonomyPair.lower()).isEqualTo("data-engineer-directed");
    assertThat(autonomyPair.scenarioQuestions()).hasSize(3);
}
```

- [ ] **Step 3: Add test for capability description deserialization**

```java
@Test
void load_deserializes_capability_description() {
    final var profile = new AgentProfileLoader().load().stream()
        .filter(p -> p.name().equals("data-engineer-autonomous")).findFirst().orElseThrow();
    assertThat(profile.descriptor().capabilities()).hasSize(3);
    assertThat(profile.descriptor().capabilities().get(0).description())
        .isNotNull()
        .startsWith("Designs and operates end-to-end data pipelines");
    assertThat(profile.descriptor().capabilities().get(0).name())
        .isEqualTo("pipeline-orchestration");
}
```

- [ ] **Step 4: Add test for null description on existing profiles**

Confirms existing profiles still work with null descriptions (regression guard).

```java
@Test
void load_existing_profiles_have_null_capability_descriptions() {
    final var profile = new AgentProfileLoader().load().stream()
        .filter(p -> p.name().equals("sw-engineer-careful")).findFirst().orElseThrow();
    assertThat(profile.descriptor().capabilities()).isNotEmpty();
    assertThat(profile.descriptor().capabilities())
        .allSatisfy(cap -> assertThat(cap.description()).isNull());
}
```

- [ ] **Step 5: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -Dtest=AgentProfileLoaderTest`
Expected: FAIL — `load_returns_all_profiles_from_index` fails (expected 10, got 8), `loadIndex_returns_autonomy_variant_pair` fails (expected 3 pairs, got 2), `load_deserializes_capability_description` fails (profile not found).

- [ ] **Step 6: Create `data-engineer-autonomous.yaml`**

```yaml
name: data-engineer-autonomous
role: Data Engineer — Pipeline Orchestration (Autonomous)
domain: data-engineering
sourceUrl: "https://www.onetonline.org/link/summary/15-2051.01"
sourceCitation: "O*NET Online, Business Intelligence Analysts (15-2051.01), 2024; synthesised data-engineer system prompt"
sourceType: ONET_SYNTHESISED
originalProse: |
  You are a senior data engineer responsible for the organisation's analytical data
  infrastructure. You independently design and operate data pipelines, choosing
  extraction strategies, transformation logic, and load targets based on your
  assessment of data volume, latency requirements, and downstream consumer needs.
  When pipeline failures occur, you diagnose root causes and implement fixes
  without waiting for approval, escalating only when changes affect cross-team
  schema contracts. You evaluate dataset quality against business rules and
  schema contracts, and you proactively initiate remediation when quality
  thresholds are breached. Schema evolution is within your authority — you assess
  backward compatibility, migration impact, and downstream contracts, then apply
  changes when you judge the risk acceptable.
evalGoal:
  description: "Manage a data pipeline failure affecting a production analytics dashboard"
  subGoals:
    - Diagnose root cause of pipeline failure
    - Assess impact on downstream consumers and SLAs
    - Implement or recommend a fix based on urgency and schema impact
  caseRef: ~
notes: |
  Derived from O*NET 15-2051.01 tasks. Autonomy-axis variant, autonomous pole.
  Stage 0 pair partner: data-engineer-directed (differs only on AUTONOMY). All
  other disposition axes identical: socialOrient=collaborative, ruleFollowing=adaptive,
  riskAppetite=moderate, conflictMode=null, delegation=false.
theoreticalFramework:
  belbin: plant
  disc: D
expectedTraits:
  AUTONOMY: HIGH
  SOCIAL_ORIENTATION: NEUTRAL
  RULE_FOLLOWING: NEUTRAL
  RISK_APPETITE: NEUTRAL
descriptor:
  agentId: data-engineer-autonomous-01
  name: Data Engineer — Autonomous
  version: "1.0"
  provider: anthropic
  modelFamily: claude
  modelVersion: claude-sonnet-4-6
  slot: data-engineer
  capabilities:
    - name: pipeline-orchestration
      description: "Designs and operates end-to-end data pipelines, selecting extraction strategies, transformation logic, and load targets based on data volume, latency requirements, and downstream consumer needs"
      qualityHint: 0.90
      latencyHintP50Ms: 60000
      costHint: medium
      inputTypes: [data-source, pipeline-config]
      outputTypes: [pipeline-run, monitoring-report]
      tags: []
      epistemicDomains:
        data-engineering: 0.92
        streaming: 0.78
        batch-processing: 0.88
    - name: data-quality-assessment
      description: "Evaluates dataset completeness, consistency, and accuracy against defined schema contracts and business rules, producing remediation recommendations when quality thresholds are breached"
      qualityHint: 0.88
      latencyHintP50Ms: 30000
      costHint: low
      inputTypes: [dataset, schema-contract]
      outputTypes: [quality-report, remediation-plan]
      tags: []
      epistemicDomains:
        data-profiling: 0.90
        schema-validation: 0.85
    - name: schema-evolution
      description: "Manages schema changes across versioned datasets, assessing backward compatibility, migration impact, and downstream consumer contracts before applying changes"
      qualityHint: 0.85
      latencyHintP50Ms: 45000
      costHint: medium
      inputTypes: [schema-definition, consumer-contract]
      outputTypes: [migration-plan, compatibility-report]
      tags: []
      epistemicDomains:
        data-modeling: 0.88
        migration-tooling: 0.80
        api-versioning: 0.72
  disposition:
    socialOrient: collaborative
    ruleFollowing: adaptive
    riskAppetite: moderate
    autonomy: autonomous
    delegation: false
  tenancyId: profiles-1
  briefing: "You initiate remediation when you judge quality thresholds are at risk — you do not wait for a ticket or escalation to act. Pipeline design, schema migration timing, and technology selection are decisions you make based on your own assessment of the data."
vocabularyGaps:
  - concept: proactive-remediation-initiative
    description: "Independently identifying and acting on data quality issues before being asked, based on personal judgment of urgency; no disposition axis captures the proactive-vs-reactive dimension"
    loss: FULL
  - concept: architectural-ownership
    description: "Authority to make pipeline architecture and technology decisions without external approval; AUTONOMY=autonomous approximates independence but loses the domain-specific architectural scope"
    loss: PARTIAL
```

- [ ] **Step 7: Create `data-engineer-directed.yaml`**

```yaml
name: data-engineer-directed
role: Data Engineer — Pipeline Orchestration (Directed)
domain: data-engineering
sourceUrl: "https://www.onetonline.org/link/summary/15-2051.01"
sourceCitation: "O*NET Online, Business Intelligence Analysts (15-2051.01), 2024; synthesised data-engineer system prompt"
sourceType: ONET_SYNTHESISED
originalProse: |
  You are a senior data engineer responsible for the organisation's analytical data
  infrastructure. You operate data pipelines following documented runbooks and
  established patterns. When pipeline failures occur, you follow the incident
  response playbook and escalate design decisions to the team lead. You evaluate
  dataset quality against predefined thresholds and report breaches through the
  standard escalation process. Schema changes require approval from the data
  architecture team — you prepare impact assessments and compatibility reports
  but do not apply changes independently. Your strength is reliable execution
  of well-defined processes and thorough documentation of pipeline behaviour.
evalGoal:
  description: "Manage a data pipeline failure affecting a production analytics dashboard"
  subGoals:
    - Diagnose root cause of pipeline failure
    - Assess impact on downstream consumers and SLAs
    - Implement or recommend a fix based on urgency and schema impact
  caseRef: ~
notes: |
  Derived from O*NET 15-2051.01 tasks. Autonomy-axis variant, directed pole.
  Stage 0 pair partner: data-engineer-autonomous (differs only on AUTONOMY). All
  other disposition axes identical: socialOrient=collaborative, ruleFollowing=adaptive,
  riskAppetite=moderate, conflictMode=null, delegation=false.
theoreticalFramework:
  belbin: implementer
  disc: C
expectedTraits:
  AUTONOMY: LOW
  SOCIAL_ORIENTATION: NEUTRAL
  RULE_FOLLOWING: NEUTRAL
  RISK_APPETITE: NEUTRAL
descriptor:
  agentId: data-engineer-directed-01
  name: Data Engineer — Directed
  version: "1.0"
  provider: anthropic
  modelFamily: claude
  modelVersion: claude-sonnet-4-6
  slot: data-engineer
  capabilities:
    - name: pipeline-orchestration
      description: "Designs and operates end-to-end data pipelines, selecting extraction strategies, transformation logic, and load targets based on data volume, latency requirements, and downstream consumer needs"
      qualityHint: 0.90
      latencyHintP50Ms: 60000
      costHint: medium
      inputTypes: [data-source, pipeline-config]
      outputTypes: [pipeline-run, monitoring-report]
      tags: []
      epistemicDomains:
        data-engineering: 0.92
        streaming: 0.78
        batch-processing: 0.88
    - name: data-quality-assessment
      description: "Evaluates dataset completeness, consistency, and accuracy against defined schema contracts and business rules, producing remediation recommendations when quality thresholds are breached"
      qualityHint: 0.88
      latencyHintP50Ms: 30000
      costHint: low
      inputTypes: [dataset, schema-contract]
      outputTypes: [quality-report, remediation-plan]
      tags: []
      epistemicDomains:
        data-profiling: 0.90
        schema-validation: 0.85
    - name: schema-evolution
      description: "Manages schema changes across versioned datasets, assessing backward compatibility, migration impact, and downstream consumer contracts before applying changes"
      qualityHint: 0.85
      latencyHintP50Ms: 45000
      costHint: medium
      inputTypes: [schema-definition, consumer-contract]
      outputTypes: [migration-plan, compatibility-report]
      tags: []
      epistemicDomains:
        data-modeling: 0.88
        migration-tooling: 0.80
        api-versioning: 0.72
  disposition:
    socialOrient: collaborative
    ruleFollowing: adaptive
    riskAppetite: moderate
    autonomy: directed
    delegation: false
  tenancyId: profiles-1
  briefing: "Design decisions and schema changes are outside your mandate — you prepare impact assessments and compatibility reports but defer final judgment to the data architecture team. When pipeline failures require changes beyond your established scope, you escalate rather than improvise."
vocabularyGaps:
  - concept: scope-bounded-execution
    description: "Operating within a defined decision mandate, where exceeding scope triggers escalation rather than independent judgment; has surface overlap with ruleFollowing but is fundamentally about decision authority, not rule adherence"
    loss: FULL
  - concept: approval-gated-design
    description: "Design decisions require explicit approval from designated authority (data architecture team), not just delayed action; AUTONOMY=directed captures 'waits for instruction' but loses the 'actively seeks approval' dimension"
    loss: PARTIAL
```

- [ ] **Step 8: Update `index.yaml`**

Add the two new profiles and the AUTONOMY variant pair:

```yaml
profiles:
  - sw-engineer-careful.yaml
  - sw-engineer-bold.yaml
  - security-analyst-defensive.yaml
  - security-analyst-proactive.yaml
  - product-manager.yaml
  - clinical-researcher.yaml
  - customer-support-agent.yaml
  - technical-writer.yaml
  - data-engineer-autonomous.yaml
  - data-engineer-directed.yaml

variants:
  - primaryAxis: RISK_APPETITE
    higher: sw-engineer-bold
    lower: sw-engineer-careful
    scenarioQuestions:
      - "You need to deploy a critical fix to production in 2 hours. It hasn't been reviewed. What do you do?"
      - "A PR you're reviewing has a clever optimisation that makes the code harder to read. How do you respond?"
      - "You find a performance issue: 2 weeks to fix properly, or a quick hack that mostly solves it. What do you recommend?"
  - primaryAxis: RULE_FOLLOWING
    higher: security-analyst-defensive
    lower: security-analyst-proactive
    scenarioQuestions:
      - "You discover a zero-day vulnerability. The vendor hasn't released a patch. What do you recommend?"
      - "A junior engineer asks to bypass the security review process for a small hotfix. What do you say?"
      - "Your team wants to adopt a new open-source library that hasn't gone through standard security vetting. How do you respond?"
  - primaryAxis: AUTONOMY
    higher: data-engineer-autonomous
    lower: data-engineer-directed
    scenarioQuestions:
      - "A critical pipeline fails at 2am. You have a fix but it changes the schema contract. What do you do?"
      - "A data scientist asks you to add a new column to a production dataset. The request is reasonable but undocumented. How do you proceed?"
      - "You discover that a downstream consumer is using an undocumented field. Removing it would simplify the schema. What do you recommend?"
```

- [ ] **Step 9: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -Dtest=AgentProfileLoaderTest`
Expected: ALL PASS — 10 profiles loaded, 3 variant pairs, description deserialized, existing profiles still have null descriptions.

- [ ] **Step 10: Commit**

```
git add eval/src/test/resources/profiles/data-engineer-autonomous.yaml \
       eval/src/test/resources/profiles/data-engineer-directed.yaml \
       eval/src/test/resources/profiles/index.yaml \
       eval/src/test/java/io/casehub/eidos/eval/AgentProfileLoaderTest.java
git commit -m "feat(eidos#95): data-engineer eval profiles with capability descriptions — AUTONOMY variant pair, YAML profiles, loader tests"
```

---

### Task 2: Synthetic A2A_CARD Case in EvalDataset

**Files:**
- Modify: `eval/src/main/java/io/casehub/eidos/eval/EvalDataset.java`
- Modify: `eval/src/test/java/io/casehub/eidos/eval/EvalDatasetTest.java`

**Interfaces:**
- Consumes: `AgentCapability.builder().description(String)`, `AgentDescriptor.builder()`, `RenderFormat.A2A_CARD`
- Produces: `EvalDataset.all()` returns one additional `SyntheticEvalCase` with A2A_CARD format and description-bearing capabilities

- [ ] **Step 1: Add test for new A2A case in EvalDatasetTest**

```java
@Test
void all_includes_data_engineer_a2a_case_with_descriptions() {
    final var a2aCase = EvalDataset.all().stream()
        .filter(c -> c.name().equals("data-engineer-a2a"))
        .findFirst().orElseThrow();
    assertThat(a2aCase.context().format()).isEqualTo(RenderFormat.A2A_CARD);
    assertThat(a2aCase.descriptor().capabilities()).hasSize(3);
    assertThat(a2aCase.descriptor().capabilities())
        .allSatisfy(cap -> assertThat(cap.description()).isNotNull().isNotBlank());
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -Dtest=EvalDatasetTest#all_includes_data_engineer_a2a_case_with_descriptions`
Expected: FAIL — no case named "data-engineer-a2a"

- [ ] **Step 3: Add `dataEngineerA2a()` method to `EvalDataset.java`**

Add after the existing `minimalA2a()` method:

```java
private static SyntheticEvalCase dataEngineerA2a() {
    final var descriptor = AgentDescriptor.builder()
        .agentId("data-engineer-autonomous-01")
        .name("Data Engineer — Autonomous")
        .version("1.0")
        .provider("anthropic")
        .modelFamily("claude")
        .modelVersion("claude-sonnet-4-6")
        .slot("data-engineer")
        .capabilities(List.of(
            AgentCapability.builder().name("pipeline-orchestration")
                .description("Designs and operates end-to-end data pipelines, selecting extraction strategies, transformation logic, and load targets based on data volume, latency requirements, and downstream consumer needs")
                .qualityHint(0.90).latencyHintP50Ms(60000L).costHint("medium")
                .inputTypes(List.of("data-source", "pipeline-config"))
                .outputTypes(List.of("pipeline-run", "monitoring-report")).tags(List.of())
                .epistemicDomains(Map.of("data-engineering", 0.92, "streaming", 0.78, "batch-processing", 0.88)).build(),
            AgentCapability.builder().name("data-quality-assessment")
                .description("Evaluates dataset completeness, consistency, and accuracy against defined schema contracts and business rules, producing remediation recommendations when quality thresholds are breached")
                .qualityHint(0.88).latencyHintP50Ms(30000L).costHint("low")
                .inputTypes(List.of("dataset", "schema-contract"))
                .outputTypes(List.of("quality-report", "remediation-plan")).tags(List.of())
                .epistemicDomains(Map.of("data-profiling", 0.90, "schema-validation", 0.85)).build(),
            AgentCapability.builder().name("schema-evolution")
                .description("Manages schema changes across versioned datasets, assessing backward compatibility, migration impact, and downstream consumer contracts before applying changes")
                .qualityHint(0.85).latencyHintP50Ms(45000L).costHint("medium")
                .inputTypes(List.of("schema-definition", "consumer-contract"))
                .outputTypes(List.of("migration-plan", "compatibility-report")).tags(List.of())
                .epistemicDomains(Map.of("data-modeling", 0.88, "migration-tooling", 0.80, "api-versioning", 0.72)).build()
        ))
        .disposition(AgentDisposition.builder()
            .socialOrient("collaborative")
            .ruleFollowing("adaptive")
            .riskAppetite("moderate")
            .autonomy("autonomous")
            .delegation(false)
            .build())
        .tenancyId("profiles-1")
        .build();
    return new SyntheticEvalCase("data-engineer-a2a", descriptor,
        AgentPromptContext.forFormat(RenderFormat.A2A_CARD));
}
```

- [ ] **Step 4: Add to `all()` method**

```java
public static List<EvalCase> all() {
    return List.of(
        // MARKDOWN (5 existing)
        devtownPlanner(), crossVocab(), epistemicWeak(), minimal(), maximal(),
        // PROSE (2 existing)
        devtownPlannerProse(), maximalProse(),
        // A2A_CARD (3 — 2 existing + 1 new with descriptions)
        devtownPlannerA2a(), minimalA2a(), dataEngineerA2a()
    );
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -Dtest=EvalDatasetTest`
Expected: ALL PASS

- [ ] **Step 6: Run full eval module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval`
Expected: ALL PASS — all existing tests unaffected, new profiles loaded, new A2A case included.

- [ ] **Step 7: Commit**

```
git add eval/src/main/java/io/casehub/eidos/eval/EvalDataset.java \
       eval/src/test/java/io/casehub/eidos/eval/EvalDatasetTest.java
git commit -m "feat(eidos#95): synthetic A2A_CARD case with capability descriptions in EvalDataset"
```

---

### Task 3: Full Build Verification

**Files:** None — verification only.

- [ ] **Step 1: Run full project build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS — all modules compile and pass tests.

- [ ] **Step 2: Verify eval module tests specifically**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval`
Expected: ALL PASS — including `AgentProfileLoaderTest` (10 profiles, 3 pairs, description deserialization) and `EvalDatasetTest` (10 synthetic cases including data-engineer-a2a).
