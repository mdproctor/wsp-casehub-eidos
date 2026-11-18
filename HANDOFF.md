# HANDOFF — eidos

## Last Session

Closed #234 and #249 in one session.

**#234 (situational personality profiles)** — found fully implemented on main from a prior session. All contract items present: `AmpdFacetShift`, `SituationalProfile`, closed mode vocabulary (5 modes), `archetype-situational-profiles.yaml` (12 families x 5 modes = 60 profiles), `SituationalProfileDeriver` wired into bootstrap, YAML deserialization, validation, full test coverage. Closed the GitHub issue.

**#249 (dialect-aware upsert)** — designed, implemented, reviewed, landed, published.

| Change | What landed |
|--------|-------------|
| `JpaDialect` utility | New package-private class in `runtime/.../jpa/`. Detects PostgreSQL vs H2 via JDBC URL from EntityManager properties. |
| `JpaDispositionSignalStore` | `recordActivation()` now selects PostgreSQL `INSERT ON CONFLICT` or H2 `MERGE` at `@PostConstruct`. Eliminates concurrent-insert race on PostgreSQL's NOT MATCHED path. |
| `JpaBehavioralSignalStore` | `record()` converted from find-then-persist to native SQL upsert (same dialect selection). Eliminates concurrent-insert race. Added `em.flush(); em.clear();` for persistence context hygiene. |
| `JpaDispositionSignalStoreTest` | New test class (9 tests) — previously missing. Uses `@BeforeEach` cleanup instead of `@TestTransaction` because `recordActivation()` is `REQUIRES_NEW`. |

Published to origin (2 commits, rebased on 2 the remote had gained).

## Queue

11 of 12 done. 1 remaining:

| # | Title | Scale | Complexity | Notes |
|---|-------|-------|------------|-------|
| #215 | external_identity_service | M | High | Credential-to-identity resolution, ties to qhorus mesh |

## Decisions Made

- D1: Runtime dialect detection over `@IfBuildProperty` split — the SQL difference is a single string constant, not behaviour. No new config properties, no class proliferation.
- D2: Fix both stores (disposition + behavioral) in #249 — different race patterns but same fix approach.
- D3: Full native SQL upsert for behavioral store (not just PG path) — consistent pattern across both stores.

## Linked Work

- casehubio/neocortex#1060 — personality evolution design; depends on disposition signal store
- qhorus#508 — API key identity resolver SPI; #215 implements the production credential resolver

## Known Issues

- Fork mirror occasionally fails to follow origin/main on publish — catches up on next run
