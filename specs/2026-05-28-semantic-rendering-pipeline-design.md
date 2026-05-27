# Semantic Rendering Pipeline — Design Spec
**Issue:** casehubio/eidos#6  
**Date:** 2026-05-28  
**Status:** Approved for implementation

---

## Problem

`ClaudeMarkdownRenderer` (Phase 3) has a two-path implementation: either the LLM
receives a hand-rolled YAML payload and returns the complete final output, or the
structural renderer builds markdown from raw descriptor fields. Both paths have
correctness problems:

- The LLM path collapses semantic enrichment and format assembly into one step,
  making format logic untestable without an LLM and making multi-format output
  impossible without per-format prompt templates.
- The structural path ignores `context.format()` and always produces CLAUDE_MD
  markdown regardless of the requested format.
- Hash inputs use Java record `toString()` — not canonical, not stable across JVM
  versions.
- The YAML payload is hand-rolled string concatenation — no escaping guarantees,
  non-deterministic with respect to vocabulary enrichment.

---

## Solution: Three-stage pipeline

```
AgentDescriptor + AgentPromptContext
        │
        ▼
┌─────────────────────────────┐
│  Stage 1: Payload Builder   │  → curated Jackson ObjectNode
│  (VocabularyRegistry used)  │    vocabulary labels alongside raw values
│                             │    tenancyId and DB id excluded
└─────────────────────────────┘
        │
        ▼
┌─────────────────────────────┐
│  Stage 2: Semantic Enrich.  │  → Optional<SemanticEnrichment>
│  SemanticEnrichmentStep     │    (empty on failure → structural path)
│  ResponseFormat + JsonSchema│
│  catch-all → structural     │
└─────────────────────────────┘
        │
        ▼
┌─────────────────────────────┐
│  Stage 3: Format Assembly   │  → RenderedPrompt
│  switch(context.format())   │    format-specific deterministic code
│  enriched or structural     │
└─────────────────────────────┘
```

The public `SystemPromptRenderer` SPI is unchanged.  
`RenderedPromptCache` SPI is new. All other types are package-private to
`io.casehub.eidos.runtime.renderer`.

---

## Components

| Component | Module | Visibility | Responsibility |
|---|---|---|---|
| `ClaudeMarkdownRenderer` | `runtime` | `@DefaultBean @ApplicationScoped` | Orchestrates all three stages |
| `SemanticEnrichmentStep` | `runtime/renderer/` | package-private | Stage 2: LLM call + parse |
| `SemanticEnrichment` | `runtime/renderer/` | package-private record | Six prose fields — intermediate type |
| `RenderedPromptCache` | `api` | public SPI | Cache lookup and store |
| `NoOpRenderedPromptCache` | `runtime` | `@DefaultBean` | No caching by default |
| `InMemoryRenderedPromptCache` | `persistence-memory` | `@Alternative @Priority(1)` | Bounded LRU, activated by classpath |

---

## Stage 1 — Canonical JSON payload

Built using Jackson `ObjectMapper` (already on the Quarkus classpath). Explicit
field-by-field construction — never `valueToTree(descriptor)`.

**Fields included:**

| Field | Payload key | Notes |
|---|---|---|
| `agentId` | `agentId` | |
| `name` | `name` | |
| `version` | `version` | |
| `provider` | `provider` | |
| `modelFamily` + `modelVersion` | `model` | Combined: `"claude/claude-3-7-sonnet"` |
| `slot` | `slot` | Raw value |
| Vocabulary-resolved slot label | `slotLabel` | Omitted if not resolved |
| Vocabulary-resolved slot description | `slotDescription` | Omitted if not resolved |
| `capabilities[]` | `capabilities` | name, qualityHint, latencyHintP50Ms, epistemicDomains |
| `disposition` | `disposition` | socialOrient, ruleFollowing, riskAppetite, autonomy, canDelegate |
| `jurisdiction` | `jurisdiction` | Omitted if null |
| `dataHandlingPolicy` | `dataHandlingPolicy` | Omitted if null |
| `context.goal` | `goal` | description, subGoals[], caseRef — omitted if absent |

**Fields explicitly excluded:**

| Field | Reason |
|---|---|
| `id` (DB UUID) | Internal — meaningless to LLM |
| `tenancyId` | Internal — privacy concern |
| `domainVocabulary` URI | Raw URI — resolved content included instead |
| `slotVocabulary` URI | Same |
| `dispositionVocabulary` URI | Same |
| `context.resources` | Rendered structurally in Stage 3 |
| `context.situationalContext` | Already natural language — Stage 3 pass-through |
| `context.format` | Stage 3 concern only |

**Vocabulary enrichment:** `VocabularyRegistry` is called in Stage 1 (before the
LLM call) so the LLM sees resolved labels and descriptions, not raw vocabulary URIs.

**Hash computation (replaces `toString()`-based hashing):**

```java
String descriptorHash = sha256(descriptorNode.toString());  // ObjectNode insertion-order stable
String contextHash    = sha256(contextNode.toString());
```

`ObjectNode.toString()` is deterministic within a JVM (insertion-order field
ordering). The same descriptor always produces the same JSON string and the same
hash.

**Internal cache key:**

```
cacheKey = descriptorHash + ":" + contextHash + ":" + TEMPLATE_VERSION
```

`TEMPLATE_VERSION` is `private static final String` in `ClaudeMarkdownRenderer`,
bumped manually when the prompt template changes. Not exposed in `RenderedPrompt`.

---

## Stage 2 — Semantic enrichment

### SemanticEnrichment record (package-private)

```java
record SemanticEnrichment(
    String identityNarrative,       // always populated
    String roleNarrative,           // always populated
    String capabilityNarrative,     // always populated
    Optional<String> dispositionNarrative,  // empty string sentinel → Optional.empty()
    Optional<String> constraintNarrative,   // empty string sentinel → Optional.empty()
    Optional<String> goalNarrative          // empty string sentinel → Optional.empty()
) {}
```

### ResponseFormat (built once at startup)

Uses the verified LangChain4j 1.14.1 `JsonObjectSchema` builder. All six fields are
marked required; optional sections use empty string `""` as the sentinel — the LLM
never needs to omit a JSON key, which avoids malformed output.

```java
ResponseFormat RESPONSE_FORMAT = ResponseFormat.builder()
    .type(JSON)
    .jsonSchema(JsonSchema.builder()
        .name("SemanticEnrichment")
        .rootElement(JsonObjectSchema.builder()
            .addStringProperty("identityNarrative",
                "Who this agent is — name, model, version context. Second person.")
            .addStringProperty("roleNarrative",
                "The agent's role and purpose. Second person.")
            .addStringProperty("capabilityNarrative",
                "What the agent can do, including domain confidence. Second person.")
            .addStringProperty("dispositionNarrative",
                "How the agent operates. Empty string if no disposition data.")
            .addStringProperty("constraintNarrative",
                "Data handling obligations. Empty string if none.")
            .addStringProperty("goalNarrative",
                "Current task and objectives. Empty string if no goal.")
            .required("identityNarrative", "roleNarrative", "capabilityNarrative",
                      "dispositionNarrative", "constraintNarrative", "goalNarrative")
            .build())
        .build())
    .build();
```

**Model compatibility:** `ResponseFormat` with `JsonSchema` is translated by
`InternalAnthropicHelper` to `AnthropicOutputConfig` (Anthropic's native structured
output, beta since November 2025). This requires Claude Sonnet 4.5 or Opus 4.1+.
Older models (Claude 3.x) produce an API-level error — caught and handled by the
fallback. `ResponseFormat.JSON` without a schema throws `UnsupportedFeatureException`
for Anthropic and must not be used.

### Prompt template (system message — constant string in renderer)

```
You are writing narrative descriptions for an AI agent's system prompt.

Given the agent definition in JSON, produce a JSON object with prose descriptions
for each field. Write in second person, addressing the agent directly.

REQUIRED FIELDS (always populate):
- identityNarrative (1-2 sentences): The agent's name, model, and version context.
- roleNarrative (1-3 sentences): The role this agent plays and its purpose.
  If slotLabel and slotDescription are present, prefer them over the raw slot value.
- capabilityNarrative (2-4 sentences): What the agent can do.
  For epistemicDomains, use natural language confidence:
    ≥ 0.7 → "strong expertise", 0.4–0.69 → "working knowledge", < 0.4 → "limited familiarity".

OPTIONAL FIELDS (use empty string "" if the source data is absent):
- dispositionNarrative (1-2 sentences): How the agent operates — autonomy,
  rule-following orientation, delegation authority.
- constraintNarrative (1-2 sentences): Data handling obligations — jurisdiction
  and compliance requirements the agent must observe.
- goalNarrative (1-3 sentences): The agent's current task and objectives.
  Include sub-goals as a natural continuation, not a bullet list.

RULES:
- Second person only: "You are...", "Your role is...", "You have...".
- Plain prose. No markdown, no bullet points, no headers.
- Be concise. Every sentence must carry information the agent needs to act on.
- Return ONLY the JSON object. No explanation, no preamble, no code fences.
```

Template is a `private static final String PROMPT_TEMPLATE` in
`ClaudeMarkdownRenderer` (Java text block). Not configurable at runtime — consumers
who need different prompt conventions implement `SystemPromptRenderer` and displace
the default bean.

### SemanticEnrichmentStep.enrich()

```java
Optional<SemanticEnrichment> enrich(ChatModel llm, ObjectNode payload) {
    try {
        ChatRequest request = ChatRequest.builder()
            .messages(
                SystemMessage.from(PROMPT_TEMPLATE),
                UserMessage.from(mapper.writeValueAsString(payload))
            )
            .responseFormat(RESPONSE_FORMAT)
            .build();

        ChatResponse response = llm.chat(request);
        return Optional.of(parse(response.aiMessage().text()));

    } catch (Exception e) {
        log.warn("Semantic enrichment failed ({}), falling back to structural rendering",
                 e.getMessage());
        return Optional.empty();
    }
}
```

`catch (Exception e)` is intentionally broad — `UnsupportedFeatureException` (LangChain4j
validation), HTTP API errors (unsupported model), and `JsonProcessingException` (parse
failure) all produce the same outcome: fall back to structural rendering. The warning
log preserves observability.

`response.aiMessage().text()` — `AiMessage` is a single-content type; `.text()` is
the correct accessor (distinct from `UserMessage.singleText()` per GE-20260525-80e370).

---

## Stage 3 — Format-specific assembly

All four `RenderFormat` values are handled. Both enriched and structural paths go
through the same format switch.

```java
private String assemble(Optional<SemanticEnrichment> enrichment,
                         AgentDescriptor descriptor,
                         AgentPromptContext context) {
    return switch (context.format()) {
        case CLAUDE_MD     -> assembleClaudeMarkdown(enrichment, descriptor, context);
        case OPENAI_SYSTEM -> assembleOpenAiSystem(enrichment, descriptor, context);
        case A2A_CARD      -> assembleA2aCard(descriptor, context);
        case GEMINI        -> assembleGemini(enrichment, descriptor, context);
    };
}
```

### CLAUDE_MD (enriched)

```
# {descriptor.name()}
**Agent ID:** {agentId}[  **Model:** {model}][  **Provider:** {provider}]

{identityNarrative}

## Role
{roleNarrative}

## Capabilities
{capabilityNarrative}

## How You Operate          ← omitted if dispositionNarrative is empty
{dispositionNarrative}

## Data Handling            ← omitted if constraintNarrative is empty
{constraintNarrative}

## Current Goal             ← omitted if goalNarrative is empty
{goalNarrative}

## Resources                ← structural, always present when resources non-empty
- **{label}**: {uri} ({type})

## Context                  ← structural pass-through, present when non-null
{situationalContext}
```

### CLAUDE_MD (structural fallback)

Refactored from current `renderStructural()`. Vocabulary-resolved slot heading
retained. Format structure mirrors the enriched path; raw descriptor field values
replace prose narratives.

### OPENAI_SYSTEM (enriched)

Dense prose, no markdown structure. All sections collapsed into paragraphs separated
by blank lines. Resources appended as a compact list. No `#` headers.

### OPENAI_SYSTEM (structural fallback)

Mirrors the enriched structure using raw field values. Fixes the current bug where
the structural path always produces CLAUDE_MD markdown regardless of requested format.

### A2A_CARD

Structural only — `SemanticEnrichment` is ignored. Per-capability prose descriptions
require a different intermediate type (deferred to eidos#13). Output is a JSON
object: `name`, `agentId`, `version`, `capabilities[]` (name, qualityHint).

### GEMINI (enriched + structural)

Delegates to `assembleClaudeMarkdown(enrichment, descriptor, context)` as a
placeholder — enrichment is passed through, not dropped. Full Gemini-specific
rendering (different tone and structure conventions) deferred to eidos#14.

---

## RenderedPromptCache SPI

Follows the `AgentStateStore` pattern exactly.

```java
// casehub-eidos-api
public interface RenderedPromptCache {
    Optional<RenderedPrompt> get(String cacheKey);
    void put(String cacheKey, RenderedPrompt result);
}
```

`put` has no return value and no checked exceptions. Implementations must handle
errors internally — a cache failure must never abort a render.

`NoOpRenderedPromptCache @DefaultBean` in `casehub-eidos` runtime: both methods are
no-ops. Zero cost when no caching is wanted.

`InMemoryRenderedPromptCache @Alternative @Priority(1)` in `casehub-eidos-memory`:
bounded LRU using `Collections.synchronizedMap(new LinkedHashMap<>(accessOrder=true))`
with `removeEldestEntry` override. `Collections.synchronizedMap` provides the
thread-safety guarantee required because `ClaudeMarkdownRenderer` is
`@ApplicationScoped` and called from multiple threads. No Caffeine dependency. Max
size configurable via `casehub.eidos.renderer.cache-size` (default 256). Activated
by adding the `casehub-eidos-memory` module as a dependency — no consumer code
changes.

The `RenderedPrompt` record is unchanged. `descriptorHash` and `contextHash` remain
for the external stale-detection use case (callers checking whether a running agent's
prompt is still current without re-rendering). The internal cache key (which includes
`TEMPLATE_VERSION`) is never surfaced.

---

## RenderedPrompt — no structural change

```java
record RenderedPrompt(String content, RenderFormat format, String descriptorHash, String contextHash) {}
```

The hashes are now computed from canonical JSON (Section — Stage 1) rather than
Java record `toString()`. Same field names, same semantics, corrected stability.

---

## Testing strategy

### Layer 1 — Payload construction (Stage 1, no LLM)

Assert on the `ObjectNode` produced before any LLM call:
- `tenancyId` absent; `id` (DB UUID) absent
- Vocabulary URIs absent; `slotLabel` / `slotDescription` present when resolved
- `model` is combined form when both family and version are set
- Goal fields present when context has a goal; absent when none
- `resources` and `situationalContext` absent from the payload

### Layer 2 — Format assembly (Stage 3, no LLM)

Pre-built `SemanticEnrichment` objects passed directly to assembly methods:
- `CLAUDE_MD` result contains `#` header with agent name
- `CLAUDE_MD` result omits `## How You Operate` when `dispositionNarrative` is empty
- `CLAUDE_MD` result always contains Resources section when non-empty
- `OPENAI_SYSTEM` result contains no `#` headers
- Structural fallback (`Optional.empty()`) produces valid, format-correct output

### Layer 3 — SemanticEnrichmentStep (Stage 2, capturing mock)

Capturing mock overrides `doChat(ChatRequest)` — asserts on `ChatRequest` content:
- System message equals `PROMPT_TEMPLATE`
- User message payload contains capability names; does not contain `tenancyId`
- `slotLabel` present in payload when vocabulary resolves it
- `ChatRequest` carries non-null `ResponseFormat`

Parse tests:
- Valid JSON with all six fields → correct `SemanticEnrichment`
- Blank optional fields → `Optional.empty()` in record
- Any exception during `llm.chat()` → `Optional.empty()` returned
- Malformed JSON → `Optional.empty()` returned

Cache behaviour:
- Cache hit → `llm.chat()` not called (verified via mock call count)

### Out of scope for CI

- Prose quality (grammar, coherence, effectiveness)
- Real Anthropic API calls
- Model-specific structured output behaviour

These require human evaluation or offline LLM-as-judge runs (eidos#16).

---

## Deferred items (filed as issues)

| Issue | Scope |
|---|---|
| eidos#13 | A2A_CARD per-capability prose narratives |
| eidos#14 | GEMINI enriched and structural rendering |
| eidos#15 | Prompt injection risk in descriptor field values |
| eidos#16 | Offline quality evaluation / LLM-as-judge |

---

## Platform coherence

- `RenderedPromptCache` is an Eidos domain SPI — stays in `casehub-eidos-api`,
  not `casehub-platform-api`.
- SPI default type: `NoOpRenderedPromptCache` (operational SPI — skipping is safe).
  Matches `AgentStateStore` precedent.
- `casehub-engine` consumes `casehub-eidos-api` for `AgentDescriptor` and
  `CapabilityHealth` only — does not call `SystemPromptRenderer.render()`. Adding
  `RenderedPromptCache` to `casehub-eidos-api` has no impact on engine.
- `PLATFORM.md` and `docs/repos/casehub-eidos.md` require updates after
  implementation commits (implementation-doc-sync step).

---

## New platform protocol to capture (post-implementation)

> Any Foundation extension that incorporates an optional LLM pass must have a
> structural fallback that produces correct, format-specific output for all declared
> `RenderFormat` values. The LLM pass is an enhancement, not a requirement.
