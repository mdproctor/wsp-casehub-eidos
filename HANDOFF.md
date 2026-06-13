# eidos Session Handover — 2026-06-13

## Last Session

Implemented PP-20260611-228599 (eidos#49, now closed): `buildDescriptorPayload` is now
format-discriminated — PROSE/MARKDOWN gets `{name, inputTypes?, outputTypes?}` only;
A2A_CARD gets full numeric routing signals + type schema. `assembleA2aCard()` now carries
`latencyHintP50Ms`, `costHint`, `epistemicDomains`, `inputTypes`, `outputTypes`. Spec went
through four design-review rounds; the push-back on costHint exclusion (cache correctness
trap when structural assembly reads from record directly) went into garden and protocol.
3 garden entries (GE-20260613-3fa95a/53e590/718a57) and 1 new protocol (PP-20260613-608684).

## Immediate Next Step

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## What's Left

- **Independent judge comparison** — run Qwen 35B as judge on Claude-rendered outputs
  (separates renderer quality from self-evaluation bias) · S · Low
- **parent#232** — PLATFORM.md "System prompt generation" entry stale (missing A2A_CARD
  capability fields); filed on casehubio/parent · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Phase 2: evaluateRealWorldScenarios(), calibrate PROXIMITY_FLOOR | S | Low | See Immediate Next Step |
| — | Phase 3: evaluateBehavioralScenarios(), calibrate ACCURACY_FLOOR | M | Med | Depends on Phase 2 |
| — | Independent judge comparison: Qwen 35B judging Claude renders | S | Low | Ollama already running |
| #50 | Extend TEMPLATE_HASH to cover RESPONSE_FORMAT schema descriptions | S | Low | Cache gap — RESPONSE_FORMAT changes not self-invalidating |
| #28 | casehub-engine: Belbin-based agent composition | L | High | Filed in casehubio/eidos repo |

## References

- Baseline artifact: `eval/src/test/resources/eval-baseline-2026-06-10.json`
- Blog: `blog/2026-06-13-mdp01-a2a-card-needed-more.md`
- Protocol (implemented): `docs/protocols/renderer/capability-metadata-rendering.md` (PP-20260611-228599)
- Protocol (new): `docs/protocols/renderer/a2a-structural-assembly-hash-coverage.md` (PP-20260613-608684)
- CLAUDE.md: eval run commands for all four backends documented; A2A_CARD capability fields updated
- TornadoVM Metal: `~/tornadovm-metal/tornadovm-4.0.0-jdk25-metal` (installed, argfile generated)
- Ollama: running on localhost:11434 with llama3.2, llama3.1:8b, gemma3:4b, qwen3.6:35b-a3b
