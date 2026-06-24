---
layout: post
title: "Seven Fixes and a Schema Decision"
date: 2026-06-24
type: phase-update
entry_type: note
subtype: diary
projects: [eidos]
tags: [cdi, jpa, validation, schema-design]
---

# Seven Fixes and a Schema Decision

I came into this session with a backlog of seven issues — three XS, three S, one M — all queued from the last two sessions of capability-health and registrar work. The goal was to clear them all in one pass.

The XS issues were mechanical: removing `@DefaultBean` from `CdiVocabularyRegistry` (#65, the same Pattern B fix we'd just applied to the renderer), reverting `AgentDescriptorMapper.toCapability()` to a positional constructor (#63, enforcing protocol PP-20260608-e694ab for compile-time field completeness), and fixing stale VocabularyRegistrar descriptions in ARC42STORIES.MD (#66) that still described the old imperative `void register(VocabularyRegistry)` API when the actual code is declarative `Class<?> vocabulary()`.

The S issues had more design surface. For #68 — allowing newlines in briefing — the question was how to punch a hole in `isBanned()` for one field without weakening validation everywhere else. I added an `allowedCodePoints` vararg to `validateField` and `validateOptional`, so the briefing call passes `0x000A` and everything else gets the full C0 ban unchanged. The mechanism is general enough that if another field needs an exemption later, it's one parameter.

For #67, the reactive bootstrap, I extracted the duplicate-detection logic from `AgentDescriptorBootstrap` into a shared `DescriptorCollector` that both blocking and reactive bootstraps call. The reactive version chains `Uni<Void>` registrations and blocks at startup with `.await().indefinitely()` — startup events run on the main thread, so blocking there is correct even in reactive deployments.

The JPA specialization store (#62) was straightforward persistence — composite key entity, upsert on `recordDecline`, TTL via `expires_at` column, `em.flush(); em.clear()` after bulk JPQL DELETE to avoid stale persistence context. Flyway V4 migration. The Flyway V-number collision with the graph module's V3 was a reminder to always check across modules before claiming a version.

The interesting decision was on #61 — the domain pre-filter. `excludedDomains` was stored as a JSON TEXT column on `agent_capability`. To filter at the SQL level, I had two choices: hack a LIKE query against the JSON (fragile — `%"java"%` matches `"javascript"`), or denormalize to a proper join table. No production installations means the V1 migration is mutable, so I replaced the JSON column with an `@ElementCollection` backed by `agent_capability_excluded_domain(capability_id, domain)`. The JPQL filter is a clean `NOT MEMBER OF` — works identically on H2 and PostgreSQL, no native queries, no database-specific functions. The mapper drops the JSON serialization for `excludedDomains` and passes the `Set<String>` directly.

The denormalization is the right call for anything you need to query. JSON columns are fine for opaque blobs the application reads whole — `epistemicDomains`, `inputTypes`, `outputTypes` all fit that pattern. But the moment a field becomes a filter predicate, it belongs in a proper relation.
