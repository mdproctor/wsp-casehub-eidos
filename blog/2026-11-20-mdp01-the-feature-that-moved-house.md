---
layout: post
title: "The Feature That Moved House"
date: 2026-11-20
entry_type: note
subtype: diary
projects: [casehubio/eidos, casehubio/platform]
tags: [identity, credentials, architecture, platform, brainstorming]
---

We sat down to design an external identity service for eidos — API key management, credential-to-identity mapping, key rotation with grace periods. The issue had been open for months, labelled M/High, waiting for a brainstorming session. It looked like a solid week of SPI design and implementation.

Ten decisions in, we had a clean architecture: an `AgentCredential` type, a `CredentialIdentityStore` SPI with issue/resolve/rotate/revoke, in-memory and JPA implementations following the established CDI ladder, a bridge module for the qhorus adapter. The design was internally consistent. It passed review.

Then the question that undid it: *"why aren't we just using platform?"*

## The overlap nobody mentioned

Platform already had `StaticCredentialRecord`, `StaticCredentialStore`, `CredentialType { API_KEY, BOT_TOKEN, PERSONAL_ACCESS_TOKEN }` — shipped months ago in platform#548. It had `WorkerCredentialStore` with full CRUD. It had the `CurrentPrincipal` identity contract and authentication infrastructure. Qhorus had already built `ApiKeyIdentityResolver` with a config-based `@DefaultBean` and an `ApiKeyAuthenticationMechanism` that converts credentials to `SecurityIdentity`.

The issue's premise — "eidos is the natural home because it owns the agent registry" — confused consuming identity with producing it. Eidos *consumes* identity: given an authenticated `actorId`, `AgentRegistry.findById()` returns the full agent descriptor. That path already works. The credential store — issuing keys, hashing them, rotating with grace periods, resolving on the auth hot path — is infrastructure. Infrastructure is platform's job.

## What actually happened

We closed eidos#215 and filed platform#900 instead. The four gaps in platform's existing credential store:

1. **`resolve(credential)`** — lookup by credential value, not by identity. The auth path.
2. **Rotation with grace period** — old key stays valid for a configurable TTL during propagation.
3. **Credential state** — `ACTIVE`, `GRACE_PERIOD`, `REVOKED`. The existing store had no state concept.
4. **Hashed storage** — SHA-256 in JPA, `MessageDigest.isEqual()` for constant-time comparison.

Platform#900 landed the same day. Three commits: `CredentialState` enum, in-memory implementation with constant-time lookup, JPA entity with hashed storage and Flyway migrations. The qhorus adapter — a 10-line class that maps `StaticCredentialRecord` to `ResolvedIdentity` — can now displace the config-based default.

## The design lesson

The brainstorming process worked exactly as intended, but not in the way we expected. We didn't produce a spec for eidos. We produced evidence that the feature belonged somewhere else. That's not a failure — it's the process catching a scope error before implementation amplified it.

The red flag was hiding in the first question. When every design decision mapped cleanly to an existing pattern in another repo, the right response isn't "great, our design is validated by prior art." It's "why are we building this at all?"

A brainstorming session that concludes "don't build this here" is worth more than one that produces a spec for the wrong repo. The cheapest code is the code you redirect before writing it.
