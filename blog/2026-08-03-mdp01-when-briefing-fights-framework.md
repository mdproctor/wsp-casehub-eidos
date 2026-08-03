---
layout: post
title: "When the Briefing Fights the Framework"
date: 2026-08-03
type: phase-update
entry_type: note
subtype: diary
projects: [eidos]
tags: [personality, jungian, coherence, rendering, goal-evolution, llm-agents, multi-agent]
---

The structured personality composition paper ended with a clear finding: the Jungian cognitive function framework delivers personality signal to the prompt reliably — 100% MBTI alignment when briefing and framework agree. But when they disagree, the briefing wins. Peter Perfect's ENFJ profile expects Judging behaviour (structured, decisive), but his briefing text describes "gallant, volunteering, tunnel vision" — which reads as Perceiving (spontaneous, adaptive). The framework is doing its job. The briefing is undermining it, and nothing in the system notices.

This branch closes that gap with three features that move personality from passive context to active constraint.

## Catching Contradictions Before They Reach the LLM

The first problem is detection. A descriptor author writes a briefing paragraph and declares a cognitive function profile, and nothing checks whether the two agree. The briefing is free-form text; the profile is a weighted vector. They live in different representations with no shared type system.

`BriefingCoherenceValidator` bridges this. It runs two tiers of validation. The structural tier scans the briefing for vocabulary term values — the literal strings "bold", "conservative", "strict" that appear in registered vocabularies — and cross-references them against the descriptor's declared disposition axes. If the briefing mentions "bold" but the descriptor declares `riskAppetite=conservative`, that's a TENSION. It also checks orientation keywords: "outgoing" and "gregarious" in a briefing for a Ti-dominant (introverted) agent gets flagged.

The structural tier is vocabulary-grounded, not keyword heuristic. It only matches terms that actually exist in registered vocabularies, so it won't flag "bold" in "bold typeface" if the context vocabulary doesn't contain that term. The second tier — LLM-based semantic analysis — is designed but deferred. The structural tier catches the cases where the descriptor author accidentally used contradicting vocabulary. The semantic tier catches "gallant, volunteering → Perceiving." Both matter, but the first one is cheap enough to run at registration time.

The validator is wired into two places. `DescriptorCollector` runs it at registration and logs warnings — advisory, not blocking. `EidosSystemPromptRenderer` attaches the `CoherenceReport` to the rendered prompt as metadata, so consumers can inspect coherence without re-running validation. The eval harness will run the full two-tier validation once the LLM tier is implemented.

## From Description to Constraint

The second change is subtler but arguably more important for anyone building personality-aware agents. The cognitive function framework has been telling agents *what* they are — "your dominant function is Extraverted Thinking" — but not *how* that should constrain their behaviour. The prompt describes Te's analytical nature. It doesn't say "produce structured plans with explicit criteria" or "avoid unstructured brainstorming."

Each of the eight Jungian functions now carries two new methods: `responseStyleGuidance()` and `antiPatternWarning()`. Te gets "produce structured plans with explicit criteria and measurable outcomes"; Ne gets "explore multiple possibilities and connections, brainstorm divergent options." The anti-patterns are the inverse: Te avoids "unstructured brainstorming or open-ended exploration without clear criteria"; Ne avoids "premature convergence on a single solution."

These are vocabulary-level methods, not renderer hacks. The renderer consumes them through the `VocabularyTerm` interface (default empty strings; Jungian overrides with content). In MARKDOWN format, they appear as "Your Response Style" and "Avoid" sections after the compensation instruction. In A2A_CARD, they're `responseStyle` and `antiPattern` fields on the dominant function's JSON — machine-readable constraints that an orchestrator can use for structured generation or output validation.

The research paper's §9 called this "prompt engineering refinements within the existing architecture." That's exactly right. The vocabulary system produces the specification; these methods produce the behavioural constraint. No new abstractions, no new SPIs — just one more thing the vocabulary knows about itself.

## Goals That Learn

The third feature is independent of personality but follows the same signal-accumulation pattern. `GoalPriority` has been static — PRIMARY or SECONDARY, declared at registration, never changing. Over time, an agent's experience should inform which goals matter more.

`GoalSignalStore` accumulates SUCCESS and FAILURE outcomes per goal, following the same lifecycle pattern as `DispositionSignalStore` for personality signals: cumulative counts, explicit decay, no TTL. `GoalEvolution` evaluates whether priorities should shift — a SECONDARY goal with 15 successes and 1 failure (93% success rate, above the 80% threshold) gets promoted to PRIMARY. A PRIMARY goal drowning in failures gets demoted.

The interesting design question was the demotion guard. What happens when the only PRIMARY goal accumulates enough failures to warrant demotion? Blocking the demotion creates a stuck state. Allowing it creates an agent with no primary purpose. The answer is swap semantics: atomically promote the highest-performing SECONDARY goal before demoting the failing PRIMARY. The agent always has at least one PRIMARY goal. If no SECONDARY exists to promote, the system returns `Dampened` — the engine decays the accumulated signals instead of making a structural change. This mirrors how personality evolution handles shadow takeover: structural transitions are permitted, but never to an invalid state.

The full CDI ladder follows the established pattern — `NoOpGoalSignalStore` @DefaultBean for deployments without goal tracking, `InMemoryGoalSignalStore` @Alternative for tests, `JpaGoalSignalStore` @IfBuildProperty for persistent storage. Five configurable preferences control the promotion/demotion thresholds. The engine orchestrates when to call `evaluate()` — eidos provides the detection and decision, not the scheduling.

## What This Opens Up

The coherence validator is structural-only today. The LLM tier — sending the briefing to a ChatModel for semantic signal extraction — is designed and specified but not implemented. When it lands, the same validator catches the class of problems the structured personality composition experiment surfaced: contradictions that require understanding behaviour, not just matching vocabulary terms.

Function-specific constraints are the bridge between personality specification and personality control. The eval harness should show improved function activation scores — agents that are told *how* to behave in terms of their cognitive function, not just *what* their function is, should activate the correct function more consistently. That's the hypothesis the next eval run will test.

Goal evolution connects to the engine's runtime adaptation story. The SPI is the contract; the engine-side orchestration (engine#800 Sub-epic C) decides when to call `evaluate()` and how to handle the result. The pattern — detect in eidos, decide in eidos, orchestrate in engine — is the same separation of concerns as capability health and disposition health.
