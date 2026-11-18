# HANDOFF — eidos

## Last Session

Completed the dialect-aware upsert trilogy and closed the verification gap.

**#250 (JpaAgentStateStore upsert)** — converted `record()` from delete-then-persist to dialect-aware native SQL upsert (PostgreSQL `ON CONFLICT` / H2 `MERGE`), matching the pattern already applied to both signal stores.

**#251 (PostgreSQL integration tests)** — added Testcontainers-based tests for all three JPA stores against real PostgreSQL 17. The tests revealed that `JpaDialect.isPostgres()` was broken under Quarkus 3.x — Agroal doesn't populate EntityManagerFactory URL properties, so the detector always returned false. Rewrote to use `DatabaseMetaData.getDatabaseProductName()` via Hibernate `Session.doReturningWork()`.

**#215 platform review** — checked the issue design against existing platform identity infrastructure (PrincipalId hierarchy, CurrentPrincipal SPI, DID resolution chain, WorkerCredentialStore). Added integration notes to the issue: must bridge into existing platform SPIs, not create a parallel identity stack.

Blog revised: "The SQL That Only Races in Production" — extended to cover the complete arc from race discovery through production-grade verification.

Filed: #252 (concurrent upsert test with retry), #253 (extract signal store contract tests). Both scheduled in queue.

## Immediate Next Step

#248 — OrgProximityRelevanceFactor and OrgDistanceAttestationEnricher. Different domain (org-layer enrichment) — fresh session recommended.

## References

- Blog: `docs/blog/2026-11-18-mdp05-the-sql-that-only-races-in-production.md`
- Design: `docs/design/2026-11-18-dialect-aware-upsert-design.md`
