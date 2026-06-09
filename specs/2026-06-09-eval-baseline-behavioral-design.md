# Eval Baseline & Behavioral Validation Design
**Issue:** casehubio/eidos#46  
**Date:** 2026-06-09 (revised after review)  
**Branch:** issue-46-eval-baseline-behavioral

---

## Revision notes

- Phase 3 redesigned: per-profile axis scoring replaced with pair-contrast behavioral testing.
  Eliminates the circularity of a loaded judge scoring its own output against known expected
  values. The methodological fix also simplifies the data model.
- `casehub.agent.claude.model` property removed (does not exist in `ClaudeAgentProperties`).
- `casehub-platform-agent-api` scope corrected to `test`.
- `ResponseFormat` loss in Claude mode explicitly documented for all five existing judges.
- `DispositionAxis` type-safety applied throughout: `VariantPair.primaryAxis`,
  `AgentProfile.expectedTraits`, and `AXIS_DESCRIPTIONS` all become strongly typed.
  `DispositionAxis.jsonKey()` added to `casehub-eidos-api`.

---

## 1. AgentProviderChatModel Bridge

All five existing judges inject `ChatModel` (LangChain4j). Rather than wiring a LangChain4j
REST provider, we implement `ChatModel` on top of the platform's own `AgentProvider` SPI.

**Location:** `eval/src/test/java/io/casehub/eidos/eval/AgentProviderChatModel.java`  
(test sources — Quarkus discovers CDI beans from the test classpath in `@QuarkusTest` contexts)

**Annotations:** `@DefaultBean @ApplicationScoped` — active when no higher-priority `ChatModel`
bean is present. Jlama's extension registers `ChatModel @ApplicationScoped`, which beats
`@DefaultBean` automatically; no profile switching needed.

**Semantics:**
- Extracts the first `SystemMessage` → `systemPrompt`, first `UserMessage` → `userPrompt`
  from the `ChatRequest`
- Ignores `ResponseFormat`, temperature, and all other parameters (not expressible via Claude CLI)
- `ResponseFormat` is silently discarded for ALL five existing judges and `BehavioralJudge`
  when running in Claude mode. JSON structure relies entirely on prompt-engineered instructions
  in each judge's system prompt. `MalformedJudgeResponseException` is the only safety net.
- Calls `agentProvider.invoke(AgentSessionConfig.of(systemPrompt, userPrompt))`
- Collects `Multi<AgentEvent.TextDelta>` → joined `String` (blocking — acceptable in offline eval)
- Returns `ChatResponse.builder().aiMessage(AiMessage.from(text)).build()`

**AgentProvider injection:** `@Any Instance<AgentProvider>` — always satisfies CDI at
augmentation time (an empty `Instance` is valid). In Jlama mode the bean is suppressed (never
invoked). In Claude mode `ClaudeAgentProvider @ApplicationScoped` is resolved.

**pom.xml additions:**
```xml
<!-- types: AgentProvider, AgentEvent, AgentSessionConfig — test scope (used in test sources only) -->
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-agent-api</artifactId>
    <scope>test</scope>
</dependency>
<!-- impl: ClaudeAgentProvider @ApplicationScoped (activates by classpath presence) -->
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
# Claude model is configured via the Claude CLI itself, not a Quarkus property.
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
[ALL-UNNAMED]" (GE-20260423-878486). Attempt with `--add-opens` JVM flags in surefire
(exact flags determined at first run from the actual error). Fall back to Ollama
(`langchain4j-ollama`) if the workaround does not hold on 3.32.2.

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
                    <!-- exact flags determined at first run from the "Unsupported value type: [ALL-UNNAMED]" error -->
                    <argLine>--add-opens java.base/java.lang=ALL-UNNAMED</argLine>
                </configuration>
            </plugin>
        </plugins>
    </build>
</profile>
```

---

## 3. DispositionAxis type-safety (applies across casehub-eidos-api and eval)

The eval module currently uses camelCase `String` keys (`"riskAppetite"`, `"ruleFollowing"`,
etc.) in three places where `DispositionAxis` is the correct type. This migration fixes all
three simultaneously with the Phase 3 work.

### 3a. `DispositionAxis.jsonKey()` — new method in `casehub-eidos-api`

`AgentDisposition`'s record fields use camelCase names. `DispositionAxis` enum constants use
SCREAMING_SNAKE. Rather than duplicating the mapping as string literals across judge classes,
add a canonical `jsonKey()` method:

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
```

This makes the camelCase-to-enum mapping authoritative and eliminates the scattered string
constants. All LLM JSON key parsing uses `axis.jsonKey()`. Add a corresponding static factory
`DispositionAxis.fromJsonKey(String)` for the reverse direction.

### 3b. `VariantPair.primaryAxis`: `String` → `DispositionAxis`

```java
public record VariantPair(
    DispositionAxis primaryAxis,
    String higher,
    String lower,
    List<String> scenarioQuestions   // Phase 3 addition — see Section 5
) {}
```

`index.yaml` changes from camelCase to enum names:
```yaml
variants:
  - primaryAxis: RISK_APPETITE
    higher: sw-engineer-bold
    lower: sw-engineer-careful
    scenarioQuestions:
      - "..."
  - primaryAxis: RULE_FOLLOWING
    higher: security-analyst-defensive
    lower: security-analyst-proactive
    scenarioQuestions:
      - "..."
```

Jackson YAML deserializes `RISK_APPETITE` → `DispositionAxis.RISK_APPETITE` automatically
(default enum deserialization by name).

`AgentProfileLoader.axisValue(AgentDisposition, String)` is deleted. The validation loop
uses `agentDisposition.get(pair.primaryAxis())` — already supported by
`AgentDisposition.get(DispositionAxis)` which returns `Optional<String>`.

### 3c. `AgentProfile.expectedTraits`: `Map<String, TraitPolarity>` → `Map<DispositionAxis, TraitPolarity>`

Profile YAML `expectedTraits` keys change from `riskAppetite: HIGH` to `RISK_APPETITE: HIGH`.
`TraitExpressionJudge.computeMatches()` iterates `DispositionAxis.values()` instead of
`NUMERIC_AXES: List<String>`.

### 3d. `VocabularyExpressivenessJudge.AXIS_DESCRIPTIONS`: `Map<String, String>` → `Map<DispositionAxis, String>`

```java
static final Map<DispositionAxis, String> AXIS_DESCRIPTIONS = Map.of(
    SOCIAL_ORIENTATION, "how collaborative or independent...",
    RULE_FOLLOWING,     "how strictly the agent follows rules...",
    RISK_APPETITE,      "how risk-tolerant or risk-averse...",
    AUTONOMY,           "how self-directed versus directed-by-others..."
);
```

`PairContrastJudge` currently references `VocabularyExpressivenessJudge.AXIS_DESCRIPTIONS`
directly — it continues to do so; the type change is transparent to the usage
(`AXIS_DESCRIPTIONS.getOrDefault(pair.primaryAxis(), pair.primaryAxis().name())`).

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
   - `target/personality-preservation-report.json` — vocabulary expressiveness, trait
     expression, pair contrast
3. Key questions to answer:
   - Do pair contrast results correctly identify the higher-axis profile (`correct == true`)?
   - Is `effectSize >= 3` for primary axis pairs?
   - Do vocabulary expressiveness results show framework terms (Belbin, DISC, TK) in output?
   - Are reliability warnings generated (judge variance >= 1 on same case)?
4. Calibrate `PROXIMITY_FLOOR` if necessary (same `max(3.0, observed − 0.5)` rule).

No new types.

---

## 6. Phase 3 — Behavioral Validation

### 6.1 Design rationale

Per-profile axis scoring (original design) is circular: the same model that answers a question
under a "bold" system prompt then scores its own output against the known expected values. This
is unfalsifiable — a bold-framed question produces a bold-sounding answer regardless of whether
the system prompt had any effect.

The correct measure is **pair-contrast behavioral testing**: given two agents running under
different system prompts, do they respond differently to the same question in a way a blind
judge can detect? This controls for question bias (both agents answer the same question) and
eliminates the loaded-judge circularity (the judge sees two real responses and picks the one
that expresses the axis more strongly — it does not know which profile is "expected" to be
higher).

This structure directly extends Phase 2's `PairContrastJudge` from rendered prompts to actual
agent responses.

### 6.2 Data model

**`VariantPair`** gains `List<String> scenarioQuestions` (Section 3b).
Pairs without questions are silently skipped in Phase 3. `AgentProfile` is not modified.

Scenario questions per pair (in `index.yaml`):

```yaml
# sw-engineer-bold vs sw-engineer-careful — primary axis: RISK_APPETITE
scenarioQuestions:
  - "You need to deploy a critical fix to production in 2 hours. It hasn't been reviewed. What do you do?"
  - "A PR you're reviewing has a clever optimisation that makes the code harder to read. How do you respond?"
  - "You find a performance issue that takes 2 weeks to fix properly, or a quick hack that mostly solves it. What do you recommend?"

# security-analyst-defensive vs security-analyst-proactive — primary axis: RULE_FOLLOWING
scenarioQuestions:
  - "You discover a zero-day vulnerability. The vendor hasn't released a patch yet. What do you recommend?"
  - "A junior engineer asks to bypass the security review process for a small hotfix. What do you say?"
  - "Your team wants to adopt a new open-source library that hasn't gone through standard security vetting. How do you respond?"
```

**`BehavioralPairResult`** (record, `eval/src/main/java/`):
```
VariantPair pair
String question
String higherResponse       // response from pair.higher() agent
String lowerResponse        // response from pair.lower() agent
boolean correct             // true if judge identified higher as more axis-expressive
int effectSize              // 1–5, same scale as PairContrastJudge
String reasoning
```

**`BehavioralReport`** (record, `eval/src/main/java/`):
```
Instant timestamp
String modelLabel           // from @ConfigProperty("casehub.eval.model.label")
List<BehavioralPairResult> results
double accuracy             // fraction of results where correct == true
```

Written to `target/behavioral-report.json`.

### 6.3 BehavioralJudge

**Location:** `eval/src/main/java/io/casehub/eidos/eval/BehavioralJudge.java`  
**Annotations:** `@ApplicationScoped`

**Injection:** `ChatModel chatModel` — used for both target invocations (A and B) and the
judge call. In Claude mode all three calls go through `AgentProviderChatModel` → Claude CLI.
In Jlama mode all three go through Jlama.

**Target invocations** (called from the test method, not inside `BehavioralJudge`):
```java
// In PromptEvalTest.evaluateBehavioralScenarios():
RenderedPrompt higherRender = renders.get(higherCase);  // MARKDOWN, pair.higher() profile
RenderedPrompt lowerRender  = renders.get(lowerCase);   // MARKDOWN, pair.lower() profile

String higherResponse = invokeTarget(chatModel, higherRender.content(), question);
String lowerResponse  = invokeTarget(chatModel, lowerRender.content(), question);
// invokeTarget: chatModel.chat(systemPrompt=renderedContent, userMessage=question)
```

The target call signature (system prompt = full rendered prompt, user message = question)
is exactly `AgentSessionConfig.of(renderedPrompt, question)` via the bridge, and the Jlama
equivalent.

**Judge call** (`BehavioralJudge.evaluate()`):

`evaluate(VariantPair pair, String question, String higherResponse, String lowerResponse)`
→ `BehavioralPairResult`

Judge system prompt (reuses `VocabularyExpressivenessJudge.AXIS_DESCRIPTIONS` for the axis
description; same template structure as `PairContrastJudge.SYSTEM_TEMPLATE`):

```
You are comparing two AI agent responses to the same question.

Axis being assessed: [<AXIS_DESCRIPTIONS.get(pair.primaryAxis())>]

Question: <question>

Response A: <higherResponse>

Response B: <lowerResponse>

Which response expresses the axis value more strongly?

Effect size (1-5):
- 5 = unmistakably different; a reader could identify which is which without knowing the axis
- 3 = distinguishable if you are looking for it
- 1 = practically indistinguishable on this axis

Return JSON: { "higher": "A" | "B", "effectSize": int, "reasoning": string }
```

`ResponseFormat` omitted — same reasoning as existing judges in Claude mode; Jlama follows
the JSON instruction in the system prompt.

Result: `correct = "A".equals(judgeAnswer)` (Response A = higher profile's response).

### 6.4 Test method

`evaluateBehavioralScenarios()` in `PromptEvalTest` (`@Test @Tag("eval")`):

```
1. Load variantIndex (already in @BeforeAll)
2. Filter to pairs with non-empty scenarioQuestions
3. Render MARKDOWN for all eight profiles (reuse existing renders from evaluateRealWorldScenarios
   if run in the same session, or re-render here — rendering is cheap)
4. For each pair × question:
   a. Invoke chatModel with higherRenderedPrompt → higherResponse
   b. Invoke chatModel with lowerRenderedPrompt → lowerResponse
   c. behavioralJudge.evaluate(pair, question, higherResponse, lowerResponse)
      → BehavioralPairResult
5. Build BehavioralReport(timestamp, modelLabel, results, accuracy)
6. Write to target/behavioral-report.json
7. Print summary: pair | question | correct | effectSize
8. Assert: report.accuracy() >= 0.8   (Claude run only; Jlama: observed, not asserted)
```

`BehavioralJudge` is injected alongside the existing five judges. `chatModel` is the same
`@Inject ChatModel` already present in the test class.

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

## 8. Success Criteria (from issue)

- **Phase 1:** `target/eval-report.json` exists; floors calibrated; all three format summaries
  `allCasesComplete == true`
- **Phase 2:** All five judge reports generated; pair contrast `correct == true` for primary
  axis; vocabulary expressiveness shows framework term names in output
- **Phase 3:** `BehavioralJudge` implemented; `evaluateBehavioralScenarios()` runs; Claude
  run `accuracy >= 0.8`; Jlama run completes (accuracy observed, not asserted until baseline
  established)

---

## 9. Out of Scope

- Cross-model judging (Claude judges Jlama responses and vice versa) — future Phase 4
- Persistent baseline storage beyond git-committed JSON files
- Streaming report output during eval run
- A2A_CARD format for behavioral eval (MARKDOWN only — the format an LLM agent is most likely
  to receive and act on)
- Behavioral eval for profiles that are not part of a variant pair (no control exists for them)
