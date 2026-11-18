# HANDOFF — eidos

## Last Session

Closed #244 (signal store polish) — three housekeeping items from the disposition fixes batch (#237–#242) code review:

| Change | What landed |
|--------|-------------|
| `Locale.ROOT` | `AmpdFacets.canonicalise()` now uses `toLowerCase(Locale.ROOT)` on both the lookup map build and the lookup call. Prevents a latent Turkish-İ mismatch on JVMs with non-ROOT default locale. |
| SQL MERGE upsert | `JpaDispositionSignalStore.recordActivation` replaced the update-then-insert two-statement pattern with SQL standard `MERGE INTO ... WHEN MATCHED / WHEN NOT MATCHED`. Closes the concurrent insert race for existing rows. NOT MATCHED path still races on PostgreSQL — filed #249. |
| `activationCounts` → `activationWeights` | Renamed across the full SPI chain (interface, 3 implementations, 2 callers, all tests, 4 doc files). Values are fractional after #237's int-to-double migration; name now matches semantics. |

Filed #249 (dialect-aware upsert for PostgreSQL — S/Low) and scheduled it in the queue.

Blog: "The Polish Pass" — `docs/blog/2026-11-18-mdp04-the-polish-pass.md`

Published to origin (2 commits, rebased on 2 the remote had gained).

## Queue

9 of 12 done. 3 remaining:

| # | Title | Scale | Complexity | Notes |
|---|-------|-------|------------|-------|
| #234 | situational_personality_profiles | M | Med | Next in queue |
| #215 | external_identity_service | — | — | |
| #249 | dialect-aware_upsert_for_JpaDispositionSignalStore | S | Low | Filed this session; PostgreSQL INSERT ON CONFLICT needed for production |

## Decisions Made

- D1: SQL MERGE over INSERT ON CONFLICT — H2 test compatibility. The final mile for PostgreSQL production needs a dialect-aware switch (#249).
- D2: `activationWeights` not `activationValues` or `functionWeights` — "activation" describes the source (function activations), "weights" describes the type (fractional values after decay).
- D3: Engine repo's `activationCounts` references left untouched — different concept (agent execution counts in `AbstractExecutionDriver`), not related to eidos's disposition signal store.

## Cross-Repo Impact

IntelliJ rename touched an engine blog post (`docs/blog/2026-08-10-mdp01-crash-recovery-pattern-execution.md`) — reverted immediately. The engine's `activationCounts` is a separate concept. No cross-repo breakage.

## Linked Work

- casehubio/neocortex#1060 — personality evolution design; depends on the disposition signal store being correct
- #249 — dialect-aware upsert (filed this session, follow-up to #244's MERGE)

## Known Issues

- Fork mirror occasionally fails to follow origin/main on publish — catches up on next run
- Generated API docs (`docs/guides/api/`) still reference `activationCounts` — CI regenerates on merge to main
