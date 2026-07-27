---
layout: post
title: "The fields a descriptor doesn't ask about"
date: 2026-07-27
type: phase-update
entry_type: note
subtype: diary
projects: [eidos]
tags: [api-design, querying, validation]
---

The previous session landed descriptor templates — reusable prose fragments for agent system prompts. This session went broader: four issues in one branch, touching the API surface, JPA layer, renderer, and Flyway schema.

## Severity as structured data

AgentConstraint had three fields: name, description, visibility. The issue noted that LLMs can infer severity from natural language — "never" reads as hard, "should" reads as soft. True, but that's the LLM doing work the descriptor should have done for it.

`ConstraintSeverity { HARD, SOFT }` goes on as the fourth field. The renderer now discriminates: MARKDOWN gets `[HARD]`/`[SOFT]` labels paralleling the goals `[PRIMARY]`/`[SECONDARY]` pattern. PROSE groups by severity. A2A_CARD carries the field for machine routing. Twenty-four construction sites across six files needed the new parameter — the blast radius of adding a field to a record that's used everywhere.

## Capabilities weren't checking what goals already checked

Goals and constraints validated name uniqueness in the descriptor's compact constructor. Capabilities didn't. Worse, `AgentDescriptorComparator.compareCapabilities()` used `Collectors.toMap()` which crashes on duplicate keys — so a descriptor with duplicate capability names registered fine but exploded at comparison time. The gap was exposed by the goals/constraints work: when I added uniqueness validation for those, the asymmetry became visible. `MAX_CAPABILITIES = 20` and a `UNIQUE (descriptor_id, name)` DDL constraint close it.

## Goal-based querying — and where it stops

The design spec for goals (#100) deferred five use cases. This session picked them up, and the first question was which belong in eidos and which belong in the engine.

The platform routing doc made it clear. The engine owns dispatch decisions — `AgentRoutingStrategy`, `WorkOrchestrator`, trust maturity classification. Eidos owns structured identity and data access. So goal-based routing (strategy decision) and goal-based termination (completion evaluation) are engine concerns. What eidos provides is the query primitive: `AgentQuery.goalName` with exact name match and an EXISTS subquery in JPA to avoid MultipleBagFetchException. No vocabulary subsumption — goals are identity-level standing objectives, not vocabulary-grounded terms.

`hasGoal()` and `hasConstraint()` on `AgentDescriptor` are one-liners that save downstream callers a stream pipeline. Small, but the kind of thing that gets reimplemented five times across five repos without them.

## Two Flyway files, one version number

The previous session's templates migration and the goals/constraints migration both used V7. Flyway doesn't forgive that — the runtime module's test suite crashed on startup. A rename to V8 fixed it; the design review caught a deeper issue: the spec proposed an incremental ALTER TABLE migration (V9) when the project's convention is base-migration rewrite since there are no deployed instances.
