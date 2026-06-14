---
date: 2026-06-14
title: "The eval fights back"
phase: eval-phase2
---

Today was supposed to be a calibration day. Run the eval, read the proximity scores, set `PROXIMITY_FLOOR`, move on. Instead we spent most of it figuring out why the eval wouldn't run at all.

The first sign: `AgentProviderChatModel.doChat()` blocks on `await().indefinitely()`. Claude CLI was returning prose instead of JSON for about six cases in a row, which is fine — the enrichment step catches that and falls back to structural rendering. But then the seventh call just hung. No error. No timeout. No response. The hang detection fired after ten minutes.

The root cause is nastier than a simple timeout: when `atMost()` cancels the Mutiny subscription, the underlying Claude CLI subprocess keeps running. After sixteen render calls — each timed out or falling back — you have sixteen zombie Claude processes. The system is resource-starved before the judge phase even starts. The fix inside `AgentProviderChatModel` was straightforward (switch to `await().atMost(Duration.ofMinutes(7))`), but that just makes the hang bounded. The actual bug is in the platform's subprocess lifecycle, which doesn't kill the process on cancellation. Filed that as eidos#52.

Since Claude CLI wasn't reliable for batch sequential calls, we switched to Ollama. That turned out to be its own adventure. The Ollama JAR wasn't in the local Maven repository when I first tried. After forcing a resolve, the augmentation cache from the previous build was still active — Quarkus reused it, ignored the new extension, and logged the Ollama config keys as "unrecognized". The fix is `mvn clean` on the first Ollama run. Obvious in retrospect; the error message gives no hint.

Once Quarkus re-augmented with the Ollama extension, the REST client hit a 10-second timeout on every judge call. Qwen 3 8B takes about 30 seconds per response locally. The default timeout is documented nowhere — it's only findable in the extension source. The flag is `quarkus.langchain4j.ollama.timeout=300s`. All three of these Ollama issues are now in the garden.

After that, the eval actually ran. Sixteen real-world profiles, MARKDOWN and PROSE formats, ProximityJudge comparing rendered output against the original prose. Results: every case scored 3 or 4, mean 3.5. `PROXIMITY_FLOOR=3.0` is confirmed — it sits at the minimum observed score, which is the right place for a floor intended to catch identity mismatches.

The `allCasesComplete()` assertion failed for both formats. That check looks for capability names verbatim in the rendered output. Qwen's semantic enrichment paraphrases the names rather than repeating them literally, so a capability called `"sprint-planning"` might appear as "sprint planning and estimation" — present in spirit, absent as an exact match. That's eidos#53 now. For the eval to work with LLM-enriched renders, the check needs to be semantic, not string-matching.

The other piece of the session was a cache correctness fix I'd been aware of but hadn't addressed. `TEMPLATE_HASH` gates the enrichment cache — if the hash changes, the cache is invalidated. The hash was computed from `PROMPT_TEMPLATE` only. But `RESPONSE_FORMAT` schema descriptions also influence LLM output: they tell the model what to put in each JSON field. A change to those descriptions would go undetected, serving stale enriched prompts indefinitely. The fix was to extract all schema description strings into named constants (`RESPONSE_FORMAT_SCHEMA_DESCRIPTIONS`, `A2A_RESPONSE_FORMAT_SCHEMA_DESCRIPTIONS`) and include them in the hash fingerprint alongside both prompt templates.

What makes this interesting is that the design made the gap invisible. The schema descriptions lived inline inside the `ResponseFormat` builder — there was no obvious connection between them and the cache key. Extracting them as named constants makes that connection explicit: the same strings that go into the `ResponseFormat` builder are the ones that get hashed. Now a developer editing a description will change the hash by construction, not by discipline. Protocol PP-20260614 formalises this.

The session ended with `evaluateWithIndependentJudge()` — a new eval method that loads previously-rendered outputs from `target/renders-cache.json` and re-judges them with whatever model is configured. The point is to run Claude's renders through a different judge (Qwen 35B) and compare scores. If Qwen agrees with Claude's self-assessment, the baseline numbers are credible. If Qwen scores lower, Claude was being generous to itself. That run still hasn't happened — we have the infrastructure, but not the comparison data yet.
