# eidos Session Handover — 2026-06-14

## Last Session

Closed eidos#50 and eidos#51 (Phase 2). PROXIMITY_FLOOR=3.0 confirmed from 16-case Qwen 3 8B run
(min=3, mean=3.5). `evaluateWithIndependentJudge()` added — loads `target/renders-cache.json` and
re-judges with any configured ChatModel to detect self-evaluation bias; Ollama eval infrastructure
fully debugged and documented. TEMPLATE_HASH now covers PROMPT_TEMPLATE + A2A_PROMPT_TEMPLATE +
all RESPONSE_FORMAT/A2A_RESPONSE_FORMAT schema descriptions (protocol PP-20260614). parent#232
closed; casehub-eidos.md Current State filed as parent#245.

## Immediate Next Step

Run `evaluateWithIndependentJudge()` with Qwen 35B — the command is in CLAUDE.md (independent
judge section). Phase 2a renders must exist at `eval/target/renders-cache.json` first; if stale
or missing, re-run `evaluateRealWorldScenarios()` with Ollama (CLAUDE.md has both commands).

## What's Left

- **parent#245** — casehub-eidos.md Current State: add eidos#50 complete, eidos#51 Phase 2 complete · XS · Low
- **eidos#52** — Claude CLI subprocesses survive `atMost()` cancellation — platform-level bug in subprocess lifecycle; `activeSessions` set should `destroyForcibly()` on cancellation · M · High
- **eidos#53** — `allCasesComplete()` fails for LLM-enriched renders (capability names paraphrased) — needs semantic check or `RenderedPrompt.enriched()` flag gate · S · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Run evaluateWithIndependentJudge (Qwen 35B vs Claude baseline) | XS | Low | See Immediate Next Step |
| — | Phase 3: evaluateBehavioralScenarios(), calibrate ACCURACY_FLOOR | M | Med | Depends on Phase 2 results |
| #28 | casehub-engine: Belbin-based agent composition | L | High | casehubio/eidos repo |

## References

- Proximity baseline (2026-06-14): `eval/target/proximity-report.json` (transient; re-run Ollama eval)
- Blog: `blog/2026-06-14-mdp01-the-eval-fights-back.md`
- Protocol (new): `docs/protocols/renderer/template-hash-input-coverage.md` (PP-20260614)
- CLAUDE.md: Ollama eval commands (timeout=300s, mvn clean required), independent judge run command
- Garden: GE-20260614-94c366 (Ollama augmentation cache), GE-20260614-1ece0f (Ollama 10s timeout), GE-20260614-337397 (@DefaultBean + Ollama ambiguity)
- Ollama: running on localhost:11434 — qwen3.6:35b-a3b for independent judge, qwen3:8b for calibration
