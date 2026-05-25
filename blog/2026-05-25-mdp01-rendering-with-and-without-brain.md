---
layout: post
title: "Rendering with and without a Brain"
date: 2026-05-25
type: phase-update
entry_type: note
subtype: diary
projects: [eidos]
tags: [renderer, langchain4j, tdd, capability-health]
---

Phase 3 is done. 121 tests, all green.

## The SPI That Finally Made Sense

The old `render(AgentDescriptor, String goal, RenderContext)` signature was wrong
from the start — I knew it during design but hadn't articulated why until we sat
down to implement it. Goals aren't static. An agent may have no goal when
registered, get a case assigned later, and accumulate sub-goals as plan items are
dispatched. Passing a flat string at call time treats the goal as a fixed input.

The fix: `AgentPromptContext` — a single immutable record with builder-style
wither methods that accumulates everything dynamic: `Optional<GoalContext>`,
`List<Resource>`, `situationalContext`, `RenderFormat`. Re-renderable at any point.
Breaking change, zero callers, free.

## Two Paths, One Renderer

`ClaudeMarkdownRenderer` does two things depending on whether a `ChatModel` is
injected:

With one: serializes the descriptor and context to single-quoted YAML, sends it
to `ChatModel.chat(prompt)` with the instruction "use whatever structure is
clearest and most concise — optimise for LLM consumption, not human readability",
and returns the response as the prompt content.

Without one: renders properties and lists in markdown. No generated sentences.
Valid output that an LLM consumer can work from immediately.

The LLM path hit two surprises on contact with LangChain4j 1.14.1. First:
`ChatLanguageModel` is gone — the interface is now `ChatModel` in a different
package. `cannot find symbol: class ChatLanguageModel` tells you nothing about
a rename. Second: `ChatModel` has no abstract methods. Every `chat()` overload
is `default`; the extension point is `doChat(ChatRequest)`, also `default` but
throws `RuntimeException("Not implemented")`. Lambda stubs fail at compile time.

```java
// Fails — ChatModel is not a functional interface
ChatModel stub = request -> ChatResponse.builder()...;

// Works — override the real extension point
ChatModel stub = new ChatModel() {
    @Override public ChatResponse doChat(ChatRequest request) {
        return ChatResponse.builder().aiMessage(AiMessage.from("ok")).build();
    }
};
```

Also: `UserMessage.text()` doesn't exist. It's `singleText()`.

## AgentStateStore: Dead Simple, Correctly Placed

`DefaultCapabilityHealth` previously had no way to return `Degraded` — it only
saw the static descriptor. The fix is an `AgentStateStore` SPI: `record()`,
`query()`, `clear()`, with a TTL `Instant`. `NoOpAgentStateStore @DefaultBean`
preserves existing behaviour. `InMemoryAgentStateStore @Alternative @Priority(1)`
in `casehub-eidos-memory` activates by classpath.

Probe order: degraded check first, then unavailable, then epistemically weak,
then ready. Fail-fast — no point checking capabilities for an agent known
to be rate-limited.

## Three Bugs Claude Caught

The code review found three real defects before the commit.

`toYaml()` had `descriptor.modelFamily() + "/" + descriptor.modelVersion()` under
an `||` condition — so with `modelFamily = null` the YAML got `null/claude-3-7-sonnet`.
The LLM would have received it silently.

`GoalContext` is a plain record. `new GoalContext("review", null, null)` is valid
Java, and both the YAML serializer and the structural renderer called `subGoals.isEmpty()`
without null-checking. Compact constructor added: `subGoals = subGoals != null ? subGoals : List.of()`.

The slot section used the raw slot string as the heading and the vocabulary
term *label* as the body — the spec says use the vocabulary *label* as the
heading and the vocabulary *description* as the body. Correct order, wrong fields.

None of these would have been caught by the tests as written. The structural
tests used an empty `CdiVocabularyRegistry`, so the vocabulary expansion path
was never exercised. That gap is now in eidos#9.
