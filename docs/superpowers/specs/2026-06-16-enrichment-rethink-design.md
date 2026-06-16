# Enrichment Rethink — Design Spec

**Date:** 2026-06-16  
**Status:** Approved (revised after code review)  
**Context:** Emerged from Phase 2 eval analysis — independent judge run, proximity scoring investigation, and systematic audit of the enrichment pipeline against its original design intent. Revised to address assembly restructuring, reactive path coverage, module placement, payload narrowing, and design questions on disposition replacement and briefing.

---

## Problem

Three separate issues compound each other but have independent fixes.

### 1. Enrichment scope crept beyond original intent

The original spec (`eidos.md`, Phase 3 section) said:

> *"LLM prose layer: disposition section only — renders socialOrient + ruleFollowing + situationalContext into 2–3 natural language sentences."*

What was implemented: 6 narrative fields covering identity, role, capability, disposition, constraint, and goal. The scope expansion is undocumented.

The eval shows the extra scope produces no value: identity, role, and capability sections scored 4.97–5.0 structurally. The LLM rephrases schema fields that already render well without it.

### 2. Enrichment mechanics are broken

`SemanticEnrichmentStep.stripCodeFences()` strips only the outer fence. It does not extract JSON from within prose preamble — it is not equivalent to `PromptJudge.extractJson()` which finds the outermost `{...}` block. Claude Opus 4.6 frequently returns prose starting with natural language, or JSON within code fences preceded by a preamble. Both fail silently as structural fallbacks.

`ReactiveSemanticEnrichmentStep.parseOrEmpty()` has the same bug: it calls `stripCodeFences()` and uses the same 6-field `SemanticEnrichment` constructor. No retry on parse failure.

Result: enrichment never runs in practice on Claude Opus 4.6. All renders are structural.

### 3. Proximity eval measures the wrong target

Proximity compares renders against `originalProse`. But `originalProse` was the source used to *create* the descriptor — not the rendering target. Every profile's `vocabularyGaps` field explicitly documents what the descriptor cannot express:

- `velocity-as-feature` → FULL LOSS  
- `six-month maintainability horizon` → FULL LOSS  
- `trust-in-pipeline` → FULL LOSS

These are intentional losses. No enrichment can recover them — they are not in the descriptor. The proximity metric tests fidelity to information that was deliberately excluded.

---

## Quality Target

**The renderer's job is: produce the best system prompt expressible from the `AgentDescriptor` as given.**

The descriptor is the truth. If `riskAppetite: bold` is in the descriptor and "90% elegant beats perfect" is not, the render correctly omits that principle — unless the author placed it in `briefing`. The renderer is not a reconstruction engine.

`originalProse` remains in eval profiles as research material — but it is not the rendering target and must not be the eval target.

---

## Changes

### Change 1 — Narrow enrichment scope to disposition + goal; decompose assembly

#### 1a. SemanticEnrichment record

```java
record SemanticEnrichment(
    Optional<String> dispositionNarrative,
    Optional<String> goalNarrative
) {}
```

Four fields removed: `identityNarrative`, `roleNarrative`, `capabilityNarrative`, `constraintNarrative`. Identity, role, capabilities, and constraint sections render structurally always — the LLM adds no measurable value there (eval: 4.97–5.0 structurally).

`goalNarrative` is retained. Sub-goal bullet enumeration ("- Flag off-by-one errors") is a poor LLM instruction format. Flowing goal prose ("Your current task is to review a Java PR, specifically looking for...") is actionable in a way bullets are not.

#### 1b. Binary gate → selective override in assembly

The current assembly uses a binary gate: if enrichment present, use all 6 LLM fields; else call the structural fallback method. After this change, structural assembly runs for every section by default, and enrichment optionally overrides only disposition and goal.

**`assembleMarkdown` structure after Change 1:**

```
// Header — always structural (unchanged)
name, agentId, model, provider line

// Role — always structural
assembleMarkdownRole(sb, descriptor)

// Capabilities — always structural  
assembleMarkdownCapabilities(sb, descriptor)

// Disposition — enriched OR structural (selective override)
if enrichment.dispositionNarrative().isPresent():
    "## How You Operate\n" + dispositionNarrative
else:
    assembleMarkdownDisposition(sb, descriptor)

// Data Handling — always structural
assembleMarkdownDataHandling(sb, descriptor)

// Goal — enriched OR structural (selective override)
if enrichment.goalNarrative().isPresent():
    "## Current Goal\n" + goalNarrative
else:
    assembleMarkdownGoal(sb, context)

// Resources — always structural (unchanged)
// Situational context — always structural (unchanged)
```

The existing `assembleMarkdownStructural` method becomes the composition of its constituent section methods. Each section method is extracted to be callable independently: `assembleMarkdownRole`, `assembleMarkdownCapabilities`, `assembleMarkdownDisposition`, `assembleMarkdownDataHandling`, `assembleMarkdownGoal`. The outer structural method calls all of them in sequence; the selective-override path calls each directly.

**`assembleProse` structure after Change 1:**

Same pattern. Structural identity+role as dense prose runs always; capabilities structural prose runs always; disposition and goal become selective overrides.

#### 1c. dispositionNarrative REPLACES structural disposition

When `dispositionNarrative` is present, it replaces the structural bullet list entirely. It does not supplement it.

Rationale: structural bullets ("- Risk appetite: bold") are factually correct but not actionable instructions. LLM prose ("You approach decisions boldly, approving when fundamentals are sound rather than blocking on theoretical concerns") is. Mixing both creates noise and redundancy.

Risk: if the LLM omits an axis from the narrative, there is no structural anchor. Mitigation:
- The PROMPT_TEMPLATE instructs: "Cover ALL axes present in the disposition payload. Omitting any axis present in the payload is incorrect."
- Retry on parse failure handles transient non-JSON; the prompt instruction handles axis omission.
- The `dispositionNarrative` field remains Optional — if enrichment fails entirely, the structural bullet list is the fallback. That is the correct fallback for total enrichment failure, not a partial-coverage supplement for enrichment success.
- FACTUAL_FIDELITY in the quality eval flags omissions at eval time.

#### 1d. Updated PROMPT_TEMPLATE and RESPONSE_FORMAT

`PROMPT_TEMPLATE` rewritten to describe only two output fields:
- `dispositionNarrative` (2-3 sentences): cover all disposition axes present in the payload; use vocabulary framework language when vocabularyName present; second person
- `goalNarrative` (1-3 sentences): current task and objectives in flowing prose; sub-goals as natural continuation, not bullets; second person; empty string if no goal

`RESPONSE_FORMAT` JsonSchema narrows to two string fields: `dispositionNarrative` and `goalNarrative`. Both optional (either may be empty string).

`TEMPLATE_HASH` is recomputed from the new template — this automatically invalidates all existing cached renders. No manual action required.

---

### Change 2 — Fix enrichment mechanics (blocking and reactive paths)

#### 2a. Extract JSON extraction to shared runtime utility

`PromptJudge.extractJson()` is in the eval module. `SemanticEnrichmentStep` is in runtime. There is no dependency from runtime → eval (and adding one would be incorrect). The logic must be extracted to a shared class in the runtime module.

**New class:** `JsonExtractionUtil` (package-private, `io.casehub.eidos.runtime.renderer`)

```java
static String extractJson(String text) {
    String s = text.strip();
    // Strip markdown code fences (```json...``` or ```...```)
    if (s.startsWith("```")) {
        int nl = s.indexOf('\n');
        if (nl != -1) s = s.substring(nl + 1).strip();
        if (s.endsWith("```")) s = s.substring(0, s.length() - 3).stripTrailing();
    }
    // Extract JSON from within prose preamble (finds outermost {...})
    int first = s.indexOf('{');
    int last = s.lastIndexOf('}');
    if (first > 0 && last > first) s = s.substring(first, last + 1);
    return s;
}
```

`SemanticEnrichmentStep.stripCodeFences()` is replaced by a call to `JsonExtractionUtil.extractJson()`.
`ReactiveSemanticEnrichmentStep.parseOrEmpty()` uses `JsonExtractionUtil.extractJson()`.
`PromptJudge.extractJson()` in the eval module is reimplemented as a call to the runtime's `JsonExtractionUtil.extractJson()` (eval depends on runtime) — or kept as a local copy with a comment pointing to the canonical location. Eval-to-runtime call is cleaner; local copy avoids a cross-module coupling for test-only code.

#### 2b. Add retry on parse failure — SemanticEnrichmentStep

```java
Optional<SemanticEnrichment> enrich(ChatModel llm, ObjectNode payload) {
    try {
        ChatRequest request = ...;
        try {
            return Optional.of(parse(llm.chat(request).aiMessage().text()));
        } catch (JsonProcessingException first) {
            log.warn("Enrichment: non-JSON response, retrying ({})", first.getMessage());
            return Optional.of(parse(llm.chat(request).aiMessage().text()));
        }
    } catch (Exception e) {
        log.warn("Semantic enrichment failed ({}), falling back to structural", e.getMessage());
        return Optional.empty();
    }
}
```

The retry uses the same request object. Second failure falls back to `Optional.empty()` — structural rendering. No change to fallback behaviour.

#### 2c. ReactiveSemanticEnrichmentStep — identical fixes

`ReactiveSemanticEnrichmentStep` requires the same two changes:

1. Replace `SemanticEnrichmentStep.stripCodeFences(json)` with `JsonExtractionUtil.extractJson(json)` in `parseOrEmpty()`.
2. Update the `SemanticEnrichment` constructor call from 6 fields to 2 fields (dispositionNarrative, goalNarrative) after Change 1 narrows the record.

Retry in the reactive path: the `StreamingChatResponseHandler.onError()` already routes to `Optional.empty()`. Adding a retry in the reactive path requires a more complex CompletableFuture chain. For the reactive path, a single attempt with graceful fallback is sufficient — the synchronous path handles the retry case. Document this decision.

---

### Change 3 — Narrow the enrichment LLM payload

`buildLlmPayload()` currently deep-copies the full descriptor node (identity, slot, capabilities, disposition, jurisdiction, data handling) and adds goal. After Changes 1, the enrichment LLM only produces disposition and goal narratives. Sending identity, slot, capability, and constraint fields is noise that wastes tokens and may distract the model.

**New method:** `buildEnrichmentPayload(descriptorNode, contextNode)` — replaces `buildLlmPayload` for the enrichment call.

Payload structure:
```json
{
  "disposition": { /* per-axis objects with vocabulary context, extracted from descriptorNode */ },
  "goal": { /* description + subGoals + caseRef, if present, extracted from contextNode */ },
  "briefing": "..."  /* if present in descriptor — see Change 4 */
}
```

`descriptorNode` already contains the disposition object with vocabulary-resolved labels and names (built by `buildDescriptorPayload`). Extract it directly: `descriptorNode.get("disposition")`. Same for goal from `contextNode.get("goal")`.

`buildLlmPayload` is retained for any existing callers if any exist outside the enrichment path; otherwise removed. If `buildLlmPayload` has no callers after this change, delete it.

---

### Change 4 — Add `AgentDescriptor.briefing` + include in enrichment payload

#### 4a. AgentDescriptor field

```java
public record AgentDescriptor(
    // ... existing fields ...
    String briefing   // nullable — principles the structured axes cannot express
) {}
```

`briefing` is a nullable free-text field. Its content is behavioral principles the structured disposition axes cannot capture: "90% elegant beats perfect. Trust the pipeline. Speed is a feature." It is the correct answer to the `vocabularyGap: FULL` cases documented in eval profiles.

`AgentDescriptorValidator` adds a length bound (e.g. MAX_BRIEFING = 500 chars) — long enough for several principles, short enough to prevent misuse as a general notes field.

#### 4b. Briefing in the enrichment payload

`briefing` is included in `buildEnrichmentPayload()` when non-null:

```json
{
  "disposition": { ... },
  "goal": { ... },
  "briefing": "Speed is a feature. Review latency is a cost."
}
```

The `PROMPT_TEMPLATE` instructs: "If briefingText is present, weave its principles naturally into the dispositionNarrative — do not quote it verbatim."

Rationale: if briefing is appended structurally after LLM-generated disposition prose, the two outputs can contradict — the LLM might write "You approach decisions with measured care" while briefing says "Speed is a feature." Giving the LLM the briefing content allows it to produce coherent prose that incorporates the principle: "You approve changes boldly when fundamentals are sound — speed is a feature for your team, and review latency is a cost that compounds."

Structural fallback when enrichment fails: if `briefing` is non-null and enrichment falls back to the structural disposition bullet list, briefing renders as a separate `## Operating Principles` section after disposition.

#### 4c. YAML placement

`briefing` is a field on `AgentDescriptor`, not on `AgentProfile`. In the eval profile YAML it lives inside the `descriptor:` block:

```yaml
descriptor:
  agentId: sw-engineer-bold-01
  briefing: "Speed is a feature. Review latency is a cost. 90% elegant beats perfect."
  slot: reviewer
  ...
```

`AgentProfileLoader` requires no change — Jackson deserialises `briefing` automatically from the descriptor block.

`AgentProfile.vocabularyGaps` with `loss: FULL` entries are the source for `briefing` content when populating eval profiles.

---

### Change 5 — Redefine proximity eval target

#### 5a. What proximity measures after this change

Current: compares render to `originalProse` — measures reconstruction fidelity against intentionally-discarded information.

New: measures **disposition axis completeness** — for each axis present in the descriptor, is that axis correctly and completely conveyed in the render?

This is distinct from `FACTUAL_FIDELITY`:
- `FACTUAL_FIDELITY` = no false positives (claims in render not grounded in descriptor — hallucinations). Asks: "Did you invent things?"
- Redesigned Proximity = no false negatives for disposition axes (axis values present in descriptor but missing or contradicted in render). Asks: "Did you miss things?"

Together they cover the full fidelity space for the disposition section.

#### 5b. Updated ProximityJudge

**System prompt rewritten** to frame around omissions:

> "You are evaluating whether a rendered AI agent system prompt correctly and completely expresses the agent's disposition axes as declared in the descriptor. For each axis present in the descriptor's disposition object, check whether the render conveys that axis value. Score 0–5: 5 = all axes clearly and correctly expressed; 4 = minor omission or softening; 3 = partial coverage; 2 = significant axes missing; 1 = most axes missing or contradicted; 0 = disposition absent from render."

**User payload** changes from `originalProse` + `rendered` to `disposition` + `rendered`:

```json
{
  "disposition": { /* the descriptor's disposition object with vocab labels */ },
  "rendered": "..."
}
```

`ProfiledEvalCase` no longer passes `originalProse` to the proximity judge. The `ProfiledEvalCase` record or `ProximityJudge.evaluate()` signature changes accordingly.

`originalProse` remains on `AgentProfile` for reference — it is not removed. It is simply no longer the proximity eval reference.

---

## What Does Not Change

- `SystemPromptRenderer` SPI and `RenderedPrompt` record — unchanged externally
- `RenderedPromptCache` SPI — unchanged. `TEMPLATE_HASH` changes automatically invalidate cache entries
- A2A format assembly — unchanged; enrichment already skipped for `A2A_CARD`
- `A2ASemanticEnrichmentStep` — out of scope; separate concern
- Structural fallback contract — any enrichment failure still produces correct structural output for all sections
- Enrichment format predicate (`usesEnrichment`) — `MARKDOWN` and `PROSE` use enrichment; `A2A_CARD` does not. No change

---

## Issues

| # | Scope | Depends on |
|---|-------|-----------|
| New: enrichment-mechanics | Changes 1 + 2 + 3: narrow scope, selective override, extractJson, retry, payload narrowing | — |
| New: briefing-field | Change 4: `AgentDescriptor.briefing` + enrichment payload inclusion + structural fallback | enrichment-mechanics (validates enrichment works before testing briefing's effect) |
| New: proximity-eval-redesign | Change 5: redefine proximity target, update judge prompt and payload | enrichment-mechanics (so proximity runs against fixed enrichment renders) |

---

## Validation

After implementing Changes 1–3 (enrichment-mechanics issue):
- Re-run `evaluateRealWorldScenarios` with Claude
- Target: enrichment success rate >80% (currently ~0%)
- Disposition sections should be prose rather than bullet lists
- Quality scores: unchanged or improved for MARKDOWN/PROSE formats
- Run `evaluateWithIndependentJudge` with Qwen 8B to check for self-eval bias on enriched renders

After implementing Change 4 (briefing-field issue):
- Populate `briefing` in eval profiles from `vocabularyGap: FULL` entries
- Re-run `evaluateRealWorldScenarios`
- Run redesigned proximity eval (Change 5) — expect improvement for FULL-LOSS cases

After implementing Change 5 (proximity-eval-redesign issue):
- Proximity scores should be high for structural renders (descriptor fields are correctly expressed)
- Verify FACTUAL_FIDELITY and redesigned proximity are measuring complementary things in practice
