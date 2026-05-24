# Phase 3: SystemPromptRenderer + AgentStateStore Design

**Date:** 2026-05-24  
**Issue:** casehubio/eidos#5  
**Branch:** issue-5-system-prompt-renderer

---

## Scope

Two deliverables:

1. **SystemPromptRenderer + ClaudeMarkdownRenderer** — render `AgentDescriptor` and dynamic agent context into a format-specific system prompt. `ClaudeMarkdownRenderer` is the `@DefaultBean` target. Implements a two-step pipeline: structural YAML serialization → optional LLM semantic pass → markdown assembly.

2. **AgentStateStore + Degraded probing** — SPI for recording and querying runtime degradation state (rate limits, overload, context exhaustion). `DefaultCapabilityHealth` checks it at probe time. Follows the standard persistence-backend CDI priority pattern.

---

## API Changes (`casehub-eidos-api`)

### New top-level types

**`DegradationReason`** — promoted from `CapabilityHealth` inner enum to top-level:

```java
public enum DegradationReason {
    RATE_LIMITED, CONTEXT_EXHAUSTED, OVERLOADED, DOMAIN_MISMATCH
}
```

**`GoalContext`** — structured goal with optional sub-goals and case reference:

```java
public record GoalContext(
    String description,
    List<String> subGoals,
    String caseRef         // opaque case reference, null when not case-bound
) {
    public static GoalContext of(String description) {
        return new GoalContext(description, List.of(), null);
    }
}
```

**`Resource`** — what the agent has access to (files, folders, APIs, MCP tools, etc.):

```java
public record Resource(String uri, String label, String type) {}
```

**`AgentPromptContext`** — single accumulation point for everything dynamic at render time. Re-renderable as goals, resources, and situational context evolve:

```java
public record AgentPromptContext(
    Optional<GoalContext> goal,
    List<Resource> resources,
    String situationalContext,
    SystemPromptRenderer.RenderFormat format
) {
    public static AgentPromptContext forFormat(SystemPromptRenderer.RenderFormat format) {
        return new AgentPromptContext(Optional.empty(), List.of(), null, format);
    }
    public AgentPromptContext withGoal(GoalContext goal) {
        return new AgentPromptContext(Optional.of(goal), resources, situationalContext, format);
    }
    public AgentPromptContext withResources(List<Resource> resources) {
        return new AgentPromptContext(goal, resources, situationalContext, format);
    }
    public AgentPromptContext withSituationalContext(String ctx) {
        return new AgentPromptContext(goal, resources, ctx, format);
    }
}
```

**`AgentStateStore`** — SPI for recording and querying agent degradation state:

```java
public interface AgentStateStore {
    void record(String agentId, DegradationReason reason, Instant expiresAt);
    Optional<DegradationReason> query(String agentId);
    void clear(String agentId);
}
```

### Modified types

**`CapabilityHealth`** — remove inner `DegradationReason` enum; reference top-level `DegradationReason` in `Degraded` record. No other changes.

**`SystemPromptRenderer`** — signature changes from `render(AgentDescriptor, String goal, RenderContext)` to:

```java
public interface SystemPromptRenderer {
    RenderedPrompt render(AgentDescriptor descriptor, AgentPromptContext context);

    enum RenderFormat { CLAUDE_MD, OPENAI_SYSTEM, A2A_CARD, GEMINI }

    record RenderedPrompt(String content, RenderFormat format,
                          String descriptorHash, String contextHash) {}
}
```

`RenderContext` is removed; `RenderFormat` and `RenderedPrompt` remain nested inside the SPI. `descriptorHash` and `contextHash` are SHA-256 of the respective records' `toString()` — stable and deterministic for cache invalidation. Regenerate when either hash changes.

---

## AgentStateStore Implementations

### `NoOpAgentStateStore` (runtime, `@DefaultBean`)

```java
@DefaultBean @ApplicationScoped
public class NoOpAgentStateStore implements AgentStateStore {
    @Override public void record(String agentId, DegradationReason reason, Instant expiresAt) {}
    @Override public Optional<DegradationReason> query(String agentId) { return Optional.empty(); }
    @Override public void clear(String agentId) {}
}
```

System behaviour with no-op: identical to today. Probe never returns `Degraded`.

### `InMemoryAgentStateStore` (casehub-eidos-memory, `@Alternative @Priority(1)`)

ConcurrentHashMap with TTL expiry. Activate by adding `casehub-eidos-memory` to the classpath — no consumer configuration changes.

```java
@Alternative @Priority(1) @ApplicationScoped
public class InMemoryAgentStateStore implements AgentStateStore {
    private final ConcurrentHashMap<String, ExpiringState> store = new ConcurrentHashMap<>();

    @Override
    public void record(String agentId, DegradationReason reason, Instant expiresAt) {
        store.put(agentId, new ExpiringState(reason, expiresAt));
    }

    @Override
    public Optional<DegradationReason> query(String agentId) {
        return Optional.ofNullable(store.get(agentId))
            .filter(s -> Instant.now().isBefore(s.expiresAt()))
            .map(ExpiringState::reason);
    }

    @Override public void clear(String agentId) { store.remove(agentId); }

    private record ExpiringState(DegradationReason reason, Instant expiresAt) {}
}
```

JPA persistence for `AgentStateStore` is deferred to a later phase (see eidos#7).

---

## DefaultCapabilityHealth Enhancement

Inject `AgentStateStore`. Probe order (fail-fast):

1. `agentStateStore.query(descriptor.agentId())` → if present → `Degraded(reason, "recorded at dispatch time")`
2. Capability declared? → `Unavailable` if not
3. Epistemic domain confidence below threshold? → `EpistemicallyWeak`
4. → `Ready`

State store check runs first — a known-degraded agent is not worth checking further.

`DefaultReactiveCapabilityHealth` receives the same change.

---

## ClaudeMarkdownRenderer

### Pipeline

```
AgentDescriptor + AgentPromptContext
        ↓
  serialize() → YAML string
        ↓  ← optional (ChatLanguageModel injected)
  llm.chat(prompt + YAML) → prose string
        ↓
  assemble markdown sections → RenderedPrompt
```

The semantic LLM pass is optional. When `ChatLanguageModel` is not available, the renderer falls back to structural-only output — properties, lists, no generated sentences. Structural output is valid and immediately usable by an LLM consumer.

### LangChain4j

Uses `dev.langchain4j:langchain4j-core` (pure Java, lightweight — interfaces only). Added as a compile dependency to `casehub-eidos` (runtime). Does not appear in any SPI signature.

`ClaudeMarkdownRenderer` accepts `ChatLanguageModel` as a constructor parameter, making it testable with a pure Java mock:

```java
// CDI wiring
@DefaultBean @ApplicationScoped
public class ClaudeMarkdownRenderer implements SystemPromptRenderer {
    private final ChatLanguageModel llm;       // null when not configured
    private final VocabularyRegistry vocab;

    @Inject
    public ClaudeMarkdownRenderer(
            @Any Instance<ChatLanguageModel> llm,
            VocabularyRegistry vocab) {
        this.llm = llm.isResolvable() ? llm.get() : null;
        this.vocab = vocab;
    }

    // Pure Java test constructor
    ClaudeMarkdownRenderer(ChatLanguageModel llm, VocabularyRegistry vocab) {
        this.llm = llm;
        this.vocab = vocab;
    }
}
```

### YAML serialization

The serialized form captures all non-null fields of `AgentDescriptor` and `AgentPromptContext`. Vocabulary terms are resolved and embedded inline when available. Example:

```yaml
agent:
  id: reviewer-1
  name: Code Reviewer
  model: claude/claude-3-7-sonnet
  provider: anthropic
  slot: reviewer
  capabilities:
    - name: code-review
      qualityHint: 0.95
      latencyHintP50Ms: 150
      epistemicDomains:
        java: 0.95
        python: 0.80
        rust: 0.30
  disposition:
    socialOrient: independent
    ruleFollowing: strict
    riskAppetite: conservative
    autonomy: directed
    canDelegate: false
context:
  goal:
    description: Review PR #342 for correctness and style
    subGoals:
      - Check for null pointer exceptions
      - Verify test coverage
    caseRef: case-abc-123
  resources:
    - uri: /src/main/java/
      label: Source code
      type: filesystem
  situationalContext: Critical release branch — no regressions acceptable
```

### LLM prompt

```
You are generating a system prompt for an LLM agent.
Below is the agent's structured definition in YAML. Generate a clear,
concise, logically coherent system prompt in prose that tells the agent
who it is, what it should do, and how to behave. Be direct and actionable.

{yaml}
```

### Structural fallback sections (no LLM)

When no `ChatLanguageModel` is available, render sections from structured data directly. Each section is omitted when content is null or empty.

```
# {name}
**Agent ID:** {agentId}  **Model:** {modelFamily}/{modelVersion}  **Provider:** {provider}

## {slot — vocabulary-expanded label if available, else raw value}
{vocabulary term description if available}

## Capabilities
- **{name}**: quality {qualityHint}, p50 {latencyHintP50Ms}ms
  Domains: {epistemicDomains as key=value list}

## Disposition
- Social orientation: {socialOrient}
- Rule following:    {ruleFollowing}
- Risk appetite:     {riskAppetite}
- Autonomy:          {autonomy}
- Can delegate:      yes | no

## Goal
{goal.description}
Sub-goals: {bulleted list if any}
Case: {caseRef if present}

## Resources
- {label}: {uri} ({type})

## Context
{situationalContext}

## Data Handling
Jurisdiction: {jurisdiction}
Policy: {dataHandlingPolicy}
```

---

## Module Structure

| Module | Changes |
|--------|---------|
| `casehub-eidos-api` | `DegradationReason`, `GoalContext`, `Resource`, `AgentPromptContext`, `AgentStateStore` (new); `CapabilityHealth`, `SystemPromptRenderer` (modified) |
| `casehub-eidos` (runtime) | `NoOpAgentStateStore`, `ClaudeMarkdownRenderer` (new); `DefaultCapabilityHealth`, `DefaultReactiveCapabilityHealth` (modified); `dev.langchain4j:langchain4j-core` dependency added |
| `casehub-eidos-memory` | `InMemoryAgentStateStore` (new) |
| `casehub-eidos-deployment` | No changes |
| `casehub-eidos-vocab` | No changes |
| `examples/agent-scenarios` | New example test for `SystemPromptRenderer` |

No Flyway migrations. No schema changes.

---

## Testing Strategy

### Pure Java unit tests (no `@QuarkusTest`)

**`ClaudeMarkdownRendererTest`** — instantiate directly with a mock `ChatLanguageModel` and mock `VocabularyRegistry`:

```java
ChatLanguageModel mockLlm = messages ->
    Response.from(AiMessage.from("You are a code reviewer specialising in Java."));
var renderer = new ClaudeMarkdownRenderer(mockLlm, mockVocab);
```

Test cases:
- LLM path: YAML serialization is correct, LLM called with expected content, response embedded in output
- Structural fallback: null `ChatLanguageModel` produces properties/lists with no generated sentences
- Empty goal: Goal section omitted
- Empty resources: Resources section omitted
- Vocabulary expansion: slot label resolved and used in section heading
- Hash stability: same inputs → same hashes; changed inputs → different hashes

**`InMemoryAgentStateStoreTest`** — pure Java:
- Record and query round-trip
- TTL expiry: expired entries return empty
- Clear removes entry
- Concurrent access (multiple threads)

### `@QuarkusTest` integration tests

**`DefaultCapabilityHealthTest`** — extend existing tests:
- Degraded takes precedence over Ready/EpistemicallyWeak
- With `NoOpAgentStateStore` (default): probe never returns Degraded
- With `InMemoryAgentStateStore` active: probe returns Degraded when state recorded and unexpired

**Example test in `agent-scenarios`** — full render scenario:
- Register an agent, assign a goal and resources, call `SystemPromptRenderer.render()`, assert output contains expected content

---

## PLATFORM.md Update

Add to Capability Ownership table:

| System prompt generation | `casehub-eidos` | `SystemPromptRenderer` SPI — `render(AgentDescriptor, AgentPromptContext)` → `RenderedPrompt`; `ClaudeMarkdownRenderer @DefaultBean` implements a two-step pipeline: structural YAML serialization → optional LangChain4j semantic LLM pass → markdown assembly. Works without LLM (structural-only output). Hashes enable cache invalidation on context change. |

Add to Capability Ownership table:

| Agent operational state (degradation tracking) | `casehub-eidos` | `AgentStateStore` SPI — `record(agentId, DegradationReason, expiresAt)`, `query(agentId)`. `NoOpAgentStateStore @DefaultBean` (no tracking). `InMemoryAgentStateStore @Alternative @Priority(1)` in `casehub-eidos-memory` (TTL-based, activate by classpath). `DefaultCapabilityHealth` checks store first at probe time. JPA persistence deferred (eidos#7). |

Update `CapabilityHealth` entry: reference top-level `DegradationReason` (no longer nested).

---

## Deferred / Related Issues

| Issue | Scope |
|-------|-------|
| eidos#6 | Semantic rendering pipeline — LLM prompt templates, YAML schema, prose quality |
| eidos#7 | Reactive parity: `AgentStateStore` + `SystemPromptRenderer` |
| eidos#8 | `EpistemicallyWeak` as soft preference signal in engine dispatch |
