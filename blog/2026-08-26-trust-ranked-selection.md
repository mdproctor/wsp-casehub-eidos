---
title: "When the review says you're in the wrong repo"
date: 2026-08-26
author: mdp
entry_type: note
subtype: diary
series: issue-144-trust-ranked-selection
projects:
  - casehubio/eidos
tags: [agent-selection, trust, bridge-pattern, design-review, capability-ownership]
---

The issue looked straightforward. Qhorus needs to select agents by trust score from eidos discovery results. Eidos already has `AgentRegistry.find()` returning `List<AgentMatch>`. Add an SPI that takes those matches, ranks by trust, returns the best one. S/Low — one interface, one sealed result type, one `@DefaultBean`.

I got through seven design decisions — SPI naming, health scope, bootstrap handling, the full approach — before the decision review came back and said: you're building this in the wrong repo.

The review pointed at `capability-ownership.md`. It says "Agent routing / selection" belongs to `casehub-engine-api`. And engine already has `TrustWeightedAgentStrategy` — a full trust maturity model with BOOTSTRAP/QUALIFIED/BORDERLINE/EXCLUDED phase classification, policy-driven thresholds, quality floor checks, and escalation. The review was right. What I'd designed was a simpler, parallel mechanism that would produce different (and worse) selections for the same inputs.

The constraint that made it interesting: qhorus cannot depend on engine-api. So "just use engine's routing" isn't an option for qhorus. But creating a parallel routing mechanism in eidos violates the platform's capability ownership. The bridge pattern resolved both: eidos owns the SPI surface (where qhorus can reach it), engine owns the selection logic (where the maturity model lives), and a new `eidos-routing` module converts between them.

The three tiers that emerged: `AgentSelector` SPI in eidos-api — pure Java, no engine dependency. `SimpleAgentSelector` `@DefaultBean` in eidos-runtime — basic trust ranking with `Instance<TrustScoreSource>` (optional injection, health-only mode when no trust source is deployed). `EngineAwareAgentSelector` `@Alternative @Priority(1)` in eidos-routing — converts `AgentMatch` to engine's `AgentCandidate`, delegates to `AgentRoutingStrategy`, maps `RoutingResult` back to eidos's `AgentSelection`. When eidos-routing is on the classpath, CDI priority does the switch. Qhorus never knows which path is active.

Two things the review caught that would have been real bugs. First: the original design used a flat default score (0.5) for bootstrap agents, ignoring their global trust score. A trusted agent (globalScore 0.85) assigned a new capability would be demoted to 0.5 — treating earned reputation as if it didn't exist. The fix is a three-step fallback: capability score, then global score, then configurable default. Second: `@Priority` on CDI proxy classes. I wrote a `resolveStrategy()` that used `s.getClass().getAnnotation(Priority.class)` to find the highest-priority routing strategy. CDI proxies are generated subclasses — they don't carry the original class's annotations. Every strategy would resolve to priority 0, making selection arbitrary. The fix: `Instance.get()`, which uses Quarkus's build-time priority resolution.

The `AgentSelection` sealed type ended up with three variants instead of the original two. `Selected` and `NoneQualified` were in the issue. `Escalated` came from the review — engine's routing distinguishes "no agents qualified" from "agents exist but need human review" (borderline stalemate). Collapsing that into `NoneQualified` loses the signal qhorus needs to route to a human. `EscalationKind` mirrors engine's `EscalationReason` without creating a dependency from eidos-api to engine-api.

What this opens up: qhorus can now resolve `role:X` targets with trust-aware selection without depending on engine. Engine can eventually replace parts of its internal candidate selection with the same SPI if that makes sense. The bridge pattern — SPI surface in one repo, logic delegation to another — might apply to other cases where the platform's capability ownership conflicts with a consumer's dependency constraints.
