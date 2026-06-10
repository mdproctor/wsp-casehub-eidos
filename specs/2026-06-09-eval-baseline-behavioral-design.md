# Eval Baseline & Behavioral Validation Design
**Issue:** casehubio/eidos#46  
**Date:** 2026-06-09 (revised v4)  
**Branch:** issue-46-eval-baseline-behavioral

---

## Revision notes (v4)

**Jackson serialization choice — Option B (result records keep `Map<String, Integer>`)**

Option A (`@JsonValue` on `jsonKey()`) was rejected: it affects ALL Jackson serialization of
`DispositionAxis`, including YAML deserialization of `VariantPair.primaryAxis` and
`AgentProfile.expectedTraits`, which use enum names (`RISK_APPETITE`). `@JsonValue` would make
Jackson expect `"riskAppetite"` during deserialization — directly conflicting with the YAML
convention we just established. That is not a trade-off; it breaks the migration.

**The type-safety migration boundary is at computation logic, not result records:**
- Computation code iterates `List<DispositionAxis>` — typed, no string typos
- Result records keep `Map<String, Integer>` / `Map<String, Boolean>` — stable camelCase output
- Bridge: `axis.jsonKey()` wherever computation code writes into String-keyed maps

Consequences for v3:
- `TraitExpressionResult.expressionScores` and `directionMatches` revert to `Map<String, Integer>`
  and `Map<String, Boolean>` (unchanged from today)
- `PersonalityPreservationReport.AXES` migrates to `List<DispositionAxis>` with `axis.jsonKey()`
  bridge points
- `ACCURACY_FLOOR = 0.0` specified as initial value before first run
- `VocabularyExpressivenessResult.expressivenessScores: Map<String, Integer>` explicitly stated as
  unchanged

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

## 3. DispositionAxis type-safety

The migration applies to computation paths. Result records and serialized reports keep
`Map<String, ...>` with camelCase keys. `axis.jsonKey()` is the bridge wherever computation
code writes into a String-keyed structure.

### 3a. `DispositionAxis` — add `jsonKey()` and `description()` in `casehub-eidos-api`

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

`fromJsonKey(String)` is **not** added — no current consumer. `@JsonValue` is **not** applied —
it would affect YAML deserialization of fields that use enum names (`RISK_APPETITE`), breaking
the YAML convention established in Section 3b.

### 3b. `VariantPair` — `primaryAxis: String → DispositionAxis`, add `scenarioQuestions`

```java
public record VariantPair(
    DispositionAxis primaryAxis,
    String higher,
    String lower,
    List<String> scenarioQuestions   // Phase 3
) {}
```

`index.yaml` uses enum names (Jackson default enum deserialization by `.name()`):
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

**Null handling:** `AgentProfileLoader.loadIndex()` post-processes each `VariantPair` after
deserialization, replacing null `scenarioQuestions` with `List.of()`.

### 3c. `AgentProfileLoader` — delete `axisValue()`, upgrade validation loop

`axisValue(AgentDisposition, String)` is **deleted**. The validation loop uses
`agentDisposition.get(DispositionAxis) → Optional<String>` directly.

`AXES: List<String>` → **`DispositionAxis.values()`** in `validateVariantPairs()`. This is safe:
`AgentDisposition.get(CONFLICT_MODE)` returns `Optional.empty()` when absent; two
`Optional.empty()` compare equal via `Objects.equals`. Using `values()` also means the
validation automatically covers `CONFLICT_MODE` — if only one profile in a pair declares it,
the pair is correctly rejected.

### 3d. `PairContrastResult.primaryAxis: String → DispositionAxis`

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

`PairContrastJudge.AXIS_DESCRIPTIONS` (the static reference to
`VocabularyExpressivenessJudge.AXIS_DESCRIPTIONS`) is **deleted**. Usage becomes
`pair.primaryAxis().description()`.

### 3e. `AgentProfile.expectedTraits: Map<String, TraitPolarity> → Map<DispositionAxis, TraitPolarity>`

Profile YAML `expectedTraits` keys change from camelCase (`riskAppetite: HIGH`) to enum names
(`RISK_APPETITE: HIGH`). Jackson deserializes by enum name automatically.

`TraitExpressionJudge.computeMatches()` looks up `profile.expectedTraits().get(axis)` where
`axis` is now `DispositionAxis` — correct key type.

### 3f. `TraitExpressionJudge` — typed `NUMERIC_AXES`, String-keyed result maps

`NUMERIC_AXES: List<String>` → `List<DispositionAxis>`:

```java
static final List<DispositionAxis> NUMERIC_AXES = List.of(
    SOCIAL_ORIENTATION, RULE_FOLLOWING, RISK_APPETITE, AUTONOMY
);
```

**Not** `DispositionAxis.values()` — the system prompt asks for exactly these four axes.
Iterating `values()` would produce a null JSON lookup for `CONFLICT_MODE` and throw
`MalformedJudgeResponseException`.

JSON parsing bridges to String via `axis.jsonKey()`:
```java
for (final DispositionAxis axis : NUMERIC_AXES) {
    final JsonNode n = root.get(axis.jsonKey());
    if (n == null) throw new MalformedJudgeResponseException("Missing axis: " + axis);
    scores.put(axis.jsonKey(), n.asInt());   // bridge: String key for result map
}
```

**`TraitExpressionResult.expressionScores: Map<String, Integer>`** — **unchanged**. The judge
builds this map with camelCase keys via `axis.jsonKey()`.  
**`TraitExpressionResult.directionMatches: Map<String, Boolean>`** — **unchanged**. Same
camelCase key convention.

`computeMatches()` in `TraitExpressionJudge`:
```java
for (final DispositionAxis axis : NUMERIC_AXES) {
    final TraitPolarity polarity = profile.expectedTraits().get(axis);  // DispositionAxis key
    if (polarity == null) continue;
    final int score = scores.getOrDefault(axis.jsonKey(), 3);           // String key bridge
    result.put(axis.jsonKey(), switch (polarity) { ... });              // String key bridge
}
```

`runReliabilityCheck()` in `PromptEvalTest` is updated to use `NUMERIC_AXES`:
```java
for (final DispositionAxis axis : TraitExpressionJudge.NUMERIC_AXES) {
    final int s1 = r1.expressionScores().getOrDefault(axis.jsonKey(), 3);
    final int s2 = r2.expressionScores().getOrDefault(axis.jsonKey(), 3);
```

### 3g. `VocabularyExpressivenessJudge` — typed `AXES`, String-keyed result

`AXES: List<String>` → `List<DispositionAxis>` (four values, no `CONFLICT_MODE`):
```java
static final List<DispositionAxis> AXES = List.of(
    SOCIAL_ORIENTATION, RULE_FOLLOWING, RISK_APPETITE, AUTONOMY
);
```

`AXIS_DESCRIPTIONS: Map<String, String>` is **deleted**. All usages replaced by
`axis.description()`.

`evaluate()` bridges when building the result map:
```java
scores.put(axis.jsonKey(), evaluateAxis(prose, axis));  // String key for result map
```

**`VocabularyExpressivenessResult.expressivenessScores: Map<String, Integer>`** — **unchanged**.
Camelcase key convention preserved; no Jackson configuration required.

### 3h. `PersonalityPreservationReport` — typed `AXES`, bridge points in `computeDiagnoses()`

`AXES: List<String>` → `List<DispositionAxis>` (four values, no `CONFLICT_MODE`):
```java
private static final List<DispositionAxis> AXES = List.of(
    SOCIAL_ORIENTATION, RULE_FOLLOWING, RISK_APPETITE, AUTONOMY
);
```

`computeDiagnoses()` bridge points — axis is now `DispositionAxis`:

```java
for (final DispositionAxis axis : AXES) {
    // expressivenessScores is Map<String, Integer> — bridge
    final int s1 = er.expressivenessScores().getOrDefault(axis.jsonKey(), -1);

    // directionMatches is Map<String, Boolean> — bridge
    final double matchRate = profileTraits.stream()
        .mapToInt(t -> Boolean.TRUE.equals(t.directionMatches().get(axis.jsonKey())) ? 1 : 0)
        .average().orElse(-1.0);

    // expressionScores is Map<String, Integer> — bridge
    final double s2Score = profileTraits.stream()
        .mapToInt(t -> t.expressionScores().getOrDefault(axis.jsonKey(), -1))
        .filter(s -> s >= 0)
        .average().orElse(-1.0);

    // primaryAxis is now DispositionAxis — enum equality, no bridge needed
    final double s3 = contrasts.stream()
        .filter(c -> c.primaryAxis() == axis
            && (c.profileHigh().equals(er.profileName())
                || c.profileLow().equals(er.profileName())))
        .mapToInt(PairContrastResult::effectSize)
        .average().orElse(-1.0);

    // AttributionDiagnosis.axis is String — bridge
    result.add(new AttributionDiagnosis(er.profileName(), axis.jsonKey(), ...));
}
```

`AttributionDiagnosis.axis: String` — **unchanged**. It is a serialization field, not a
computation key.

This is the critical fix: `c.primaryAxis() == axis` (enum equality) replaces the previous
`c.primaryAxis().equals(axis)` (String/DispositionAxis mismatch that silently always returns
false via `Object.equals`).

---

## 4. Phase 1 — Structural Baseline

Steps:
1. Wire deps (Section 1), configure Claude CLI model, build.
2. Run `evaluateAllScenarios()`.
3. Inspect `target/eval-report.json`: `allCasesComplete`, mean scores per dimension.
4. Calibrate `SCORE_FLOORS`: `max(3.0, observed_mean − 0.5)` per format.
5. Commit `target/eval-report.json` as baseline alongside the calibration commit.

---

## 5. Phase 2 — Real-World Scenarios

Steps:
1. Run `evaluateRealWorldScenarios()`.
2. Inspect three report files: quality scores, proximity, personality preservation.
3. Key checks: pair contrast `correctlyIdentified`, `effectSize >= 3`, vocabulary expressiveness,
   reliability warnings.
4. Calibrate `PROXIMITY_FLOOR` if necessary (`max(3.0, observed − 0.5)` rule).

---

## 6. Phase 3 — Behavioral Validation

### 6.1 Design rationale

Per-profile axis scoring is circular — the same model that answers under a "bold" system prompt
then scores its own output against known expected values. The correct measure is
**pair-contrast behavioral testing**: given two agents running under different system prompts,
can a blind judge detect which answered more boldly on the same question? This controls for
question bias and eliminates the loaded-judge circularity.

### 6.2 Data model

`VariantPair.scenarioQuestions: List<String>` carries questions per pair (Section 3b). Pairs
with empty `scenarioQuestions` are silently skipped. `AgentProfile` is not modified.

**`BehavioralPairResult`** (record, `eval/src/main/java/`):
```java
public record BehavioralPairResult(
    VariantPair pair,
    String question,
    String higherResponse,
    String lowerResponse,
    boolean correct,      // true if judge identified higherResponse as more axis-expressive
    int effectSize,       // 1–5
    String reasoning
) {}
```

**`BehavioralReport`** (record, `eval/src/main/java/`):
```java
public record BehavioralReport(
    Instant timestamp,
    String modelLabel,    // from @ConfigProperty("casehub.eval.model.label")
    List<BehavioralPairResult> results,
    double accuracy       // fraction where correct == true
) {}
```

Written to `target/behavioral-report.json`.

### 6.3 Accuracy calibration

**`ACCURACY_FLOOR`** is a constant in `PromptEvalTest`. Initial value before the first run:
```java
private static final double ACCURACY_FLOOR = 0.0;
```

`0.0` means the assertion `accuracy >= ACCURACY_FLOOR` always passes on the first run,
allowing observation without failure. After the first run:
- Set `ACCURACY_FLOOR = max(0.5, observed_accuracy − 0.1)` and commit.

**Statistical note:** 6 data points (2 pairs × 3 questions) is insufficient for statistical
claims. The assertion is a directional sanity check only — not a statistically powered
threshold.

**Known limitation — position bias:** `higherResponse` is always Response A. Judge positional
preference inflates or deflates accuracy. Shared with Phase 2's `PairContrastJudge`. Future
Phase 4: randomize positions per trial and measure bias separately.

### 6.4 BehavioralJudge

**Location:** `eval/src/main/java/io/casehub/eidos/eval/BehavioralJudge.java`  
**Annotations:** `@ApplicationScoped`  
**Injection:** `ChatModel chatModel`

Judge call signature:
```java
public BehavioralPairResult evaluate(
    VariantPair pair, String question,
    String higherResponse, String lowerResponse)
```

Judge system prompt:
```
You are comparing two AI agent responses to the same question.

Axis being assessed: [pair.primaryAxis().description()]

Question: <question>

Response A: <higherResponse>

Response B: <lowerResponse>

Which response expresses the axis value more strongly?
Effect size (1-5): 5 = unmistakably different, 3 = distinguishable if looking, 1 = indistinguishable.

Return JSON: { "higher": "A" | "B", "effectSize": int, "reasoning": string }
```

`correct = "A".equals(judgeAnswer)`. `ResponseFormat` omitted — prompt-engineered JSON.

### 6.5 Test method

**New fields** in `PromptEvalTest` (do not exist today):
```java
@Inject ChatModel chatModel;
@Inject BehavioralJudge behavioralJudge;
```

`evaluateBehavioralScenarios()` is a new `@Test @Tag("eval")` method. Renders are always
re-computed — JUnit 5 test methods are isolated; the `renders` map from
`evaluateRealWorldScenarios()` is a local variable and inaccessible from another test.

Method outline:
```
1. Filter variantIndex.variants() to pairs with non-empty scenarioQuestions
2. For each pair:
   a. Build ProfiledEvalCase for pair.higher() at MARKDOWN (from realWorldCases)
   b. Build ProfiledEvalCase for pair.lower() at MARKDOWN
   c. Render both: renderer.render(descriptor, context) → RenderedPrompt
   d. For each question in pair.scenarioQuestions():
      i.  chatModel target call: systemPrompt=higherRender.content(), userMessage=question → higherResponse
      ii. chatModel target call: systemPrompt=lowerRender.content(), userMessage=question → lowerResponse
      iii. behavioralJudge.evaluate(pair, question, higherResponse, lowerResponse)
3. Build BehavioralReport(timestamp, modelLabel, results, accuracy)
4. Write to target/behavioral-report.json
5. Print summary: pair | question | correct | effectSize
6. Assert report.accuracy() >= ACCURACY_FLOOR
```

---

## 7. Report Outputs

| File | Phase | Contents |
|------|-------|----------|
| `target/eval-report.json` | 1 | Synthetic quality scores, calibrated floors |
| `target/real-world-eval-report.json` | 2 | Real-world quality scores per format |
| `target/proximity-report.json` | 2 | MARKDOWN vs PROSE consistency |
| `target/personality-preservation-report.json` | 2 | Expressiveness, trait expression, pair contrast |
| `target/behavioral-report.json` | 3 | Pair-contrast behavioral results, accuracy |

---

## 8. Success Criteria

- **Phase 1:** `target/eval-report.json` exists; floors calibrated; all format summaries
  `allCasesComplete == true`
- **Phase 2:** All five judge reports generated; pair contrast `correctlyIdentified == true`;
  vocabulary expressiveness shows framework term names
- **Phase 3:** `BehavioralJudge` implemented; Claude run `accuracy >= ACCURACY_FLOOR` (calibrated
  post first run); Jlama run completes (accuracy observed, not asserted until baseline set)

---

## 9. Out of Scope

- Cross-model judging (Phase 4)
- Position bias measurement / randomized A/B assignment (Phase 4)
- Persistent baseline storage beyond git-committed JSON files
- A2A_CARD format for behavioral eval (MARKDOWN only)
- Behavioral eval for profiles not in a variant pair
