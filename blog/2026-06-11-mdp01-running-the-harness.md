---
layout: post
title: "Running the Harness"
date: 2026-06-11
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-eidos]
tags: [eval, local-models, jlama, tornadovm, ollama, renderer, disposition]
---

The eval harness has been sitting in the repo since eidos#16. This session was supposed to be simple: run it, read the numbers, set the floor constants. It took two days and uncovered three separate bugs before a single score came back clean.

## The bugs

The first was embarrassing: the `-Peval` Maven profile was documented in `PromptEvalTest.java` but never actually defined in the pom. The surefire default excludes `@Tag("eval")` tests. Running `-Peval` did nothing, and zero tests executed. Fixed by adding the profile with `combine.self="override"` to clear the exclusion.

The second was subtler. Every enrichment step — `SemanticEnrichmentStep`, `A2ASemanticEnrichmentStep`, and both reactive variants — called `mapper.readTree(json)` on raw LLM output. When the Claude CLI wraps its JSON response in backtick code fences (which it does, routinely), `readTree` throws, the exception is caught silently, and the renderer falls back to structural output. Structural output is third-person, documentation-style prose. The SECOND_PERSON score for MARKDOWN was 1.2/5.0. For PROSE it was 0.00/5.0. These aren't quality scores — they're silent failure scores. The fix was a `stripCodeFences()` method applied to all four enrichment steps and both judges.

The third was the completeness check treating `sprint-planning` as a different string from `sprint planning`. The semantic enricher writes natural prose; the check expected exact slug matches. Normalized comparison fixed it.

With all three resolved, the first clean baseline run:

| Format | Score | All complete |
|--------|-------|-------------|
| MARKDOWN | 3.95 / 5.0 | ✅ |
| PROSE | 4.50 / 5.0 | ✅ |
| A2A_CARD | 5.00 / 5.0 | ✅ |

Claude Opus 4.6. Nine eval cases. About nine minutes end to end, most of it API latency.

## Into the local model wilderness

The original plan after baseline was a Jlama comparison. That turned into a two-day investigation of every pure-Java and local inference path available on Apple Silicon.

**Jlama 0.26.1**: Immediate boot failure. Built against `ChatLanguageModel`; LangChain4j 1.14.1 renamed that to `ChatModel`. Class not found at Quarkus bootstrap.

**Jlama 1.9.1, no native library**: Got past bootstrap. Then: `UnsupportedOperationException: ARM_128` in `PanamaTensorOperations.batchDotProduct`. The F32×Q4 computation path has switch cases for AVX_256 and AVX_512 but no ARM_128 case. Throws immediately. Checked current HEAD on GitHub — same gap. No fix in progress.

**Jlama 1.9.1 + jlama-native**: The native jar (`jlama-native-0.8.4-osx-aarch_64.jar`) contains `libjlama.dylib` — a compiled NEON implementation that includes `gemm_f32_q4` for ARM. Adding it to the eval classpath bypassed Panama entirely and loaded `NativeSimdTensorOperations`. Inference ran: 45 minutes for nine judge calls, roughly 5 tokens per second for Llama 3.2 3B on M4 Max CPU via NEON. The run crashed at the end with a non-JSON judge response — the 3B model doesn't reliably follow JSON output instructions when acting as both renderer and judge.

**GPULlama3 via TornadoVM OpenCL**: The 1B model ran on GPU in about 4 minutes per case. Too small for consistent JSON output. The 3B model hit `TornadoOutOfMemoryException` — Apple's OpenCL driver caps single buffer allocations around 448MB, and one 3B tensor exceeded that. Fixed with `-Dtornado.device.memory=32GB` (a TornadoVM software limit, not the hardware ceiling — the M4 Max has 36GB unified memory and `tornado --devices` confirms it). With the fix, the 3B ran for 35 minutes but still produced non-JSON output on some judge calls.

**GPULlama3 via TornadoVM Metal**: The Metal backend shipped separately in the v4.0.0 release. It's confirmed working — `MetalCommandQueue`, `MetalKernelScheduler`, `MetalInstalledCode` all appear in the stack traces. But the throughput is disappointing: 112 minutes for nine cases, significantly slower than OpenCL. The TornadoVM team's own documentation says "use Apple Silicon for development, NVIDIA for performance testing" — the Metal backend is a first implementation, not yet optimized. This one will improve.

**Ollama**: Already installed with `llama3.2:latest`, `qwen3.6:35b-a3b`, and others. Adding `quarkus-langchain4j-ollama` to the eval profile took about ten minutes. The critical flag: `format=json` enables Ollama's JSON-constrained sampling, which forces structurally valid JSON output regardless of whether the model follows the system prompt instruction to do so. First run: 75 seconds for all nine cases.

## The comparison

| Model | Time | MARKDOWN | PROSE | A2A_CARD | Pass? |
|-------|------|----------|-------|----------|-------|
| Claude Opus 4.6 | ~9 min | **3.95** | **4.50** | **5.00** | ✅ |
| Llama 3.2 3B (Ollama Metal) | **75s** | 3.90 | 4.50 | 4.50 | ✅ |
| Gemma3 4B (Ollama Metal) | 16.6 min | 4.25 | 4.25 | 4.75 | ❌ completeness |
| Llama 3.1 8B (Ollama Metal) | ~38 min† | 4.30 | 4.63 | 4.50 | ✅ |
| Qwen3.6 35B MoE (Ollama Metal) | ~52 min | 3.60 | 4.75 | 5.00 | ❌ completeness |

†The 8B run hit memory pressure on the M4 Max — the machine was swapping. The 38-minute time is not a reliable inference benchmark and the scores may be affected.

Two caveats apply to all local model rows. First, the same model acts as both renderer and judge — it writes the system prompt and then scores it. Self-evaluation bias is real and visible: Qwen judges its own MARKDOWN rendering with FACTUAL_FIDELITY=3 and flags "plan sprint 42 is ungrounded" — which is wrong, because sprint 42 comes from the `AgentPromptContext`, a valid render-time input the judge never sees. Second, the llama3.2:3b row shows significant variance: 3.90 MARKDOWN on the first run, 2.40 on a repeat run 90 minutes later. At 3B, the random seed matters.

The `allCasesComplete=NO` failures for Gemma and Qwen are the hyphen normalization edge case — the completeness checker finds `sprint-planning` as a slug but the model wrote "sprint planning" or "sprint plans" in prose that the normalized check still misses in some formulations.

## What the rendered output actually looks like

For the devtown-planner case with a PROSE render, the same input descriptor produces:

**Claude Opus 4.6:**
> You are Devtown Planner (v1.0), an Anthropic agent running on Claude 3.5 Sonnet. Your role is sprint planner — you own the process of turning a raw backlog and team capacity into a concrete, actionable sprint plan. You bridge strategic priorities and team execution by sizing work, sequencing it, and ensuring the sprint is realistic.

**Qwen 3.6 35B:**
> You are Devtown Planner, version 1.0, running on the Claude 3.5 Sonnet model provided by Anthropic. You operate as a dedicated planner tasked with structuring and organizing development workflows. Your function is to translate raw requirements and team parameters into actionable, time-boxed execution cycles.

Both are usable. Claude's is more natural and role-first. Qwen's is more literal. The A2A cards were near-identical across all models in structure, differing only in how they phrased capability descriptions:

- Claude: *"drawing on strong agile expertise and solid kanban knowledge to balance scope against available effort"*
- Qwen: *"You leverage agile and kanban principles to optimize workflow and delivery timelines"*

Claude's version is more specific and grounded in the descriptor's domain values. Qwen's reads slightly more generic.

## The design question

Looking at the rendered outputs — particularly `qualityHint: 0.9`, `latencyHintP50Ms: 200`, `epistemicDomains: {agile: 0.9}` appearing in PROSE and MARKDOWN — raised an honest question about what these fields are actually for.

They're routing signals. casehub-engine uses them to pick which agent handles a task. Telling a language model "you have agile expertise at 0.9" doesn't give it more agile knowledge — it shapes tone and confidence level at best. And the scores are self-declared today with no calibration mechanism. A model expressing high confidence on agile questions it gets wrong is actively misleading for routing, which is exactly what the research on LLM calibration shows (Scale AI's calibration studies found errors exceeding 80% paired with sub-10% accuracy at launch).

The fix is narrow: `epistemicDomains`, `qualityHint`, `latencyHintP50Ms`, `costHint` render only in A2A_CARD. PROSE and MARKDOWN render capability names only. A renderer protocol captures this. The fields stay in the descriptor — they're the right structure for calibration once there's a measurement path.

What calibration would look like: domain-specific benchmark task sets authored by practitioners (5–10 questions per declared domain with known-good answers), run against the agent, accuracy and calibration score stored as the descriptor values. The calibration score — does the model's confidence correlate with its correctness? — is what would eventually determine whether to render "this is your core domain" (authoritative stance) vs "this is outside your primary expertise" (hedge). The eval harness we just wired is structurally the right infrastructure for this. It needs domain task sets and a feedback loop back to the descriptor.

## What's open

The local inference story on Apple Silicon is unfinished. Jlama NEON works but runs at around 5 tokens per second for a 3B model. TornadoVM Metal is confirmed but not yet optimized. Ollama via llama.cpp Metal is the practical path right now — fast, reliable JSON output, easy model switching. The Jlama Q8 comparison remains open; no pre-quantized Q8 Llama 3.2 exists from tjake, and creating one requires the raw BF16 model and a HuggingFace token.

The comparison that matters most — separating renderer quality from judge quality by using an independent judge model — hasn't been run yet. Using Qwen 35B as judge on Claude-rendered outputs would give a cleaner signal than any self-evaluation run.
