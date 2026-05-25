# Design Journal — issue-5-system-prompt-renderer

## §Phase3 — SystemPromptRenderer + AgentStateStore

**Date:** 2026-05-25

### Goals are dynamic, not a static string

The original SPI had `render(AgentDescriptor, String goal, RenderContext)`. Goals in casehub align with engine cases — an agent may be case-bound (the case is the goal, plan items are sub-goals) or generalised (no case yet). Goals evolve over time as sub-tasks are assigned.

Decision: replace `String goal` + `RenderContext` with `AgentPromptContext` — a single accumulation point carrying `Optional<GoalContext>`, `List<Resource>`, `situationalContext`, and `RenderFormat`. Re-renderable at any point as the context grows. Breaking change accepted; no callers existed.

### Rendering pipeline is two-phase

The renderer has two paths:
1. **Structural** — serializes AgentDescriptor + AgentPromptContext to single-quoted YAML, renders properties and lists. No LLM call. Valid output immediately usable by an LLM consumer.
2. **Semantic (optional)** — passes YAML to `ChatModel.chat(String)` (LangChain4j 1.14.1). The LLM generates an optimised system prompt for LLM consumption (not human prose). Returns the LLM response as content.

Prompt instruction: "Generate a system prompt optimised for LLM consumption — use whatever structure is clearest and most concise. Do not optimise for human readability." This deliberately avoids biasing toward paragraphs — the LLM chooses the right format.

Semantic pass is optional — the structural output is the valid fallback. Full semantic pipeline (canonical YAML schema, prompt templates, quality evaluation) is tracked in eidos#6.

### AgentStateStore follows persistence-backend CDI pattern

`DegradationReason` was nested inside `CapabilityHealth`. Promoted to top-level because both `AgentStateStore` and `CapabilityHealth` reference it — coupling the store to the probe SPI for no architectural benefit.

`AgentStateStore` SPI: `record(agentId, reason, expiresAt)`, `query(agentId)`, `clear(agentId)`. Operational SPI → `@DefaultBean` no-op. `InMemoryAgentStateStore @Alternative @Priority(1)` in `casehub-eidos-memory`. JPA persistence deferred.

`DefaultCapabilityHealth` probe order (fail-fast): degraded state → unavailable → epistemically weak → ready. State store check runs first — a known-degraded agent is not worth checking further.

### LangChain4j 1.x API changed from 0.x

In LangChain4j 1.14.1, the chat model interface is `ChatModel` (not `ChatLanguageModel`). The call method is `chat(String)` (default, convenient) which internally builds a `ChatRequest` and calls `doChat(ChatRequest)` — the method implementations override. `UserMessage.singleText()` (not `.text()`) is the accessor for the message content. Test mocks must implement `doChat(ChatRequest)` returning `ChatResponse.builder().aiMessage(...).build()`.

### Slot heading uses vocabulary label, body uses term description

Reviewer caught that the spec says: slot label → heading, slot description → body. Initial implementation used raw slot value for heading and term label for body. Fixed.

### GoalContext.subGoals requires compact constructor guard

Plain record constructor allows `new GoalContext("review", null, null)` — both YAML serializer and structural renderer called `subGoals.isEmpty()` without null check. Added compact constructor: `subGoals = subGoals != null ? subGoals : List.of()`.

### YAML values must be single-quoted

Raw value concatenation produces invalid YAML for values containing `:`, `#`, `|`, `>`. Single-quote scalar style handles all special characters; single quotes in values are doubled (`''`).
