# eidos Session Handover — 2026-06-12

## Last Session

Closed #47 (eval test polish) and #48 (Phase 1 eval baseline). Fixed three silent bugs in
the enrichment pipeline (missing `eval` Maven profile, code-fence fallback in all four enrichment
steps, hyphen normalization in completeness check). Calibrated `SCORE_FLOORS` from Claude Opus 4.6
baseline: MARKDOWN 3.95 / PROSE 4.50 / A2A_CARD 5.00. Ran full local model comparison via Jlama
NEON, TornadoVM Metal, and Ollama (5 models). Captured PP-20260611-228599 (capability numeric
metadata renders only in A2A_CARD). Branch hygiene complete — all 24 branches closed.

## Immediate Next Step

Run Phase 2:
```
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -Peval \
  -Dtest=PromptEvalTest#evaluateRealWorldScenarios
```
Inspect `target/real-world-eval-report.json` — exercises real-world profiles and proximity judge.
Set `PROXIMITY_FLOOR` from observed mean.

## What's Left

- **PP-20260611-228599 implementation** — renderer protocol captured but not implemented:
  `EidosSystemPromptRenderer` still renders `epistemicDomains`/`qualityHint`/`latencyHint` in
  PROSE/MARKDOWN. Suppress numeric metadata for non-A2A formats in `EidosRenderPipeline` · M · Med
- **Independent judge comparison** — run Qwen 35B as judge on Claude-rendered outputs
  (separates renderer quality from self-evaluation bias) · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Phase 2: evaluateRealWorldScenarios(), calibrate PROXIMITY_FLOOR | S | Low | See Immediate Next Step |
| — | Phase 3: evaluateBehavioralScenarios(), calibrate ACCURACY_FLOOR | M | Med | Depends on Phase 2 |
| — | Implement PP-20260611-228599 — suppress capability metadata in PROSE/MARKDOWN | M | Med | Renderer change |
| — | Independent judge comparison: Qwen 35B judging Claude renders | S | Low | Ollama already running |
| #28 | casehub-engine: Belbin-based agent composition | L | High | Verify status — showed MERGED in cross-check |

## References

- Baseline artifact: `eval/src/test/resources/eval-baseline-2026-06-10.json`
- Eval comparison files: `/tmp/eval-comparison/` (eval-prose.md, eval-markdown.md, eval-a2a-card.md)
- Blog: `blog/2026-06-11-mdp01-running-the-harness.md`
- Protocol: `docs/protocols/renderer/capability-metadata-rendering.md` (PP-20260611-228599)
- CLAUDE.md: eval run commands for all four backends now documented
- TornadoVM Metal: `~/tornadovm-metal/tornadovm-4.0.0-jdk25-metal` (installed, argfile generated)
- Ollama: running on localhost:11434 with llama3.2, llama3.1:8b, gemma3:4b, qwen3.6:35b-a3b
