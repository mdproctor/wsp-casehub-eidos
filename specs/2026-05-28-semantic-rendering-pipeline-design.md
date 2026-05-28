# Semantic Rendering Pipeline — Design Spec
**Issue:** casehubio/eidos#6
**Date:** 2026-05-28
**Status:** In review (round 3)

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
│                             │    tenancyId and vocabulary URI fields excluded
└─────────────────────────────┘
        │
        ▼
┌─────────────────────────────┐
│  Stage 2: Semantic Enrich.  │  → Optional<SemanticEnrichment>
│  SemanticEnrichmentStep     │    skipped when !usesEnrichment(format)
│  ResponseFormat + JsonSchema│    skipped when llm == null
│  catch-all → structural     │    empty on any failure → structural path
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
| `SemanticEnrichmentStep` | `runtime/renderer/` | package-private class | Stage 2 only: LLM call + parse |
| `SemanticEnrichment` | `runtime/renderer/` | package-private record | Six prose fields — intermediate type |
| `RenderedPromptCache` | `api` | public SPI | Cache lookup and store |
| `NoOpRenderedPromptCache` | `runtime` | `@DefaultBean` | No caching by default |
| `InMemoryRenderedPromptCache` | `persistence-memory` | `@Alternative @Priority(1)` | Bounded LRU, activated by classpath |

---

## Stage 1 — Canonical JSON payload

Built using Jackson `ObjectMapper` (a `@Produces @ApplicationScoped` CDI bean in
`quarkus-jackson`, injected into `ClaudeMarkdownRenderer`). Explicit field-by-field
construction — never `valueToTree(descriptor)`.

### AgentDescriptor fields

| Field | Payload key | Included? | Notes |
|---|---|---|---|
| `agentId` | `agentId` | ✅ | |
| `name` | `name` | ✅ | |
| `version` | `version` | ✅ | |
| `provider` | `provider` | ✅ | |
| `modelFamily` + `modelVersion` | `model` | ✅ | Combined: `"claude/claude-3-7-sonnet"` |
| `weightsFingerprint` | `weightsFingerprint` | ✅ | Included for hash coverage — if the weights checkpoint changes, cached narratives must be regenerated. The LLM may use it in `identityNarrative` or ignore it. |
| `slot` | `slot` | ✅ | Raw value |
| Vocab-resolved slot label | `slotLabel` | ✅ if resolved | From `VocabularyRegistry` |
| Vocab-resolved slot description | `slotDescription` | ✅ if resolved | From `VocabularyRegistry` |
| `jurisdiction` | `jurisdiction` | ✅ if non-null | |
| `dataHandlingPolicy` | `dataHandlingPolicy` | ✅ if non-null | |
| `domainVocabulary` | — | ❌ | Raw URI — resolved content included instead |
| `slotVocabulary` | — | ❌ | Same |
| `dispositionVocabulary` | — | ❌ | Same |
| `tenancyId` | — | ❌ | Internal — privacy concern |
| `disposition` | `disposition` | ✅ if non-null | socialOrient, ruleFollowing, riskAppetite, autonomy, canDelegate |

Note: `AgentDescriptor` is a pure-Java domain record. It has no `id` (DB UUID)
field — that lives on `AgentDescriptorEntity` (the JPA entity in the runtime module)
and never reaches the domain record being serialised here.

### AgentCapability fields (per capability in the `capabilities` array)

| Field | Included? | Notes |
|---|---|---|
| `name` | ✅ | |
| `qualityHint` | ✅ | |
| `latencyHintP50Ms` | ✅ | |
| `inputTypes` | ✅ | Defines what the capability accepts — actionable for the agent |
| `outputTypes` | ✅ | Defines what the capability produces — actionable for the agent |
| `epistemicDomains` | ✅ | Domain confidence map |
| `costHint` | ❌ | Operational metadata for the platform — not useful in agent prose |
| `tags` | ❌ | Routing/filtering labels — not meaningful for agent self-description |

### Context fields

| Field | Included? | Notes |
|---|---|---|
| `goal` (description, subGoals, caseRef) | ✅ if present | |
| `resources` | ❌ | Rendered structurally in Stage 3 — no LLM enrichment needed |
| `situationalContext` | ❌ | Already natural language — Stage 3 pass-through |
| `format` | ❌ | Stage 3 concern only |

### Hash computation

```java
String descriptorHash = sha256(descriptorNode.toString());
String contextHash    = sha256(contextNode.toString());
```

`ObjectNode.toString()` uses insertion-order field ordering (controlled entirely by
explicit construction code), making it insertion-order stable and deterministic
across JVM restarts. This replaces the current `sha256(descriptor.toString())`
which uses Java record `toString()` — not canonical.

### Internal cache key

```java
String cacheKey = descriptorHash + ":" + contextHash + ":" +
                  context.format().name() + ":" + TEMPLATE_HASH;
```

`context.format()` is included because `RenderedPrompt` is format-specific: the same
descriptor and context rendered as `CLAUDE_MD` and `OPENAI_SYSTEM` produce different
content and must occupy different cache entries.

`TEMPLATE_HASH` (see below) invalidates all cached results when the prompt template
changes.

---

## Stage 2 — Semantic enrichment

### Format predicate

Stage 2 is conditionally skipped based on the requested format. The predicate is a
private static method in `ClaudeMarkdownRenderer` — renderer-internal semantics must
not pollute the public `RenderFormat` enum:

```java
private static boolean usesEnrichment(RenderFormat format) {
    return switch (format) {
        case CLAUDE_MD, OPENAI_SYSTEM, GEMINI -> true;
        case A2A_CARD                          -> false;
    };
}
```

An exhaustive switch expression is used deliberately: adding a new `RenderFormat`
constant without a corresponding case produces a compile error, enforcing that the
implementer explicitly decides whether the new format uses enrichment.

### Template hash (replaces manual TEMPLATE_VERSION)

```java
// PROMPT_TEMPLATE must be declared before TEMPLATE_HASH — static initializers run
// in declaration order. Reversing them causes sha256(null) at class load:
// NullPointerException wrapped in ExceptionInInitializerError, not a quiet wrong value.
private static final String PROMPT_TEMPLATE = """
        ...(see below)...
        """;
private static final String TEMPLATE_HASH = sha256(PROMPT_TEMPLATE).substring(0, 8);
```

`sha256()` is a static method and is available from class load, before field
initializers run. The only ordering constraint is that `PROMPT_TEMPLATE` is
initialised before `TEMPLATE_HASH`.

`TEMPLATE_HASH` replaces the manually-maintained `TEMPLATE_VERSION` constant.
Changing `PROMPT_TEMPLATE` automatically produces a new hash and invalidates all
cached results — no separate manual bump required.

### SemanticEnrichment record (package-private)

```java
record SemanticEnrichment(
    String identityNarrative,       // always populated
    String roleNarrative,           // always populated
    String capabilityNarrative,     // always populated
    Optional<String> dispositionNarrative,   // empty string sentinel → Optional.empty()
    Optional<String> constraintNarrative,    // empty string sentinel → Optional.empty()
    Optional<String> goalNarrative           // empty string sentinel → Optional.empty()
) {}
```

### SemanticEnrichmentStep (package-private class)

A plain package-private class — no CDI annotations, not a bean. Constructed once by
`ClaudeMarkdownRenderer`'s constructor. Holds `ObjectMapper` (for Stage 2 JSON
serialisation) and the `RESPONSE_FORMAT` constant (static field). `ChatModel` is
passed to `enrich()` per call — the enrichment step does not hold the LLM reference.

```java
// Static — built once, shared across all render calls.
static final ResponseFormat RESPONSE_FORMAT = ResponseFormat.builder()
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

**Model compatibility:** `ResponseFormat` with `JsonSchema` is translated by the
LangChain4j Anthropic integration to `AnthropicOutputConfig` (Anthropic's native
structured output). Based on Anthropic's November 2025 structured output beta
documentation, this requires Claude Sonnet 4.5 or Opus 4.1+. These version claims
are based on external documentation and are untested in this codebase — the
`catch (Exception e)` block is the authoritative safety net regardless of model
version. Older models produce an API-level error that is caught and handled by
fallback to structural rendering.

`ResponseFormat.JSON` without a schema throws `UnsupportedFeatureException` in
`InternalAnthropicHelper.validate()` and must not be used.

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

`catch (Exception e)` is intentionally broad: `UnsupportedFeatureException` (LangChain4j
validation), HTTP API errors (unsupported model version), and `JsonProcessingException`
(parse failure) all produce the same outcome.

`response.aiMessage().text()` — `AiMessage` is a single-content type; `.text()` is
the correct accessor (distinct from `UserMessage.singleText()` per GE-20260525-80e370).

### parse() specification

`parse()` returns a fully-formed `SemanticEnrichment` with `Optional` fields already
resolved — the `optional()` helper inside `parse()` owns the `String → Optional`
conversion. No raw strings are held in the record.

`.strip()` is applied to all string fields before the blank check. Whitespace-only
strings (`"  "`) map to `Optional.empty()` via the same path as `""`.

```java
private SemanticEnrichment parse(String json) throws JsonProcessingException {
    JsonNode node = mapper.readTree(json);
    return new SemanticEnrichment(
        node.get("identityNarrative").asText(),
        node.get("roleNarrative").asText(),
        node.get("capabilityNarrative").asText(),
        optional(node, "dispositionNarrative"),
        optional(node, "constraintNarrative"),
        optional(node, "goalNarrative")
    );
}

private static Optional<String> optional(JsonNode node, String field) {
    JsonNode n = node.get(field);
    if (n == null || n.isNull()) return Optional.empty();
    String v = n.asText("").strip();
    return v.isEmpty() ? Optional.empty() : Optional.of(v);
}
```

If `parse()` throws `JsonProcessingException` it propagates to `enrich()`'s
`catch (Exception e)` and triggers structural fallback.

### Prompt template (system message)

```
You are writing narrative descriptions for an AI agent's system prompt.

Given the agent definition in JSON, produce a JSON object with prose descriptions
for each field. Write in second person, addressing the agent directly.

REQUIRED FIELDS (always populate):
- identityNarrative (1-2 sentences): The agent's name, model, and version context.
- roleNarrative (1-3 sentences): The role this agent plays and its purpose.
  If slotLabel and slotDescription are present, prefer them over the raw slot value.
- capabilityNarrative (2-4 sentences): What the agent can do.
  Include inputTypes and outputTypes when present.
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

---

## ClaudeMarkdownRenderer constructors

Both constructors must wire all four dependencies and initialise `enrichmentStep`.

### CDI constructor

```java
@Inject
public ClaudeMarkdownRenderer(
        @Any Instance<ChatModel> llm,
        VocabularyRegistry vocab,
        RenderedPromptCache cache,
        ObjectMapper mapper) {
    this.llm = llm.isResolvable() ? llm.get() : null;
    this.vocab = vocab;
    this.cache = cache;
    this.mapper = mapper;
    this.enrichmentStep = new SemanticEnrichmentStep(mapper);
}
```

`ObjectMapper` is injected from CDI (`quarkus-jackson` registers it as
`@Produces @ApplicationScoped`). The same instance is used by `ClaudeMarkdownRenderer`
for Stage 1 payload building and passed to `SemanticEnrichmentStep` for Stage 2
serialisation.

### Package-private test constructor

```java
/** Package-private constructor for pure-Java tests — no CDI required. */
ClaudeMarkdownRenderer(ChatModel llm, VocabularyRegistry vocab,
                       RenderedPromptCache cache, ObjectMapper mapper) {
    this.llm = llm;
    this.vocab = vocab;
    this.cache = cache;
    this.mapper = mapper;
    this.enrichmentStep = new SemanticEnrichmentStep(mapper);
}
```

Existing tests using the two-argument constructor will fail to compile — this is the
correct signal to update them with the new dependencies.

---

## render() orchestration

```java
@Override
public RenderedPrompt render(AgentDescriptor descriptor, AgentPromptContext context) {
    ObjectNode descriptorNode = buildDescriptorPayload(descriptor);
    ObjectNode contextNode    = buildContextPayload(context);

    String descriptorHash = sha256(descriptorNode.toString());
    String contextHash    = sha256(contextNode.toString());
    String cacheKey       = descriptorHash + ":" + contextHash + ":"
                          + context.format().name() + ":" + TEMPLATE_HASH;

    Optional<RenderedPrompt> cached = cache.get(cacheKey);
    if (cached.isPresent()) return cached.get();

    Optional<SemanticEnrichment> enrichment = Optional.empty();
    if (llm != null && usesEnrichment(context.format())) {
        ObjectNode fullPayload = buildFullPayload(descriptorNode, contextNode);
        enrichment = enrichmentStep.enrich(llm, fullPayload);
    }

    String content = assemble(enrichment, descriptor, context);
    RenderedPrompt result = new RenderedPrompt(content, context.format(),
                                               descriptorHash, contextHash);
    cache.put(cacheKey, result);
    return result;
}
```

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

**This is a deliberate behavioral change from the current implementation.** The
current structural path uses the vocabulary-resolved slot value as the section
heading (e.g., `## Code Reviewer`). The new structural fallback uses `## Role` as a
fixed heading, mirroring the enriched path's structure. Both enriched and structural
paths must produce structurally equivalent output — consistent headings are the
correct design. Callers that parse section headings from the structural path will
need to be updated.

### OPENAI_SYSTEM (enriched)

Dense prose, no markdown structure. All sections collapsed into paragraphs separated
by blank lines. No `#` headers.

```
{identityNarrative} {roleNarrative}

{capabilityNarrative}

{dispositionNarrative if present}

{constraintNarrative if present}

{goalNarrative if present}

Resources: {label} ({uri})[, ...]

{situationalContext if present}
```

### OPENAI_SYSTEM (structural fallback)

Mirrors the enriched structure above using raw descriptor field values. Fixes the
current bug where the structural path always produces CLAUDE_MD markdown regardless
of requested format.

### A2A_CARD

Structural only — Stage 2 is skipped (`usesEnrichment(A2A_CARD) == false`). Per-capability
prose descriptions require a different intermediate type (deferred to eidos#13).
Output is a JSON object: `name`, `agentId`, `version`, `capabilities[]`
(name, qualityHint).

### GEMINI (enriched + structural)

**Placeholder until eidos#14.** Delegates to
`assembleClaudeMarkdown(enrichment, descriptor, context)` — enrichment is passed
through unchanged. Until eidos#14 lands, `RenderedPrompt.format() == GEMINI` but
`RenderedPrompt.content()` is CLAUDE_MD-structured. Callers must not branch on
`result.format() == GEMINI` expecting Gemini-specific structure.

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
errors internally so a cache failure never aborts a render.

`NoOpRenderedPromptCache @DefaultBean` in `casehub-eidos` runtime: both methods are
no-ops.

`InMemoryRenderedPromptCache @Alternative @Priority(1)` in `casehub-eidos-memory`:
bounded LRU using `Collections.synchronizedMap(new LinkedHashMap<>(accessOrder=true))`
with `removeEldestEntry` override. `Collections.synchronizedMap` provides the
thread-safety guarantee required because `ClaudeMarkdownRenderer` is
`@ApplicationScoped` and called from multiple threads. No Caffeine dependency. Max
size configurable via `casehub.eidos.renderer.cache-size` (default 256). Activated
by adding `casehub-eidos-memory` as a dependency — no consumer code changes.

### Why RenderedPrompt rather than SemanticEnrichment is cached

Caching `SemanticEnrichment` (Stage 2 output) would be more efficient: it is
format-agnostic, so one cache entry serves CLAUDE_MD, OPENAI_SYSTEM, and GEMINI, and
the LLM call (the expensive operation) is cached rather than the cheap string
assembly. The blocker: `SemanticEnrichment` is package-private. Making it the SPI
value type either exposes an implementation detail in the public API or degrades to
`Map<String, String>`, both of which are worse. Caching `RenderedPrompt` keeps the
SPI boundary clean at the cost of per-format cache entries. With the
`format.name()`-in-cache-key fix applied this is correct, if not maximally efficient.

---

## RenderedPrompt — no structural change

```java
record RenderedPrompt(String content, RenderFormat format, String descriptorHash, String contextHash) {}
```

Hashes now computed from canonical JSON (same `ObjectNode.toString()` used as LLM
input) rather than Java record `toString()`. Same field names, same external
semantics, corrected stability. `descriptorHash` and `contextHash` remain for the
external stale-detection use case.

---

## Testing strategy

### Layer 1 — Payload construction (Stage 1, no LLM)

Assert on the `ObjectNode` produced by `buildDescriptorPayload()` and
`buildContextPayload()`:

- `tenancyId` absent; vocabulary URI fields absent
- `slotLabel` / `slotDescription` present when vocabulary resolves them
- `weightsFingerprint` present in the node when non-null
- `model` is the combined form when both family and version are set
- `inputTypes` and `outputTypes` present in each capability entry
- `costHint` and `tags` absent from capability entries
- Goal fields present when context has a goal; absent when none
- `resources` and `situationalContext` absent from the payload

### Layer 2 — Format assembly (Stage 3, no LLM)

Pre-built `SemanticEnrichment` objects passed directly to assembly methods:

- `CLAUDE_MD` result contains a `#` header with the agent name
- `CLAUDE_MD` result omits `## How You Operate` when `dispositionNarrative` is empty
- `CLAUDE_MD` result always contains Resources section when resources are non-empty,
  regardless of LLM presence
- `CLAUDE_MD` structural fallback uses `## Role` heading (not `## {slot_label}`)
- `OPENAI_SYSTEM` result contains no `#` headers (enriched path)
- `OPENAI_SYSTEM` structural fallback contains no `#` headers
- `GEMINI` produces the same content as `CLAUDE_MD` (placeholder delegation verified)
- Different formats (CLAUDE_MD, OPENAI_SYSTEM) for the same descriptor and context
  produce different cache keys — this test validates both the format-in-key fix from
  review finding #1 and serves as the regression guard for that fix

### Layer 3 — SemanticEnrichmentStep (Stage 2, capturing mock)

`PROMPT_TEMPLATE` is declared package-private (not `private`) to allow exact-match
assertions from tests in the same package.

Capturing mock overrides `doChat(ChatRequest)` and asserts on the `ChatRequest`
content (not LLM output):

- System message equals `PROMPT_TEMPLATE` constant
- User message contains capability names, `slotLabel` when vocabulary resolves it,
  and `inputTypes`/`outputTypes` when present
- `tenancyId` does not appear in the user message payload
- `ChatRequest` carries a non-null `ResponseFormat`

Parse tests:

- Valid JSON with all six fields → correct `SemanticEnrichment`
- Blank or whitespace-only optional fields → `Optional.empty()` in record
- Any exception during `llm.chat()` → `Optional.empty()` from `enrich()`
- Malformed JSON → `Optional.empty()` from `enrich()`

Cache tests use an inline `TestRenderedPromptCache` (simple `HashMap<String,
RenderedPrompt>`) defined in the test class. Pulling `casehub-eidos-memory` into
runtime tests would create a cross-module dependency; an inline implementation is
cleaner and sufficient.

- Cache hit → `llm.chat()` not called (verified via mock call count)
- A2A_CARD format → `llm.chat()` not called (Stage 2 skipped by `usesEnrichment`)

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
| eidos#14 | GEMINI enriched and structural rendering; `ClaudeMarkdownRenderer` rename to `DefaultSystemPromptRenderer` |
| eidos#15 | Prompt injection risk in descriptor field values |
| eidos#16 | Offline quality evaluation / LLM-as-judge |

`ClaudeMarkdownRenderer` is a misnomer once it handles OPENAI_SYSTEM, A2A_CARD, and
GEMINI. `DefaultSystemPromptRenderer` is the correct name. Deferring to eidos#14 —
at that point GEMINI renders differently from CLAUDE_MD, which is the natural moment
to rename.

---

## Platform coherence

- `RenderedPromptCache` is an Eidos domain SPI — stays in `casehub-eidos-api`.
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
