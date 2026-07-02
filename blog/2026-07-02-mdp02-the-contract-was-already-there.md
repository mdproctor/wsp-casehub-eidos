---
layout: post
title: "The Contract Was Already There"
date: 2026-07-02
type: phase-update
entry_type: note
subtype: diary
projects: [eidos]
tags: [behavioral-contracts, probe, compliance]
series: issue-85-behavioral-contracts
---

*Part of a series on [#85 — Behavioral contracts and runtime validation](https://github.com/casehubio/eidos/issues/85). Previous: [The Tag That Didn't Match](2026-07-02-mdp01-the-tag-that-didnt-match.md).*

I started this branch thinking I needed to design a behavioral contract system — some new type that would sit alongside the descriptor and define what an agent should do. The brainstorming went deep: what does "behavioral contract" mean for an LLM agent? Where does it live? Who sets the expectations?

The answer turned out to be embarrassingly simple. The descriptor already declares behavioral promises. `latencyHintP50Ms = 5000` is a latency contract. `delegation = true` is a delegation contract. `excludedDomains = {"rust"}` is a negative constraint. `ruleFollowing = strict` is a behavioral expectation. Every field that describes what an agent is or can do implicitly promises how it will behave.

What was missing wasn't the contract — it was the feedback loop. Nobody checked whether the promises were kept.

The platform had exactly two narrow pathways from behavior to routing: human attestation (subjective, manual, expensive) and DECLINE signals (agent refuses a task). Everything else — latency drift, quality degradation, failure to delegate, failure to escalate — was invisible to the routing system.

The fix fell out of the existing architecture. `CapabilitySpecializationStore` already implemented the right pattern: accumulate signals over time, query at probe time, influence routing. It was a behavioral signal counter that happened to be named after its first use case. Renaming it to `BehavioralSignalStore` and adding COMPLIANT/VIOLATED alongside DECLINE/SUCCESS was the natural generalization.

The `domain` parameter became `qualifier` — an honest name for a field that was already doing double duty. For specialization signals it qualifies by task domain ("rust", "java"). For compliance signals it qualifies by expectation dimension ("latency", "attestation-rate"). Same shape, different semantics.

Probe got a sixth step. After checking degradation, availability, declared exclusions, learned exclusions, and epistemic weakness — all the existing checks — it now queries VIOLATED signal counts across all dimensions. If any dimension exceeds the threshold, the agent gets `BehavioralViolation` status. The engine decides what to do with that signal; eidos just surfaces it.

The attestation bridge closes the second loop. Compliance violations can optionally generate `LedgerAttestation` records — the platform itself as attestor, with `confidence = 1.0` because these are measured, not judged. The trust scoring pipeline processes them like any other attestation, and the routing system deprioritizes agents with declining compliance scores. Two independent feedback paths from a single observation: the fast path through the signal store for immediate routing exclusion, and the trust path through attestations for gradual deprioritization.

The design review caught the thing I would have missed. The `domain` parameter overload was defensible — it works — but it's not honest. A parameter that means "task domain" for some callers and "compliance dimension" for others is carrying two concepts in one name. `qualifier` doesn't pretend to be specific to either use case. The review also caught that `BehavioralViolation` should carry a `Map<String, Integer>` (all exceeding dimensions) rather than a single worst dimension — because Step 6 queries all dimensions by design, and collapsing that to one dimension throws away information the engine needs.

The pattern here is worth naming: the store was always more general than its name. `CapabilitySpecializationStore` was a behavioral signal counter from day one — it counted events by (agent, capability, qualifier, signal type) with TTL-based expiry. The name just reflected the first use case instead of the actual abstraction. The rename costs nothing (no deployed consumers), and it makes the next extension obvious rather than surprising.
