# Eval Baseline & Behavioral Validation Design
**Issue:** casehubio/eidos#46  
**Date:** 2026-06-09 (revised v3)  
**Branch:** issue-46-eval-baseline-behavioral

---

## Revision notes (v3)

- `NUMERIC_AXES: List<DispositionAxis>` — explicitly four axes (no CONFLICT_MODE). Neither
  `TraitExpressionJudge` nor `VocabularyExpressivenessJudge` iterates `DispositionAxis.values()`.
- `description()` added to `DispositionAxis` alongside `jsonKey()`. `AXIS_DESCRIPTIONS` static
  maps removed entirely from all judge classes; all callers use `axis.description()` directly.
- `fromJsonKey(String)` removed — no current consumer.
- Phase 3 accuracy threshold calibrated like Phase 1 (run once, observe, set floor). The 0.8
  target is aspirational, not a first-run assertion.
- "Reuse existing renders" clause removed — JUnit 5 test methods are isolated; re-rendering
  is always required.
- `@Inject ChatModel chatModel` stated as a new field in `PromptEvalTest`.
- `VariantPair.scenarioQuestions` null handling specified: post-process in `AgentProfileLoader`.
- Position bias documented as known limitation.
- `TraitExpressionResult` maps (`expressionScores`, `directionMatches`) become
  `Map<DispositionAxis, ...>`. `PairContrastResult.primaryAxis: String → DispositionAxis`.
  `runReliabilityCheck()` updated to match.

---

## 1. AgentProviderChatModel Bridge

All five existing judges inject `ChatModel` (LangChain4j). We implement `ChatModel` on top of
the platform's `AgentProvider` SPI rather than wiring a LangChain4j REST provider.

**Location:** `eval/src/test/java/io/casehub/eidos/eval/AgentProviderChatModel.java`  
(test sources — Quarkus discovers CDI beans from the test classpath in `@QuarkusTest` contexts)

**Annotations:** `@DefaultBean @ApplicationScoped` — active when no higher-priority `ChatModel`
bean is present. Jlama's extension registers `ChatModel @ApplicationScoped`, which beats
`@DefaultBean` automatically.

**Semantics:**
- Extracts the first `SystemMessage` → `systemPrompt`, first `UserMessage` → `userPrompt`
  from the `ChatRequest`
- **`ResponseFormat` is silently discarded for all five existing judges and `BehavioralJudge`
  when running in Claude mode.** JSON structure relies entirely on prompt-engineered instructions
  in each judge's system prompt. `MalformedJudgeResponseException` is the only safety net.
- Calls `agentProvider.invoke(AgentSessionConfig.of(systemPrompt, userPrompt))`
- Collects `Multi<AgentEvent.TextDelta>` → joined `String` (blocking — acceptable in offline eval)
- Returns `ChatResponse.builder().aiMessage(AiMessage.from(text)).build()`

**AgentProvider injection:** `@Any Instance<AgentProvider>` — always satisfies CDI at
augmentation time. In Jlama mode the bean is suppressed (never invoked). In Claude mode
`ClaudeAgentProvider @ApplicationScoped` is resolved.

**pom.xml additions:**
```xml
<!-- types: AgentProvider, AgentEvent, AgentSessionConfig — test scope only -->
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-agent-api</artifactId>
    <scope>test</scope>
</dependency>
<!-- impl: ClaudeAgentProvider @ApplicationScoped -->
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-agent-claude</artifactId>
    <scope>test</scope>
</dependency>
```

---

## 2. Multi-Model Support

### Claude profile (default)

`AgentProviderChatModel @DefaultBean` is the only `ChatModel` bean. All judge calls and
Phase 3 target invocations go through Claude CLI (`ClaudeAgentProvider`).

`application-eval.properties`:
```properties
# Claude model is configured via the Claude CLI — not a Quarkus property.
# Run: claude config set model claude-sonnet-4-6
# ClaudeAgentProperties exposes only: binaryPath, defaultTimeout, maxConcurrentSessions.
```

Run:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -Peval -Dgroups=eval
```

### Jlama profile (`-Peval-jlama`)

Adds `quarkus-langchain4j-jlama` to the test classpath. Jlama registers
`ChatModel @ApplicationScoped`, which beats `AgentProviderChatModel @DefaultBean`. All judge
calls and Phase 3 target invocations go through Jlama in-process.

**Known issue:** fails at test bootstrap on Quarkus 3.32+ with "Unsupported value type:
[ALL-UNNAMED]" (GE-20260423-878486). Add `--add-opens` JVM flags in surefire (exact flags
determined from the actual error at first run). Fall back to Ollama (`langchain4j-ollama`)
if the workaround does not hold on 3.32.2.

`application-eval.properties` (Jlama additions):
```properties
quarkus.langchain4j.jlama.model-path=/path/to/model.gguf
quarkus.langchain4j.jlama.chat-model.temperature=0.0
```

Run:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -Peval,eval-jlama -Dgroups=eval
```

Maven profile in `eval/pom.xml`:
```xml
<profile>
    <id>eval-jlama</id>
    <dependencies>
        <dependency>
            <groupId>io.quarkiverse.langchain4j</groupId>
            <artifactId>quarkus-langchain4j-jlama</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <configuration>
                    <!-- exact flags determined at first run from the error -->
                    <argLine>--add-opens java.base/java.lang=ALL-UNNAMED</argLine>
                </configuration>
            </plugin>
        </plugins>
    </build>
</profile>
```

---

## 3. DispositionAxis type-safety (casehub-eidos-api and eval)

### 3a. `DispositionAxis` — add `jsonKey()` and `description()`

Two new methods in `casehub-eidos-api`. Both are axis metadata, not LLM-specific:
`jsonKey()` makes the camelCase↔enum mapping canonical; `description()` moves the axis
descriptions off judge-internal constants onto the type they describe, removing the cross-judge
coupling where `PairContrastJudge` currently imports `VocabularyExpressivenessJudge.AXIS_DESCRIPTIONS`.

`fromJsonKey(String)` is **not** added — no current consumer exists; add when needed.

```java
public String jsonKey() {
    return switch (this) {
        case SOCIAL_ORIENTATION -> "socialOrient";
        case RULE_FOLLOWING     -> "ruleFollowing";
        case RISK_APPETITE      -> "riskAppetite";
        case AUTONOMY           -> "autonomy";
        case CONFLICT_MODE      -> "conflictMode";
    };
}

public String description() {
    return switch (this) {
        case SOCIAL_ORIENTATION ->
            "how collaborative or independent the agent is — whether it seeks input and " +
            "coordinates with others or acts independently";
        case RULE_FOLLOWING ->
            "how strictly the agent follows rules and conventions versus adapting its " +
            "approach to context";
        case RISK_APPETITE ->
            "how risk-tolerant or risk-averse the agent is — whether it favours bold " +
            "decisions under uncertainty or prioritises caution";
        case AUTONOMY ->
            "how self-directed versus directed-by-others the agent is — whether it takes " +
            "initiative or waits for instruction";
        case CONFLICT_MODE ->
            "how the agent approaches disagreement and conflict — whether it avoids, " +
            "accommodates, compromises, competes, or collaborates";
    };
}
```

### 3b. `VariantPair` — `primaryAxis: String → DispositionAxis`, add `scenarioQuestions`

```java
public record VariantPair(
    DispositionAxis primaryAxis,
    String higher,
    String lower,
    List<String> scenarioQuestions   // Phase 3 — see Section 6.2
) {}
```

`index.yaml` changes from camelCase to enum names (Jackson deserializes by name automatically):
```yaml
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
```

**Null handling:** Jackson will deserialize missing `scenarioQuestions` fields as null.
`AgentProfileLoader.loadIndex()` post-processes each `VariantPair` after deserialization,
replacing null `scenarioQuestions` with `List.of()`:
```java
raw.variants().stream()
    .map(v -> new VariantPair(v.primaryAxis(), v.higher(), v.lower(),
        v.scenarioQuestions() != null ? v.scenarioQuestions() : List.of()))
    .toList()
```

### 3c. `VariantPair.primaryAxis` ripple — `AgentProfileLoader` and `PairContrastResult`

`AgentProfileLoader.axisValue(AgentDisposition, String)` is **deleted**. The validation loop
uses `agentDisposition.get(pair.primaryAxis())` — already supported by
`AgentDisposition.get(DispositionAxis) → Optional<String>`.

`AgentProfileLoader.AXES: List<String>` is replaced by **`DispositionAxis.values()`** in
`validateVariantPairs()`. This is safe: `AgentDisposition.get(CONFLICT_MODE)` returns
`Optional.empty()` when the field is absent; two `Optional.empty()` compare equal via
`Objects.equals`, so pairs without `conflictMode` pass validation. The upgrade also makes
validation cover `CONFLICT_MODE` automatically — if only one profile in a pair declares it,
the validation correctly rejects the pair.

`PairContrastResult.primaryAxis: String → DispositionAxis` (consistent with `VariantPair`):
```java
public record PairContrastResult(
    String profileHigh,
    String profileLow,
    DispositionAxis primaryAxis,   // was String
    RenderFormat format,
    boolean correctlyIdentified,
    int effectSize,
    String reasoning
) { ... }
```

`PairContrastJudge.AXIS_DESCRIPTIONS` (currently a reference to
`VocabularyExpressivenessJudge.AXIS_DESCRIPTIONS`) is **deleted**. Usage becomes
`pair.primaryAxis().description()`.

### 3d. `TraitExpressionJudge` — typed axes, typed result maps

`NUMERIC_AXES: List<String>` → `List<DispositionAxis>`:

```java
static final List<DispositionAxis> NUMERIC_AXES = List.of(
    SOCIAL_ORIENTATION, RULE_FOLLOWING, RISK_APPETITE, AUTONOMY
);
```

**Not** `DispositionAxis.values()` — the judge's system prompt asks for exactly these four
axes and does not ask for `CONFLICT_MODE`. Iterating `values()` would cause a null lookup for
`CONFLICT_MODE` in the JSON response and throw `MalformedJudgeResponseException`.

JSON parsing uses `axis.jsonKey()`:
```java
for (final DispositionAxis axis : NUMERIC_AXES) {
    final JsonNode n = root.get(axis.jsonKey());
    if (n == null) throw new MalformedJudgeResponseException("Missing axis: " + axis);
    scores.put(axis, n.asInt());
}
```

`TraitExpressionResult` result maps become typed:
```java
public record TraitExpressionResult(
    ProfiledEvalCase evalCase,
    RenderFormat format,
    Map<DispositionAxis, Integer> expressionScores,   // was Map<String, Integer>
    Map<DispositionAxis, Boolean> directionMatches,   // was Map<String, Boolean>
    String delegationAssessment
) {}
```

`computeMatches()` and `runReliabilityCheck()` in `PromptEvalTest` both iterate
`NUMERIC_AXES` with `DispositionAxis` keys. `runReliabilityCheck()` specifically:
```java
for (final DispositionAxis axis : TraitExpressionJudge.NUMERIC_AXES) {
    final int s1 = r1.expressionScores().getOrDefault(axis, 3);
    final int s2 = r2.expressionScores().getOrDefault(axis, 3);
```

`AgentProfile.expectedTraits: Map<String, TraitPolarity> → Map<DispositionAxis, TraitPolarity>`.
Profile YAML `expectedTraits` keys change from camelCase (`riskAppetite: HIGH`) to enum names
(`RISK_APPETITE: HIGH`). Jackson deserializes by enum name automatically.

### 3e. `VocabularyExpressivenessJudge` — typed axes, remove `AXIS_DESCRIPTIONS`

`AXES: List<String>` → `List<DispositionAxis>`:
```java
static final List<DispositionAxis> AXES = List.of(
    SOCIAL_ORIENTATION, RULE_FOLLOWING, RISK_APPETITE, AUTONOMY
);
```

Again, not `values()` — the judge only scores these four axes.

`AXIS_DESCRIPTIONS: Map<String, String>` is **deleted**. All three judges that previously
referenced it (`VocabularyExpressivenessJudge`, `PairContrastJudge`, `BehavioralJudge`) use
`axis.description()` directly.

---

## 4. Phase 1 — Structural Baseline

**Goal:** confirm all three format summaries are structurally complete; calibrate score floors.

Steps:
1. Wire deps (Section 1), configure Claude CLI model, build.
2. Run `evaluateAllScenarios()` — synthetic cases, no renderer `ChatModel`.
3. Inspect `target/eval-report.json` per format: `allCasesComplete`, mean scores per dimension.
4. Calibrate `SCORE_FLOORS` in `PromptEvalTest`: `max(3.0, observed_mean − 0.5)` per format.
5. Commit `target/eval-report.json` as a baseline artifact alongside the floor calibration commit.

No new types. Only change: updated `SCORE_FLOORS` constants in `PromptEvalTest`.

---

## 5. Phase 2 — Real-World Scenarios

**Goal:** run all five judges over the eight real-world profiles; validate pair contrast and
vocabulary expressiveness.

Steps:
1. Run `evaluateRealWorldScenarios()`.
2. Inspect three report files:
   - `target/real-world-eval-report.json` — quality scores per format
   - `target/proximity-report.json` — MARKDOWN vs PROSE consistency
   - `target/personality-preservation-report.json` — expressiveness, trait expression,
     pair contrast
3. Key questions:
   - Do pair contrast results correctly identify the higher-axis profile (`correctlyIdentified == true`)?
   - Is `effectSize >= 3` for primary axis pairs?
   - Do vocabulary expressiveness results show framework terms in output?
   - Are reliability warnings generated (judge variance >= 1 on same case)?
4. Calibrate `PROXIMITY_FLOOR` if necessary (same `max(3.0, observed − 0.5)` rule).

No new types.

---

## 6. Phase 3 — Behavioral Validation

### 6.1 Design rationale

Per-profile axis scoring is circular: the same model that answers a question under a "bold"
system prompt then scores its own output against known expected values. This is unfalsifiable —
a bold-framed question produces a bold-sounding answer regardless of whether the system prompt
had any behavioral effect.

The correct measure is **pair-contrast behavioral testing**: given two agents running under
different system prompts, can a blind judge detect who answered more boldly when answering the
same question? This controls for question bias and eliminates the loaded-judge circularity.
The judge sees two real responses and picks which expresses the axis more strongly — it does
not know which profile is "expected" to be higher.

This structure directly extends Phase 2's `PairContrastJudge` from rendered prompts to actual
agent responses.

### 6.2 Data model

`VariantPair.scenarioQuestions: List<String>` carries the scenario questions per pair
(Section 3b). Pairs with empty `scenarioQuestions` are silently skipped in Phase 3.
`AgentProfile` is not modified.

**`BehavioralPairResult`** (record, `eval/src/main/java/`):
```java
public record BehavioralPairResult(
    VariantPair pair,
    String question,
    String higherResponse,       // response from pair.higher() agent
    String lowerResponse,        // response from pair.lower() agent
    boolean correct,             // true if judge identified higherResponse as more axis-expressive
    int effectSize,              // 1–5, same scale as PairContrastJudge
    String reasoning
) {}
```

**`BehavioralReport`** (record, `eval/src/main/java/`):
```java
public record BehavioralReport(
    Instant timestamp,
    String modelLabel,           // from @ConfigProperty("casehub.eval.model.label")
    List<BehavioralPairResult> results,
    double accuracy              // fraction of results where correct == true
) {}
```

Written to `target/behavioral-report.json`.

### 6.3 Accuracy calibration

Phase 3 follows the same calibration pattern as Phase 1. The 0.8 accuracy target is
aspirational — it is **not** a first-run assertion.

First run:
1. Run `evaluateBehavioralScenarios()`, record `accuracy` in `target/behavioral-report.json`.
2. Set `ACCURACY_FLOOR = max(0.5, observed_accuracy − 0.1)` in `PromptEvalTest`.
3. Commit the floor. Assert `>= ACCURACY_FLOOR` going forward.

**Statistical note:** 6 data points total (2 pairs × 3 questions) is insufficient for
statistical claims. The assertion is a directional sanity check that the system prompt has some
differentiating effect — not a statistically powered threshold.

**Known limitation — position bias:** `higherResponse` is always passed as Response A,
`lowerResponse` as Response B. If the judge model has a systematic preference for "A" or "B",
accuracy is artificially inflated or deflated. Shared with Phase 2's `PairContrastJudge`.
Future Phase 4: randomize positions per trial and measure bias separately.

### 6.4 BehavioralJudge

**Location:** `eval/src/main/java/io/casehub/eidos/eval/BehavioralJudge.java`  
**Annotations:** `@ApplicationScoped`

**Injection:** `ChatModel chatModel` — used for both target invocations and the judge call.
In Claude mode, `AgentProviderChatModel` routes all three calls through Claude CLI. In Jlama
mode, Jlama handles all three.

**Judge call** — `evaluate(VariantPair pair, String question, String higherResponse, String lowerResponse) → BehavioralPairResult`:

```
You are comparing two AI agent responses to the same question.

Axis being assessed: [pair.primaryAxis().description()]

Question: <question>

Response A: <higherResponse>

Response B: <lowerResponse>

Which response expresses the axis value more strongly?

Effect size (1-5):
- 5 = unmistakably different; a reader could identify which without knowing the axis
- 3 = distinguishable if you are looking for it
- 1 = practically indistinguishable on this axis

Return JSON: { "higher": "A" | "B", "effectSize": int, "reasoning": string }
```

`ResponseFormat` omitted — same reasoning as existing judges in Claude mode.

`correct = "A".equals(judgeAnswer)` (Response A = higher profile's response, consistent with
`PairContrastJudge`).

### 6.5 Test method

`evaluateBehavioralScenarios()` is a new `@Test @Tag("eval")` method in `PromptEvalTest`.

**New field** in `PromptEvalTest` (does not exist today):
```java
@Inject
ChatModel chatModel;

@Inject
BehavioralJudge behavioralJudge;
```

**Renders are always re-computed** — JUnit 5 test methods are isolated; the `renders` map from
`evaluateRealWorldScenarios()` is not accessible from another test method.

Method outline:
```
1. Filter variantIndex.variants() to pairs with non-empty scenarioQuestions
2. For each pair:
   a. Build ProfiledEvalCase for pair.higher() at MARKDOWN (from realWorldCases)
   b. Build ProfiledEvalCase for pair.lower() at MARKDOWN
   c. Render both: renderer.render(descriptor, context) → RenderedPrompt
   d. For each question in pair.scenarioQuestions():
      i.  chatModel.chat(systemPrompt=higherRender.content(), userMessage=question) → higherResponse
      ii. chatModel.chat(systemPrompt=lowerRender.content(), userMessage=question) → lowerResponse
      iii. behavioralJudge.evaluate(pair, question, higherResponse, lowerResponse) → BehavioralPairResult
3. Build BehavioralReport(timestamp, modelLabel, results, accuracy)
4. Write to target/behavioral-report.json
5. Print summary table: pair | question | correct | effectSize
6. Assert report.accuracy() >= ACCURACY_FLOOR  (set after first run; see Section 6.3)
```

`chatModel.chat(systemPrompt, userMessage)` is the target invocation — `AgentProviderChatModel`
bridges this to `AgentSessionConfig.of(renderedPrompt, question)` in Claude mode, and Jlama
handles it natively in Jlama mode.

---

## 7. Report Outputs (complete list)

| File | Phase | Contents |
|------|-------|----------|
| `target/eval-report.json` | 1 | Synthetic quality scores, calibrated floors |
| `target/real-world-eval-report.json` | 2 | Real-world quality scores per format |
| `target/proximity-report.json` | 2 | MARKDOWN vs PROSE consistency |
| `target/personality-preservation-report.json` | 2 | Expressiveness, trait expression, pair contrast |
| `target/behavioral-report.json` | 3 | Pair-contrast behavioral results, accuracy |

---

## 8. Success Criteria

- **Phase 1:** `target/eval-report.json` exists; floors calibrated; all three format summaries
  `allCasesComplete == true`
- **Phase 2:** All five judge reports generated; pair contrast `correctlyIdentified == true` for
  primary axis; vocabulary expressiveness shows framework term names in output
- **Phase 3:** `BehavioralJudge` implemented; `evaluateBehavioralScenarios()` runs; Claude run
  `accuracy >= ACCURACY_FLOOR` (calibrated after first run); Jlama run completes (accuracy
  observed, not asserted until baseline established)

---

## 9. Out of Scope

- Cross-model judging (Claude judges Jlama responses and vice versa) — future Phase 4
- Position bias measurement (randomized A/B assignment) — future Phase 4
- Persistent baseline storage beyond git-committed JSON files
- Streaming report output during eval run
- A2A_CARD format for behavioral eval (MARKDOWN only)
- Behavioral eval for profiles not part of a variant pair (no control group exists)
