---
layout: post
title: "Vocabulary-Aware Compliance — Delegation and Escalation Observation Semantics"
date: 2026-07-04
type: phase-update
entry_type: note
subtype: diary
projects: [eidos]
tags: [behavioral-contracts, vocabulary, compliance]
---

Delegation and escalation were the compliance dimensions I'd been avoiding.

Not because they're hard to implement — the implementation turned out to be small. Because they're hard to *define*. The #85 behavioral contracts spec explicitly deferred both: delegation needs "observation window semantics" and escalation needs "vocabulary-aware interpretation." Both sounded like they'd require new infrastructure.

They didn't.

The observation window question dissolves once you realise the existing TTL + threshold model already provides windowing. VIOLATED signals expire after 90 days by default. The probe threshold is 3 per-dimension. So "3 delegation failures within 90 days → BehavioralViolation" — that's a window. No new concept needed.

The real question for delegation was never *how to window observations* but *what constitutes a single observation*. And that's the engine's call, not eidos's. The engine sees task characteristics — complexity, decomposability, whether delegation was warranted. Eidos says "this agent expects to delegate." The engine decides when to check.

Escalation was the interesting one. `AgentDisposition.autonomy()` is an open String — "directed", "semi-autonomous", "autonomous" from ConscientiousnessTerm, or completely different values from domain vocabularies. `BehavioralExpectations.escalationExpected()` needs to interpret that string to determine "should this agent escalate when uncertain?" But eidos-api can't depend on casehub-eidos-vocab. The vocabulary terms are in a different module.

The solution: add `impliesSupervision()` as a default method on `VocabularyTerm` itself. Returns `false` by default. Vocabulary authors override it for terms that indicate supervised behavior. ConscientiousnessTerm.DIRECTED returns `true`. AUTONOMOUS inherits `false`. DISC terms get their own overrides — STEADINESS (maps to DIRECTED) returns `true`, DOMINANCE (maps to AUTONOMOUS) returns `false`.

`escalationExpected()` resolves the autonomy string through VocabularyRegistry, gets the term, calls `impliesSupervision()`. If the vocabulary isn't registered or the value doesn't resolve — `false`. Fail open. An agent without vocabulary-grounded disposition shouldn't be penalised for not escalating.

The design review caught something I'd missed: DISC terms need their own `impliesSupervision()` overrides, not reliance on cross-vocabulary equivalence. The `axisExactMatch` bridge declares term *equivalence*, not behavioral property *inheritance*. STEADINESS being equivalent to DIRECTED doesn't mean it automatically inherits DIRECTED's `impliesSupervision()` return value. Each vocabulary independently declares the behavioral properties of its own terms. A cross-vocabulary consistency test validates they don't drift.

The review also pushed for a convenience overload — `escalationExpected(AgentDescriptor, VocabularyRegistry)` that internalises the `vocabUriForAxis(AUTONOMY)` call. The 3-param version stays for callers with pre-resolved URIs. Both overloads exist; the engine picks whichever fits its observation path.

Five commits, 384 insertions across api, vocab, and runtime. The probe pipeline didn't change — Step 6 already counts all VIOLATED signals regardless of dimension key. Adding `ComplianceDimension.DELEGATION` and `ComplianceDimension.ESCALATION` as string constants was enough to make the probe handle both dimensions.

The engine-side observation logic — deciding what constitutes a delegation opportunity or an escalation trigger — is tracked in engine#645. That's where the policy decisions live: task complexity for delegation eligibility, uncertainty detection for escalation triggers. Eidos defined the contract; the engine implements the observation.
