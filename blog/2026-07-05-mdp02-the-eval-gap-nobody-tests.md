---
layout: post
title: "The Eval Gap Nobody Tests"
date: 2026-07-05
type: phase-update
entry_type: note
subtype: diary
projects: [eidos]
tags: [eval, capability-description, variant-pairs]
series: issue-95-eval-capability-description
---

*Part of a series on [#95 — eval: New eval profiles exercising capability description field](https://github.com/casehubio/eidos/issues/95). Previous: [The Field That Was Already Missing](2026-07-05-mdp01-the-field-that-was-already-missing.md).*

The `description` field on `AgentCapability` landed in #77. The rendering pipeline handles it in all three formats. The YAML loader deserialises it. The validator enforces the 500-char limit. Everything works — and nothing tests it.

All eight eval profiles have `null` on every capability description. The field exists in the type system, the renderer handles it, but no eval case ever exercises the path where a real description is present. MARKDOWN's " — " separator, PROSE's parenthetical rendering, A2A_CARD's declared-description fallback — all untested with actual data.

## Choosing the axis

The existing variant pairs cover RISK_APPETITE (sw-engineer) and RULE_FOLLOWING (security-analyst). Three axes remain: SOCIAL_ORIENTATION, AUTONOMY, CONFLICT_MODE. I chose AUTONOMY because it creates the cleanest behavioral contrast for pair testing — one agent who makes architectural decisions independently, one who escalates. The scenario questions write themselves: "A pipeline fails at 2am. You have a fix but it changes the schema contract. What do you do?" The autonomous agent acts; the directed agent escalates. CONFLICT_MODE was tempting (no existing profile even sets it) but the pair-contrast judge has never seen it, and introducing both a new field AND a new axis would confound what we're measuring.

## The delegation trap

The first draft had `delegation: true` on the autonomous variant and `delegation: false` on the directed one. It felt natural — autonomous agents delegate, directed agents don't. The design review caught it in round 1: `AgentProfileLoader.validateVariantPairs()` explicitly throws `IllegalStateException` when delegation differs between pair members. And protocol PP-20260605-1000ad says delegation is a platform capability ("can spawn sub-agents"), not a disposition trait. Correlating it with autonomy would confound the eval. Both profiles got `delegation: false`.

This is exactly the kind of mistake that passes a smell test but fails the structural one. The code would have thrown at test time, but the design review caught it before a single line was written.

## The fallback path we can't isolate

The A2A_CARD rendering path has a two-step precedence: LLM-enriched description wins; declared `cap.description()` is the fallback. In the eval harness, the ChatModel is always available, so enrichment always runs. The declared description never surfaces in the standard eval run. We added a synthetic A2A_CARD case with descriptions so `computeA2aMissingDescriptions()` runs against description-bearing capabilities — but the actual fallback path where enrichment is absent and declared descriptions provide the sole content remains untested. That's now #97.

The interesting thing about the data-engineer profiles is not the descriptions themselves — those are straightforward. It's what the profiles exercise beyond descriptions: AUTONOMY as a new variant pair axis, a new domain (data engineering) broadening the eval surface, and the first profiles where briefings must avoid confounding the primary axis. The directed variant's briefing deliberately avoids rule-following language because `ruleFollowing=adaptive` is locked on both sides. If the briefing said "follow runbooks," the pair-contrast judge might attribute the behavioral difference to rule-following rather than autonomy. The vocabulary gap system makes this explicit — `scope-bounded-execution` (FULL loss) captures the concept without leaking into an adjacent axis.
