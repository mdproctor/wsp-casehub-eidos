---
layout: post
title: "Terminology that swaps itself"
date: 2026-09-07
entry_type: note
subtype: diary
projects: [casehubio/eidos]
tags: [vocabulary, display, cross-vocab, aliasing]
---

Devtown calls them planners and reviewers. Gastown calls them witnesses and polecats. The agents are doing the same work — the terminology is a domain choice, not a technical one. We wanted a way to set up a base vocabulary, set up a second one with mappings, and have the display labels swap automatically.

The vocabulary system already had the hard infrastructure. `VocabularyTerm.exactMatch()` maps between vocabularies. `axisExactMatch()` handles disposition terms where the mapping is axis-specific (DISC and Thomas-Kilmann only connect through specific disposition axes, not generically). `VocabularyRegistry.equivalentValues()` walks the cross-vocab chain. All of it was built for capability matching and subsumption — but it works just as well for display label resolution.

What was missing was a one-call service that encapsulates the resolution chain: look up the value in a source vocabulary, translate to a target vocabulary via the cross-vocab mapping, resolve the target term, return its label. Fall back gracefully at every step — unknown vocab returns raw value, no cross-vocab match returns the source label, null source vocab triggers auto-discovery across all registered vocabularies.

The API ended up with four parameters: value, source vocab URI, target vocab URI, and disposition axis. Three convenience overloads cover the common cases. The cross-vocab resolution is two-step because `equivalentValues()` returns a target *value* (String), not the term — a second `resolve()` call retrieves the term for its `label()`. The decision review caught this and also surfaced the axis-aware gap: disposition terms use `axisExactMatch()` exclusively, so without the axis parameter the swap would silently fail for personality profiles.

The whole thing delegates to `VocabularyRegistry` — sixty lines of implementation, twelve tests, and the org diagram can now render any agent in any domain's terminology by passing a target vocabulary URI. The blocks-ui issue has the reference SVG showing what the full diagram should look like; `DisplayTermResolver` is what makes the labels inside those agent cards swappable.

Temporary home in eidos-api. Moves to platform-api when the platform team has bandwidth — the refactor is a single IntelliJ `ide_move_file`.
