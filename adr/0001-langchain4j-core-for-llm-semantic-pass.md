# 0001 — LLM Integration Library for SystemPromptRenderer Semantic Pass

Date: 2026-05-25
Status: Accepted

## Context and Problem Statement

`ClaudeMarkdownRenderer` implements an optional semantic pass — serializing
`AgentDescriptor + AgentPromptContext` to YAML and calling an LLM to generate
an optimised system prompt. The renderer needs an LLM abstraction that works in
a Quarkus CDI environment without forcing a specific model provider on consumers.

## Decision Drivers

* eidos is a Quarkus extension — dependencies must not bloat consumer classpaths
* The semantic pass is optional — no LLM should be required for the extension to function
* Model provider choice belongs to the consuming application, not the framework
* casehub-engine already uses LangChain4j; version consistency matters

## Considered Options

* **langchain4j-core** — pure-Java interfaces only (`ChatModel`, `ChatRequest`, `ChatResponse`)
* **quarkus-langchain4j-core** — full Quarkus extension with CDI, config properties, dev services
* **Direct Anthropic SDK** — `com.anthropic:anthropic-sdk-java`, single-provider

## Decision Outcome

Chosen option: **langchain4j-core (1.14.1)**, because it provides the `ChatModel`
interface with no framework coupling, no CDI requirements, and no provider lock-in.
Consumers add their preferred model implementation (OpenAI, Anthropic, Ollama) as a
separate dependency; eidos never needs to know which one.

### Positive Consequences

* Minimal classpath impact — interfaces only, no I/O code or model-specific deps
* Provider-neutral — any `ChatModel` implementation works, including mocks in tests
* Optional injection via `@Any Instance<ChatModel>` — structural rendering is the fallback
* Consistent with casehub-engine's LangChain4j version

### Negative Consequences / Tradeoffs

* Consumers must configure and provide a `ChatModel` bean to activate semantic rendering
* No built-in config property support — `quarkus-langchain4j` would have given that for free
* `ChatModel` must be `@ApplicationScoped` or broader — `@Dependent` beans leak
  (documented in code comment; Quarkus LangChain4j always uses `@ApplicationScoped`)

## Pros and Cons of the Options

### langchain4j-core

* ✅ Lightweight — interfaces only, no runtime overhead
* ✅ Provider-neutral — works with any model implementation
* ✅ Version already in use by casehub-engine
* ❌ Consumers must wire up the model bean themselves

### quarkus-langchain4j-core

* ✅ CDI + config property integration out of the box
* ✅ Dev services for local model in `@QuarkusTest`
* ❌ Extension lifecycle ties eidos to Quarkiverse release cadence
* ❌ Heavier — pulls in CDI producers, config handling, metrics

### Direct Anthropic SDK

* ✅ Native Anthropic types, no abstraction overhead
* ❌ Locks all consumers to a single provider
* ❌ No standard interface — consumers cannot swap models

## Links

* eidos#5 — Phase 3 implementation
* eidos#6 — Semantic rendering pipeline (future — prompt templates, quality evaluation)
