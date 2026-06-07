---
layout: post
title: "Five Axes"
date: 2026-06-07
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-eidos]
tags: [vocabulary, belbin, disc, thomas-kilmann, disposition, eidos]
---

The vocabulary system that eidos#40 cleaned up needed something to run on. Three vocabularies and a new disposition axis — that was the scope of today.

The axis question came first because everything else depended on it. `AgentDisposition` had four fields: `socialOrient`, `ruleFollowing`, `riskAppetite`, `autonomy`. The personality framework mapping work (eidos#29) had already flagged the gap: Thomas-Kilmann conflict modes don't fit any of them. Competing and Collaborating aren't social preferences. They're conflict strategies — genuinely orthogonal, not approximations of something we already had.

So `conflictMode` became the fifth axis. `CONFLICT_MODE` in `DispositionAxis`. The value of the exhaustive switch pattern showed up immediately: adding the new enum constant broke `AgentDisposition.get()` at compile time, which is exactly what you want. Nothing silently missing coverage.

The vocabulary hierarchy that had been building for months finally started to look like a system. Belbin goes into `slot` — nine team roles answering "what do you contribute to this team?" DISC goes into disposition — four behavioral styles answering "how do you behave in any context?" Thomas-Kilmann goes into `conflictMode`. Three frameworks, three different axes, none of them interfering with each other.

DISC→Conscientiousness is the mapping that eidos#40 made possible. D-type agents (`dominance`) are `independent` on social orientation, `flexible` on rule following, `bold` on risk appetite, `autonomous` on autonomy. The DISC→TK mapping is simpler: Dominance and Competing occupy the same assertive/low-cooperation quadrant of TK's two-dimensional space. The geometry matches. `DiscTerm.axisExactMatch` implements both targets with an exhaustive switch — `ConscientiousnessTerm` for axes one through four, `ThomasKilmannTerm` for `conflictMode`. Any future axis addition breaks both switches at compile time.

The single `dispositionVocabulary` field had an architectural gap I hadn't thought through properly: if you want DISC for the four behavioral axes and TK for conflict mode, one URI can't express both. The fix was `axisVocabularies` — a `Map<DispositionAxis, String>` that overrides the default per axis. `vocabUriForAxis(axis)` does the three-step resolution: per-axis override first, then `dispositionVocabulary`, then `domainVocabulary`. Callers don't implement the precedence chain — they call the method.

Thomas-Kilmann presented a classification question that took longer to resolve than the implementation did. The existing `personality-frameworks.md` classified TK modes as situational — conflict strategy, not stable trait. That classification blocks a `conflictMode` axis: you can't have a disposition field for something you've defined as situational. The resolution is that the distinction between stable and situational depends on what you're modelling. An agent's *default* conflict approach is as stable as its social orientation. Both vary in practice. Both can be encoded as a prior. The field captures the prior, not the behaviour in any specific interaction.

One gotcha worth noting: `AgentDescriptor.axisVocabularies` stores `Map<DispositionAxis, String>` as JSON in a TEXT column, serialized by Jackson with enum names as keys. That means renaming a `DispositionAxis` constant is a database migration, not just a code change. Old rows contain `"CONFLICT_MODE"` — Jackson's `valueOf()` won't recognize a renamed constant and will throw on load. The operations guide now documents this.

The Belbin vocabulary was mechanical. Nine roles, single-letter aliases, no cross-vocabulary mappings — Belbin Associates haven't published canonical URIs. The DISC vocabulary was where the design mattered. Four constants, each overriding `axisExactMatch` via the anonymous subclass pattern, covering all five axes for two target vocabularies. The `CONSCIENTIOUSNESS_DISC` constant's label is "Analytical (DISC-C)" rather than "Conscientiousness" — rendered output needs to distinguish it from the `urn:casehub:vocab:conscientiousness` vocabulary, which is a different thing sharing a name.

Three issues closed, an ADR written, a new protocol for `vocabUriForAxis` usage. The vocabulary system has five axes now. The next thing that wants to express conflict disposition has somewhere to put it.
