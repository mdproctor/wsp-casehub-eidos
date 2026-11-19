# HANDOFF — eidos

## Last Session

Implemented the eidos companion to ledger#231 — org-proximity attestation relevance weighting.

**#248 (OrgProximityRelevanceFactor + OrgDistanceAttestationEnricher)** — two `@ApplicationScoped` CDI beans in `org-runtime/src/main/java/.../trust/` that bridge eidos org-proximity to ledger's attestation relevance pipeline:

- `OrgProximityRelevanceFactor` implements `AttestationRelevanceFactor`. Reads `"org-distance"` from the attestation snapshot, maps via configurable distance-indexed weight table (default: 1.0, 0.7, 0.65, 0.6). Null/missing/unknown → NEUTRAL. Defensive `instanceof Number` check on snapshot value to prevent ClassCastException on the trust computation hot path.
- `OrgDistanceAttestationEnricher` implements `LedgerAttestationEnricher`. Calls `OrgAwareDispatch.orgDistance()` with raw pre-tokenised identities at write time, stores LCA hop distance in the attestation's relevance snapshot JSON. Distance -1 (unknown/disconnected) → no-op. Merges into existing snapshot when other enrichers have written.

**Dependencies:** Added `casehub-ledger-api` to `org-runtime/pom.xml`. Updated `parent/docs/platform/dependency-map.md` with the new dependency edge.

**Tests:** 25 unit tests (17 factor + 8 enricher) covering weight table lookup, edge cases (null snapshot, -1/0 distance, beyond-table bounds, non-Number values, JSON number type variations, existing snapshot merge), startup validation, and enricher-to-factor round-trip contract test.

**Docs:** Consumer guide updated with Trust Relevance subsection and module table entry. CLAUDE.md project structure updated. Blog: "Whose Evidence Matters" — covers accuracy vs independence tension, write-time snapshot pattern, mass-preserving normalisation, and the factor pipeline SPI design.

**Filed:** #256 (CDI integration test for trust beans, S/Low), #257 (independence-aware anti-collusion factor, M/High), #258 (ARC42STORIES attestation relevance update, S/Low). All scheduled in queue.

**Cross-repo:** parent repo commit `686348ef` — dependency-map update. Published.

## Immediate Next Step

#215 — external identity service (API key and credential-to-identity resolution). Scale M, Complexity High. Needs brainstorming — full SPI design for credential registry, dynamic registration, key rotation, and resolution. Different domain from org-proximity trust — fresh session recommended.

## Queue (6 open)

| # | Title | Scale | Complexity |
|---|-------|-------|------------|
| #215 | external identity service | M | High |
| #252 | concurrent upsert integration test | S | Low |
| #253 | extract signal store contract tests | S | Med |
| #256 | CDI integration test for trust beans | S | Low |
| #257 | independence-aware anti-collusion factor | M | High |
| #258 | ARC42STORIES attestation relevance update | S | Low |

## References

- Blog: `docs/blog/2026-11-19-mdp01-whose-evidence-matters.md`
- Ledger spec: `ledger/docs/specs/issue-231-org-scoped-attestation-weighting/2026-11-18-attestation-relevance-design.md`
- Source: `org-runtime/src/main/java/io/casehub/eidos/org/runtime/trust/`
