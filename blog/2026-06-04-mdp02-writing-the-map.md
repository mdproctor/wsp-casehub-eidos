---
layout: post
title: "Writing the Map"
date: 2026-06-04
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-eidos]
tags: [arc42stories, documentation, architecture]
---

## Writing the Map

The thing about documentation for a project like eidos is that the knowledge doesn't live in one place. The architecture says module A depends on module B, but to understand *why* the cache SPI was unified, you need a session diary from May 30th about a deadlock that only made sense after two previous failed design attempts. The git log tells you what landed but not why the rendering pipeline went through three redesigns before settling on its current shape.

The ARC42STORIES.MD stub committed to the workspace a few weeks ago had §1 and §2 done and twelve 🔲 markers for everything else. I wanted to close all of them in one session. The source material was 15 blog entries from May 23 through June 4, eight design specs, two ADRs, the full git log, the Arc42Stories spec itself, the CaseHub Foundation tier profile, and every Java file in the `api/` and `runtime/` packages. Claude read all of it in parallel before we wrote a word.

Two decisions shaped the document's structure.

**The layer taxonomy.** The Foundation tier profile says "foundation modules define their own layer taxonomy — no prescribed taxonomy." Devtown has a natural stack: domain baseline → work → qhorus → ledger → engine → trust routing. Eidos doesn't have that shape — it's a set of increasingly capable layers building on each other. I settled on seven: Zero-Dep API (L1), CDI Discovery (L2), JPA Registry (L3), Capability Health (L4), Rendering Pipeline (L5), Eval Framework (L6), Knowledge Graph (L7). The distinction between delivery order (chapters) and reading order (layers) was the key structural insight. You deliver C1 before C6, but if you're trying to understand eidos, you read L1 through L7 in order because each layer builds on what came before.

**The anti-patterns.** Arc42Stories requires anti-patterns inline in §8 — not a reference to the harness guide, actual Symptom → Cause → Fix content. Eidos has four that genuinely bit the project during development: the `ProbeContext.taskDomain` / `capabilityTag` confusion (EpistemicallyWeak never fires; the garden entry from May 23 has the full story), the `emitOn` vs `runSubscriptionOn` mistake (event loop blocking; covered in the "hold no threads" entry from May 29), the CDI no-arg constructor loss (startup failure, no compile-time signal; this one appeared twice), and the NaN IEEE 754 confidence guard (`Double.NaN < 0.0` is false; caught by code review on the last branch). All four are real. None of them are the kind of thing you'd infer from reading the codebase.

The "Pattern to replicate" sections in each layer entry were the most demanding to write. Arc42Stories wants domain-agnostic numbered steps an LLM in a completely different domain could follow. That means no "implement AgentDescriptor" — it has to say "create a pure-Java record for your domain entity, add a compact constructor that calls a static validator." Writing these forced me to name what each layer's core pattern actually is, rather than describing what eidos specifically does. It turns out that's harder than it sounds — and probably more useful.

The CaseHub profile listed the Foundation tier reference implementation as *(deferred)* since the profile was first written. As of this branch, that changes.
