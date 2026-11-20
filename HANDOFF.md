# HANDOFF — eidos

## Last Session

Three testing issues completed on branch `issue-252-concurrent-upsert-integration-test`:

**#252 — Concurrent upsert integration test.** `PostgresConcurrentUpsertTest` with 3 tests: 10 virtual threads × 10 records per thread hitting the same PK, with retry on transient PostgreSQL errors (23505/40P01/40001). Verifies no data loss for all three JPA stores (behavioral, disposition, agent state).

**#253 — Contract test extraction.** `BehavioralSignalStoreContractTest` (20 tests) and `DispositionSignalStoreContractTest` (9 tests) extracted into the api test-jar. All 4 JPA subclasses (H2 + PostgreSQL) now inherit the full contract. PostgreSQL behavioral coverage went from 5 to 20 tests. Discovery: Quarkus `@TestTransaction` does not wrap inherited `@Test` methods — explicit `@BeforeEach @Transactional` cleanup needed.

**#256 — CDI integration test for trust beans.** `TrustBeansCdiTest` in org-runtime verifies `OrgProximityRelevanceFactor` and `OrgDistanceAttestationEnricher` wire correctly via CDI: config property injection, `OrgAwareDispatch` resolution to `DefaultOrgAwareDispatch`, no ambiguous/unsatisfied dependencies.

**Code review nits fixed:** import ordering, fully-qualified names replaced with imports, unnecessary `throws InterruptedException` removed.

**Blog:** "Three Gaps Past Green" — progressive verification from SQL correctness to concurrent correctness to contract consistency to CDI wiring.

**Issues filed:**
- #259 — Standardise agent state store test cleanup (replace `@TestTransaction` with explicit cleanup). XS/Low.
- #260 — Adopt contract tests for in-memory signal stores. XS/Low.

## Immediate Next Step

#257 — independence-aware anti-collusion factor. Scale M, Complexity High. Different domain — fresh session recommended.

## Queue (4 open + 2 new)

| # | Title | Scale | Complexity |
|---|-------|-------|------------|
| #257 | independence-aware anti-collusion factor | M | High |
| #258 | ARC42STORIES attestation relevance update | S | Low |
| #259 | standardise contract test cleanup | XS | Low |
| #260 | adopt contract tests for in-memory stores | XS | Low |

## References

- Contract test pattern: `AgentStateStoreContractTest` in `api/src/test/java/`
- Blog: `blog/2026-11-20-mdp02-three-gaps-past-green.md`
- Prior SQL blog: `docs/blog/2026-11-18-mdp05-the-sql-that-only-races-in-production.md`
