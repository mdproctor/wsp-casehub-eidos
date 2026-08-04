---
layout: post
title: "The Probe Pipeline Gets Its Reference"
date: 2026-08-04
type: phase-update
entry_type: note
subtype: diary
projects: [eidos]
tags: [examples, capability-health, behavioral-signals]
---

I've been wanting to write the operational examples for eidos for a while — the features that differentiate it from "just a registry" had no consumer-facing demonstrations. Vocabulary registration and subsumption matching were covered, but the probe pipeline, learned exclusion, cost-aware routing? All buried in runtime unit tests. A consumer team looking at eidos would see the data model and the vocabulary, but not the operational machinery that makes it useful.

The epic had three children: learned specialization lifecycle (#78), full probe pipeline (#80), and cost-aware multi-agent routing (#81). Each got its own self-contained `@QuarkusTest` — no shared helpers, no base classes. Each test method uses a unique tenancy ID for isolation. A developer can open one file and see the complete story.

The interesting moment came during the design review. I'd written the spec for #78 with a narrative about subsumption-path exclusion — "agent gets excluded from security dispatch but continues serving code-review tasks." The robustness reviewer caught that this narrative was wrong. Learned exclusion is keyed on `(capabilityName, qualifier)` where the qualifier is the task domain string. Both `security-code-review` and `code-review` resolve to the same declared capability via subsumption. The differentiator is `ProbeContext.taskDomain`, not the capability query tag. When `taskDomain` is null, the probe pipeline skips the learned exclusion check entirely.

This is the garden gotcha (GE-20260523-fa7407) made concrete. The spec's original narrative would have produced a test that passed for the wrong reason — the `code-review` probe would have returned `Ready` not because the learned exclusion didn't apply, but because a null `taskDomain` skipped the check. We reframed the spec before writing any code: the test demonstrates task-domain-scoped filtering, not subsumption-path exclusion.

The full probe pipeline (#80) ended up with eight agents instead of the seven I'd originally planned. The design review caught that `BehavioralViolation` has two distinct modes — `PER_DIMENSION` (any single compliance dimension exceeds its threshold) and `AGGREGATE` (total violations across all dimensions exceed the aggregate threshold, but no single dimension exceeds on its own). The spec said "complete pipeline" but was missing a distinct code path. Adding the eighth agent was the right call.

For the cost-aware routing example (#81), we used YAML descriptors instead of programmatic registration — three agents with different `qualityHint`, `latencyHintP50Ms`, and `costHint` values. The consumer-side ranking logic is inline in the test methods: sort by quality, filter by latency budget, filter by cost tier, combined ranking. No utility class, no abstraction. A consumer team copies the pattern.

The probe pipeline now has its canonical reference. Each `CapabilityStatus` variant is demonstrated in one place. The `ProbeContext` gotcha is tested explicitly — correct usage alongside the common mistake. The cost-aware routing shows what structured identity enables that free-text description can't: filtering, ranking, and machine-readable A2A cards with routing signals.

The open question is whether these examples are sufficient for engine integration. Engine dispatches through `CapabilityHealth.probe()` at dispatch time — the examples show how to set up the descriptors and signals, but the engine's `GoalFailureRecorder` (engine#860) will need per-goal discrimination when recording signals. That's a different layer of complexity that these eidos-side examples don't reach.
