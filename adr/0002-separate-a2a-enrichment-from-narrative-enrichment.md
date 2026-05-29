# 0002 — Separate A2A enrichment from narrative enrichment

Date: 2026-05-29
Status: Accepted

## Context and Problem Statement

`EidosSystemPromptRenderer` uses a shared `SemanticEnrichmentStep` for LLM-generated
narrative fields (identity, role, capability, disposition, constraint, goal). When
adding per-capability prose for the A2A_CARD format, the initial design added
`capabilityNarratives` as a required field in the shared schema. This caused two
problems: (1) every CLAUDE_MD/OPENAI_SYSTEM/GEMINI render generates and discards
N capability descriptions (token waste), and (2) A2A cards received goal context
from the caller's render context, contaminating descriptor-only card metadata with
transient request state.

## Decision Drivers

* A2A cards describe structural capability metadata — they are descriptor-only and
  context-independent; goal context must not influence them
* Token waste proportional to capability count on every non-A2A render is unacceptable
* The `llm-pass-structural-fallback` and `renderer-cache-key-includes-format` protocols
  require format-specific correctness throughout the rendering pipeline

## Considered Options

* **Option A** — Add `capabilityNarratives` as a required field in the shared schema
* **Option B** — Add `capabilityNarratives` as optional in the shared schema, with
  separate A2A-only prompt guidance and stripped goal payload
* **Option C** — Dedicated `A2ASemanticEnrichmentStep` with its own schema, its own
  prompt template, and a descriptor-only payload

## Decision Outcome

Chosen option: **Option C**, because it provides complete semantic isolation — the
shared schema is untouched, no token waste on non-A2A renders, and the A2A payload
can be scoped correctly to descriptor-only without special-casing `buildLlmPayload`.

### Positive Consequences

* Zero token overhead on non-A2A renders regardless of capability count
* A2A card narratives cannot be contaminated by caller goal context
* `SemanticEnrichment` record and `SemanticEnrichmentStep` schema are stable — no
  version-skew risk for cache keys that include `TEMPLATE_HASH`
* `render()` flow is readable: Stage 2a = narrative enrichment, Stage 2b = A2A enrichment

### Negative Consequences / Tradeoffs

* Two enrichment step classes to maintain instead of one
* If a future format also needs capability narratives, it needs its own step too
  (acceptable — this is the correct isolation boundary)

## Pros and Cons of the Options

### Option A — shared schema, required field

* ✅ Single enrichment class and schema
* ❌ Generates and discards N capability descriptions on every CLAUDE_MD/OPENAI_SYSTEM/GEMINI render
* ❌ A2A payload receives goal context from the caller's render context

### Option B — shared schema, optional field

* ✅ No schema duplication
* ✅ capabilityNarratives absent from non-A2A renders if LLM omits
* ❌ "May omit" is an unreliable guarantee — LLM compliance varies
* ❌ Goal contamination still occurs unless buildLlmPayload is special-cased

### Option C — dedicated A2ASemanticEnrichmentStep

* ✅ Complete semantic isolation — schema, prompt, and payload scoped to A2A
* ✅ Zero token waste on non-A2A renders
* ✅ Descriptor-only payload without conditional stripping logic
* ❌ Two enrichment step classes (acceptable cost for the isolation benefit)

## Links

* Protocol PP-20260529-368527 — format-specific enrichment schema isolation
* Protocol PP-20260529-35f3bd — llm-pass-structural-fallback
* eidos#13
