# Idea Log

Undecided possibilities — things worth remembering but not yet decided.
Promote to an ADR when ready to decide; discard when no longer relevant.

---

## 2026-06-11 — Empirical calibration of epistemicDomains and capability metadata

**Priority:** high
**Status:** active

`epistemicDomains`, `qualityHint`, `latencyHintP50Ms`, and `costHint` are self-declared by developers today — there's no mechanism to validate or measure them. For routing they're useful as relative signals; for system prompt rendering they're noise until calibrated. The path: define domain-specific benchmark task sets (5–10 questions per declared domain, practitioner-authored with known-good answers), run the agent through them, record accuracy and calibration score (does confidence correlate with correctness?). Store the measured profile in the descriptor instead of declared values. Rendering implication: once calibrated, surface the *behavioural stance* implied by the score ("this is your core domain / adjacent / outside your expertise") rather than a number the LLM can't act on.

**Context:** Surfaced during eidos#48 Phase 1 eval run and local model comparisons. Running the eval harness against Ollama models made clear that the same descriptor rendered by a 3B model includes latency hints and quality scores as literal prose — judges penalised this as FACTUAL_FIDELITY issues. The eval scores themselves are the right calibration infrastructure; they just need domain-specific task sets and a feedback loop back to the descriptor. General LLM domain evaluation research (HealthBench, HELM, Scale AI calibration studies) confirms the overconfidence problem: models express high confidence on wrong answers, meaning uncalibrated scores actively mislead routing.

**Promoted to:**

---

## 2026-06-10 — Compare eval quality scores: Claude Opus 4.6 vs Jlama GGUF

**Priority:** medium
**Status:** active

Run `evaluateAllScenarios()` with `-Peval-jlama` once a GGUF model file is available, and compare the resulting `eval-report.json` against the Claude Opus 4.6 baseline. The `modelLabel` field already distinguishes runs. Primary interest: whether FACTUAL_FIDELITY and SECOND_PERSON/TONE scores diverge between model families — local models hallucinate differently and may handle style instructions less reliably.

**Context:** Came up during Phase 1 baseline calibration (eidos#48). The `-Peval-jlama` Maven profile already exists; the only blocker is the GGUF model file. Garden entry GE-20260423-878486 documents a known Jlama bootstrap issue on Quarkus 3.32+.

**Promoted to:**
