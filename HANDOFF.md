# HANDOFF — eidos

## Last Session

Brainstormed eidos#215 (external identity service — API key and credential-to-identity resolution). The design analysis determined that credential management is a **platform concern**, not an eidos concern. Platform already owns `StaticCredentialRecord`, `StaticCredentialStore`, `CredentialType`, and authentication infrastructure. Eidos's `AgentRegistry.findById(actorId, tenancyId)` already closes the identity resolution loop once authentication produces a `CurrentPrincipal`.

**eidos#215** — closed, superseded by platform#900.

**platform#900** (credential resolution + rotation for StaticCredentialStore) — filed, implemented, verified, and closed in the same session. Adds `lookup(credential)`, rotation with grace period, `CredentialState { ACTIVE, GRACE_PERIOD, REVOKED }`, and SHA-256 hashed storage to `StaticCredentialStore`.

**platform#901** — filed for deferred dynamic self-registration (blocked on self-organising societies design).

**casehubio/qhorus#527** — filed for the thin adapter that wires `ApiKeyIdentityResolver` to platform's `StaticCredentialStore.lookup()`. XS/Low.

**Blog:** "The Feature That Moved House" — brainstorming session that redirected credential management from eidos to platform.

## Immediate Next Step

#252 — concurrent upsert integration test. Scale S, Complexity Low. Testcontainers-based PostgreSQL tests for the signal store dialect-aware upsert. Different domain from this session — fresh session recommended.

## Queue (5 open)

| # | Title | Scale | Complexity |
|---|-------|-------|------------|
| #252 | concurrent upsert integration test | S | Low |
| #253 | extract signal store contract tests | S | Med |
| #256 | CDI integration test for trust beans | S | Low |
| #257 | independence-aware anti-collusion factor | M | High |
| #258 | ARC42STORIES attestation relevance update | S | Low |

## References

- Design artifacts: `docs/specs/issue-215-external-identity-service/decisions.md`
- Blog: `blog/2026-11-20-mdp01-the-feature-that-moved-house.md`
- Platform credential store: `StaticCredentialStore` in `io.casehub.platform.api.authn`
- Qhorus auth: `ApiKeyIdentityResolver` in `io.casehub.qhorus.api.spi`
