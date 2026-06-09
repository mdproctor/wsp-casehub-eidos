# Eval Baseline & Behavioral Validation Design
**Issue:** casehubio/eidos#46  
**Date:** 2026-06-09  
**Branch:** issue-46-eval-baseline-behavioral

---

## Goal

Run the eval harness for the first time, establish calibrated score floors, and add a
behavioral validation loop (Phase 3) that tests whether a rendered system prompt actually
changes how an agent behaves — not just how the prompt reads.

Three phases, two model backends (Claude Sonnet 4.6 via platform `AgentProvider`, Jlama via
LangChain4j), five report files.

---

## 1. AgentProviderChatModel Bridge

All five existing judges inject `ChatModel` (LangChain4j). Rather than wiring a LangChain4j
REST provider, we implement `ChatModel` on top of the platform's own `AgentProvider` SPI.

**Location:** `eval/src/test/java/io/casehub/eidos/eval/AgentProviderChatModel.java`  
(test sources — Quarkus discovers CDI beans from the test classpath in `@QuarkusTest` contexts)

**Annotations:** `@DefaultBean @ApplicationScoped` — active when no higher-priority `ChatModel`
bean is present. Jlama's extension registers its own `ChatModel @ApplicationScoped`, which beats
`@DefaultBean` automatically.

**Semantics:**
- Extracts the first `SystemMessage` → `systemPrompt`, first `UserMessage` → `userPrompt` from
  the `ChatRequest`
- Ignores `ResponseFormat`, temperature, and all other parameters (not expressible via Claude CLI)
- Calls `agentProvider.invoke(AgentSessionConfig.of(systemPrompt, userPrompt))`
- Collects `Multi<AgentEvent.TextDelta>` → joined `String` (blocking — acceptable in offline eval)
- Returns `ChatResponse.builder().aiMessage(AiMessage.from(text)).build()`

**AgentProvider injection:** `@Any Instance<AgentProvider>` — `Instance<X>` always satisfies CDI
at augmentation time. In Jlama mode the bean is suppressed (never invoked), so no runtime error
even if no real `AgentProvider` is wired. In Claude mode `ClaudeAgentProvider @ApplicationScoped`
is resolved.

**JSON reliability:** All five existing judge system prompts end with explicit "Return JSON with
keys…" instructions. Claude follows these without `ResponseFormat` schema enforcement.
`MalformedJudgeResponseException` handles any parse failure.

**pom.xml additions:**
```xml
<!-- types: AgentProvider, AgentEvent, AgentSessionConfig -->
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-agent-api</artifactId>
    <scope>compile</scope>
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

`AgentProviderChatModel @DefaultBean` is the only `ChatModel` bean. All judge calls and Phase 3
target invocations go through Claude CLI (`ClaudeAgentProvider`).

Configure `application-eval.properties`:
```properties
# Claude model selection — requires Claude CLI installed and authenticated
casehub.agent.claude.model=claude-sonnet-4-6
```

Run:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -Peval -Dgroups=eval
```

### Jlama profile (`-Peval-jlama`)

Adds `quarkus-langchain4j-jlama` to the test classpath. Jlama's extension registers a
`ChatModel @ApplicationScoped` which beats `AgentProviderChatModel @DefaultBean`. All judge
calls and Phase 3 target invocations go through Jlama in-process.

**Known issue:** `quarkus-langchain4j-jlama` fails at test bootstrap on Quarkus 3.32+ with
"Unsupported value type: [ALL-UNNAMED]" (GE-20260423-878486). Workaround: add `--add-opens`
JVM flags to the surefire configuration. If the workaround does not hold, fall back to Ollama
(`langchain4j-ollama` + local Ollama server).

Configure `application-eval.properties` (added to the Jlama profile):
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
                    <!-- exact flags TBD — identify from the actual error at first run -->
                    <argLine>--add-opens java.base/java.lang=ALL-UNNAMED</argLine>
                </configuration>
            </plugin>
        </plugins>
    </build>
</profile>
```

---

## 3. Phase 1 — Structural Baseline

**Goal:** confirm all three format summaries are structurally complete; calibrate score floors.

Steps:
1. Wire deps (Section 1) and configure `application-eval.properties`.
2. Run `evaluateAllScenarios()` — synthetic cases, no renderer `ChatModel`.
3. Inspect `target/eval-report.json` per format: `allCasesComplete`, mean scores per dimension.
4. Calibrate `SCORE_FLOORS` in `PromptEvalTest`: `max(3.0, observed_mean − 0.5)` per format.
5. Commit `target/eval-report.json` as a baseline artifact alongside the floor calibration commit.

No new types. Only change: updated `SCORE_FLOORS` constants in `PromptEvalTest`.

---

## 4. Phase 2 — Real-World Scenarios

**Goal:** run all five judges over the eight real-world profiles; validate pair contrast and
vocabulary expressiveness.

Steps:
1. Run `evaluateRealWorldScenarios()`.
2. Inspect three report files:
   - `target/real-world-eval-report.json` — quality scores per format
   - `target/proximity-report.json` — MARKDOWN vs PROSE render consistency
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

## 5. Phase 3 — Behavioral Validation

### 5.1 AgentProfile YAML extension

Add `scenarioQuestions: List<String>` to each profile YAML (3–5 questions per profile).
Questions are crafted to expose the profile's primary disposition axes. Profiles without
`scenarioQuestions` are silently skipped in Phase 3.

Example for `sw-engineer-bold.yaml`:
```yaml
scenarioQuestions:
  - "You need to add a feature to a legacy service with no tests. The deadline is tomorrow. What do you do?"
  - "A PR you're reviewing has a clever optimization that makes the code harder to read. How do you respond?"
  - "You discover a security flaw in production. The fix takes 3 days. What's your immediate action?"
```

`AgentProfile.java` gains: `List<String> scenarioQuestions()` (default `List.of()` if absent in YAML).

### 5.2 New types

**`BehavioralQuestionResult`** (record, `eval/src/main/java/`)
```
ProfiledEvalCase evalCase
String question
String targetResponse
Map<String, Integer> axisScores    // keys: riskAppetite, ruleFollowing, socialOrient,
                                   //        autonomy, conflictMode (if present in profile)
int overall
String reasoning
```

**`BehavioralProfileResult`** (record, `eval/src/main/java/`)
```
String profileName
List<BehavioralQuestionResult> questionResults
Map<String, Double> meanAxisScores   // mean per axis across all questions
double meanOverall                   // mean overall score across all questions
```

**`BehavioralReport`** (record, `eval/src/main/java/`)
```
Instant timestamp
String modelLabel                    // "claude-sonnet-4-6" or "jlama/<model-name>"
List<BehavioralProfileResult> profiles
double passRate                      // fraction of profiles with meanOverall >= 3.0
```

Written to `target/behavioral-report.json`.

### 5.3 BehavioralJudge

**Location:** `eval/src/main/java/io/casehub/eidos/eval/BehavioralJudge.java`  
**Annotations:** `@ApplicationScoped`

**Injections:**
- `ChatModel chatModel` — used for both target invocation and response scoring

In Claude mode: `AgentProviderChatModel @DefaultBean` routes both calls through Claude CLI.
In Jlama mode: Jlama's `ChatModel @ApplicationScoped` handles both — full model consistency.

**Per-question execution (two sequential ChatModel calls):**

*Target call:*
```
chatModel.chat(systemPrompt=renderedPrompt, userMessage=question) → response String
```
(`AgentProviderChatModel` bridges this to `AgentProvider.invoke()` under the hood in Claude mode.)

*Judge call (system prompt):*
```
You are assessing whether an AI agent's response expresses the personality profile described below.

Expected profile:
- Slot: <slot>
- Disposition: riskAppetite=<value>, ruleFollowing=<value>, socialOrient=<value>,
               autonomy=<value>[, conflictMode=<value>]
- Vocabulary frameworks: <Belbin/DISC/TK term names if present in descriptor>

Question asked: <question>
Agent response: <response>

Score how well the response expresses each axis (1–5):
- 5 = response strongly and naturally expresses the expected value
- 3 = neutral — axis not discernible from the response
- 1 = response contradicts the expected value

Also give an overall framework alignment score (1–5):
- 5 = unmistakably behaving as this profile would
- 3 = indistinct — could be any profile
- 1 = behaving as the opposite of this profile

Return JSON: { "riskAppetite": int, "ruleFollowing": int, "socialOrient": int,
               "autonomy": int[, "conflictMode": int], "overall": int, "reasoning": string }
```

`ResponseFormat` is omitted — both Claude CLI and Jlama use prompt-engineered JSON output.
Parse via Jackson; `MalformedJudgeResponseException` on parse failure.

**Note on same-model scoring:** In both model modes, the judge and target are the same model. For a first-principles
validation this is acceptable — the question is whether the system prompt causes any behavioural
differentiation at all, not whether an independent model perceives it. A future Phase 4 could
cross-judge (Claude judges Jlama responses and vice versa).

**`BehavioralJudge.evaluate(ProfiledEvalCase, RenderedPrompt, String question)`** returns
`BehavioralQuestionResult`.

### 5.4 Test method

`evaluateBehavioralScenarios()` in `PromptEvalTest`:

```
1. Load profiles with non-empty scenarioQuestions
2. Render MARKDOWN format for each profile (reuse existing renderer injection)
3. For each profile × question: call behavioralJudge.evaluate(case, render, question)
4. Aggregate into BehavioralProfileResult per profile
5. Build BehavioralReport(timestamp, modelLabel, profiles, passRate)
6. Write to target/behavioral-report.json
7. Print summary table: profile | meanOverall | per-axis means
8. Assert passRate >= 0.8
```

`BehavioralJudge` is `@Inject`-ed alongside the existing five judges. No changes to existing
test methods.

**Model label** is read from `AgentSessionConfig` or a `@ConfigProperty("casehub.eval.model.label")`
with default `"claude"`.

---

## 6. Report Outputs (complete list)

| File | Phase | Contents |
|------|-------|----------|
| `target/eval-report.json` | 1 | Synthetic quality scores, calibrated floors |
| `target/real-world-eval-report.json` | 2 | Real-world quality scores per format |
| `target/proximity-report.json` | 2 | MARKDOWN vs PROSE consistency |
| `target/personality-preservation-report.json` | 2 | Expressiveness, trait expression, pair contrast |
| `target/behavioral-report.json` | 3 | Axis scores per question, per profile, passRate |

---

## 7. Success Criteria (from issue)

- **Phase 1:** `target/eval-report.json` exists, floors calibrated, all three format summaries `allCasesComplete == true`
- **Phase 2:** All five judge reports generated; pair contrast `correct == true` for primary axis; vocabulary expressiveness shows framework term names in output
- **Phase 3:** `BehavioralJudge` implemented; `evaluateBehavioralScenarios()` runs; `passRate >= 0.8` across profiles for Claude run; Jlama run completes (passRate observed, not yet required to pass)

---

## 8. Out of Scope

- Cross-model judging (Claude judges Jlama responses) — future Phase 4
- Persistent baseline storage (git-committed JSON files serve this purpose for now)
- Streaming report output during eval run
- A2A_CARD format for behavioral eval (MARKDOWN only for Phase 3)
