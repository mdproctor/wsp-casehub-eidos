# HANDOFF — eidos

## Last Session

Three testing issues landed on main from branch `issue-252-concurrent-upsert-integration-test`:

**#252** — Concurrent upsert integration test. `PostgresConcurrentUpsertTest` with 3 tests: 10 virtual threads × 10 records per thread hitting the same PK, with retry on transient PostgreSQL errors. Verifies no data loss for all three JPA stores.

**#253** — Contract test extraction. `BehavioralSignalStoreContractTest` (20 tests) and `DispositionSignalStoreContractTest` (9 tests) in api test-jar. PostgreSQL behavioral coverage 5→20, disposition 6→9. Discovery: Quarkus `@TestTransaction` does not wrap inherited `@Test` methods.

**#256** — CDI integration test for trust beans. `TrustBeansCdiTest` in org-runtime verifying config property injection and OrgAwareDispatch resolution.

**Blog:** "Three Gaps Past Green" — published.
**Issues filed:** #259 (standardise agent state cleanup), #260 (in-memory contract tests).
**All repos published.** Branch squashed, rebased, landed, closed.

## Immediate Next Step

#257 — independence-aware anti-collusion factor. Scale M, Complexity High. Fresh session recommended — different domain (trust/collusion vs testing infrastructure).

## Queue (4 open)

| # | Title | Scale | Complexity | Blocked by |
|---|-------|-------|------------|------------|
| #257 | independence-aware anti-collusion factor | M | High | — |
| #258 | ARC42STORIES attestation relevance update | S | Low | — |
| #259 | standardise contract test cleanup | XS | Low | — |
| #260 | adopt contract tests for in-memory stores | XS | Low | — |

## References

- Contract test pattern: `AgentStateStoreContractTest` in `api/src/test/java/`
- Blog: `docs/blog/2026-11-20-mdp02-three-gaps-past-green.md`
