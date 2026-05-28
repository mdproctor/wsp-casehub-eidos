---
layout: post
title: "Two architectural errors in the rendering pipeline"
date: 2026-05-29
type: phase-update
entry_type: note
subtype: diary
projects: [eidos]
---

The existing `ClaudeMarkdownRenderer` sent YAML to an LLM and returned whatever
came back. Format, structure, tone — all the LLM's problem. I didn't question
this until I had to design the next version.

## Three options, two architectural errors

The real question was what role the LLM should play. Three options:

**Option C** — the LLM produces the complete format-specific output. Simple,
but format logic lives in prompt strings rather than code. Adding a new
`RenderFormat` means a new prompt template. Format correctness is untestable
without running an LLM. This isn't simplicity — it's moving the problem
somewhere less visible.

**Option B** — the LLM produces a unified prose narrative; the format renderer
wraps it. Sounds like clean separation until you check what the renderer actually
does. For `CLAUDE_MD` it adds a name header and a resource list. For `A2A_CARD`
— a structured JSON format — the prose is unusable; the renderer ignores it
entirely. Option B separates concerns that share nothing to separate.

**Option A** — the LLM returns section-keyed prose: `identityNarrative`,
`roleNarrative`, `capabilityNarrative`, and three optional fields. The format
renderer assembles these deterministically. Testable without an LLM, consistent
across formats, and the only option where semantic enrichment and format assembly
are genuinely distinct operations.

I brought Claude in to implement and verify as we worked through the details.
The format assembly for `OPENAI_SYSTEM` and `A2A_CARD` came out cleanly.
`GEMINI` is a placeholder until that format's conventions are clearer.

## The API that doesn't degrade

Before committing to `ResponseFormat.JSON` for the LLM call, I wanted to know
what happens when the schema is absent. The LangChain4j docs describe
`ResponseFormat.JSON` as a valid type value. The source says otherwise.

`InternalAnthropicHelper.validate()` throws `UnsupportedFeatureException` — not
a warning, not a degradation — when `responseFormat().type() == JSON` and
`responseFormat().jsonSchema() == null`. Anthropic has no schemaless JSON mode;
LangChain4j reflects this by refusing the request before any network call.
This isn't in the public documentation.

The design uses `ResponseFormat` with a full `JsonSchema` and wraps the LLM call
in `catch (Exception e)`. When the model doesn't support structured output —
older Claude 3.x, or any provider without JSON schema support — the exception is
caught and the structural renderer takes over.

Making all six prose fields required in the schema simplifies parsing. The prompt
instructs the LLM to return empty strings for absent sections; the parser maps
blank-or-whitespace to `Optional.empty()`.

## What the cache needs to know

The subtler bug appeared during implementation. The cache key came from
`descriptorHash` and `contextHash`. The context hash was computed from what
goes to the LLM — which initially meant only `goal`.

But `resources` and `situationalContext` don't go to the LLM. They're rendered
structurally in Stage 3. Changing a resource URI wouldn't change the context hash.
The cache would serve a result with the wrong URI.

The fix: split the two concerns. `buildContextPayload` captures everything that
affects the rendered output — goal, resources, situationalContext. A separate
`buildLlmPayload` sends only goal to the model. The cache key derives from the
full output contract, not the LLM's input.

The principle generalises: when a renderer has a generative step and a structural
step, the cache key must cover both. What the model receives and what the final
output depends on are not the same set.

One smaller thing. `sha256()` in the renderer truncated to 16 hex chars —
fine for cache collision resistance, but the name implied a full hash. Claude
flagged it during code review. We renamed it `fingerprint()`.
