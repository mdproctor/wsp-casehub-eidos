---
layout: post
title: "The Field That Was Already Missing"
date: 2026-07-05
type: phase-update
entry_type: note
subtype: diary
projects: [eidos]
tags: [capability, rendering, degradation]
---

`AgentCapability` had a name and it had routing signals — `qualityHint`, `latencyHintP50Ms`, `costHint`, `epistemicDomains`. What it didn't have was a way to say what the capability actually *does*. The name `code-review` tells you the category; it doesn't tell you "reviews Java code for correctness, style violations, and concurrency bugs." That second sentence is what an LLM routing prompt needs when it's deciding which agent to dispatch.

The fix is a single nullable `String description` field, but threading it through the system surfaces an interesting rendering question. Protocol PP-20260611-228599 draws a hard line: numeric routing signals appear only in A2A_CARD format (they're engine dispatch data, not behavioural instructions for the agent). But `description` is human-readable text — it belongs in all three render formats, same as the capability name itself. MARKDOWN gets `**code-review** — Reviews code for quality`. PROSE gets `code-review (Reviews code for quality)`. A2A_CARD gets the full JSON field.

The A2A_CARD path had a subtlety worth getting right. The `A2ASemanticEnrichmentStep` already generates LLM-enriched capability descriptions when a `ChatModel` is available. With a declared `description` as fallback, there are now two sources. The rule: enriched wins when non-blank, declared is the fallback. This matches the existing pattern — disposition narratives work the same way. A design review caught that the original spec didn't handle the case where enrichment produces an empty string. The blank-check fallback landed before implementation started.

The same review caught a test isolation bug I wouldn't have noticed until it bit someone: `InMemoryAgentStateStore` is `@ApplicationScoped`, so its `ConcurrentHashMap` persists across JUnit test methods. Without an explicit `stateStore.clear()` in `@BeforeEach`, test 3 (which records degradation, then clears it) would leave state that poisons test 4.

The degradation example itself is straightforward — five tests showing an agent cycling through Ready, Degraded (various reasons), and back to Ready via explicit clear or TTL expiry. The interesting design choice is that the test injects the `AgentStateStore` and `CapabilityHealth` SPIs directly, never a concrete implementation. Whether the backing store is in-memory or JPA is pure configuration. The example demonstrates the contract, not the implementation.
