---
layout: post
title: "eidos gets a memory"
date: 2026-06-03
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-eidos]
tags: [knowledge-graph, jpa, wilson, poc-gate, phase-4]
---

casehub-eidos knows what agents *claim* to be. Phase 4 gives it a record of what agents have actually done.

The gap has been obvious for a while. You can declare an agent as bold and risk-accepting, capable in code review, strong in Rust. The system stores that and renders it into prompts. But there's nothing connecting what was declared to what happened at runtime — no "did this agent do Rust code reviews, and how did those go?" The knowledge graph is the answer to that question.

## Fixing the foundation first

Before writing a single graph entity, we had two schema correctness issues to address. Both came from the same root cause: `agentId` is not globally unique across tenancies. The same agent persona can exist in multiple tenancies, so any method that accepts only `agentId` can't be correctly implemented by a store that is tenancy-scoped.

The `AgentStateStore` SPI had this problem. Three methods, none of them took `tenancyId`. I added it across the SPI and all implementations — not painful, because the platform has no deployed instances. The lesson is now a protocol so the next SPI doesn't repeat the mistake.

The second problem was `agent_descriptor` using `agent_id` as its primary key. That's the natural key, but it's only unique within a tenancy, not globally. The right design is a BIGSERIAL surrogate PK with a unique constraint on `(agent_id, tenancy_id)`. Rewrote V1. Clean slate.

## Prove it before building it

The design spec for Phase 4 included a hard gate: validate the API and query semantics with a pure in-memory POJO before touching JPA. I wanted to know that the Wilson lower bound formula produces the right ordering, that the semantic enricher actually changes routing decisions (not just in theory), and that the sufficiency model is honest about small samples.

So we built `InMemoryAgentGraph` first — no annotations, no CDI, no framework. Just the candidate API exercised directly. The critical test was `v2_routingPicksDifferentAgentThanV1`: identical data, two graphs — one with the enricher, one without — and the enricher has to flip the routing decision. Not just make it available; actually change which agent wins.

The Wilson formula earned its place there. An agent with 20 code-review outcomes at 0.78 average quality beats one with 5 outcomes at 0.90, because the Wilson lower bound penalises small sample counts even when raw quality is higher. Agent B's 0.90 looks better but translates to a Wilson score of ~0.53; Agent A's 0.78 across 20 samples gets ~0.60. More data wins, which is the right behaviour for a routing signal — you don't want a single high-confidence result overriding an established track record.

The PoC caught a design flaw immediately: my original scenario for the enricher comparison had the wrong numbers. At n=3 with confidence 0.95, agent B's Wilson score already beats Agent A's n=10 at 0.70 *without* the enricher — so V1 would never have picked Agent A first. We fixed the scenario, established genuine divergence, then proceeded to JPA.

## The H2 surprise

Building `JpaAgentGraphStore`, I used `INSERT ... ON CONFLICT (ledger_entry_hash, tenancy_id) DO NOTHING` for idempotent attestation writes. Reasonable assumption: H2 in `MODE=PostgreSQL` supports standard PostgreSQL syntax. It doesn't. H2 2.4.240 rejects `ON CONFLICT` with a syntax error.

The fix is a JPQL existence check + conditional persist within `@Transactional`. Less elegant than native upsert, but portable across H2 and PostgreSQL, and safe within a transaction boundary. Claude flagged a similar class of problem later — `historyByCapability` was returning all-agent outcomes regardless of capability tag, a silent data cross-contamination — and the same "check the scope of every query" discipline applied.

## What the graph adds

The graph is the layer that connects declared identity to observable behaviour. An `AgentTask` records that an agent was dispatched to exercise a capability in a domain. An `AgentOutcome` records how it went — succeeded, partially, failed — with a confidence value. An `AttestationRef` links that outcome to a casehub-ledger entry hash, so the trust score chain is auditable: score → outcome history → specific ledger evidence.

The `TaskSemanticEnricher` SPI is the platform's way of staying domain-agnostic while still enabling richer analysis. eidos doesn't know what "code-review in a security-sensitive codebase" means for riskAppetite. But devtown does. The enricher is a pull interface — eidos asks, the application answers, eidos applies without knowing the domain. Without an enricher, the graph still works for structural queries: task history, outcome stats, attestation chains. The enricher unlocks personality axis correlation and semantic domain equivalence.

The new `casehub-eidos-graph` module follows the existing eidos Jandex library pattern — activates by classpath presence, no quarkus:build goal. The V3 migration creates three tables alongside the schema corrections in V1 and V2.

The platform has never had this layer before. Identity declared (descriptor), capability probed (health), task assigned (engine), outcome recorded (graph), evidence attested (ledger), trust scored (ledger) — that's the full loop. Phase 4 closed the gap between assignment and evidence.
