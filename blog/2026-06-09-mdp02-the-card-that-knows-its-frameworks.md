---
layout: post
title: "The card that knows its frameworks"
date: 2026-06-09
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-eidos]
tags: [a2a, vocabulary, frameworks, machine-to-machine, slot, disposition]
---

The A2A card has always been thin. Name, agentId, version, capabilities — enough
to know what an agent can do, nothing about what kind of thing it is. An orchestrator
querying for a Belbin Monitor Evaluator had to pull the full `AgentDescriptor` from the
registry; the card alone told it nothing.

`eidos#45` was framed as a question: should the A2A card expose slot, disposition, and
framework references? The issue listed them as candidates, not decisions. I spent more
time on the design than on the implementation.

The decision that needed the most thought was the `frameworks` array. Slot and disposition
are straightforward — structured fields, vocabulary-resolved, machine-parseable. But
`frameworks` looks redundant at first: if you can already see `slot.vocabularyUri` and
`disposition.socialOrient.vocabularyUri`, why repeat the same URIs in a top-level array?

The answer is that the card has to serve two different query patterns. Navigation queries
("what is this agent's conflict mode, and which vocabulary grounds it?") are served by
the inline fields. Discovery queries ("does this agent use Belbin at all?") require
scanning every field to find out — O(N axes) — unless there's an index. The frameworks
array is that index. It's not redundancy; it's a different access pattern served by the
same underlying data. Once I framed it that way, the design question answered itself.

The implementation surfaced a pre-existing asymmetry I hadn't noticed. `vocabUriForAxis()`
has a three-step fallback: `axisVocabularies.get(axis)` → `dispositionVocabulary` →
`domainVocabulary`. The slot lookup in `buildDescriptorPayload()` was a bare `if
(descriptor.slotVocabulary() != null)` — no fallback at all. The CLAUDE.md said
`domainVocabulary` is the default for all fields, but the slot implementation didn't
implement that. We added `vocabUriForSlot()` with a two-step chain (`slotVocabulary` →
`domainVocabulary`), applied it to both the LLM payload and the A2A card, and updated
CLAUDE.md to reflect what the method actually does and why `dispositionVocabulary` is
explicitly excluded from the fallback chain.

The spec went through two review rounds before implementation. The first round caught
something I'd misstated: the frameworks invariant read "contains exactly the URIs that
appear as `vocabularyUri` fields elsewhere in this card — no more, no less." But an
unregistered URI *does* appear as `vocabularyUri` in an axis object while being absent
from frameworks. The invariant was internally contradictory. The precise statement is a
conjunction: reachable by resolution AND registered with a non-blank name. "No more, no
less" against a precisely-defined set — that's fine. "No more, no less" against a set
that includes unregistered URIs — that's wrong.

The second review caught canDelegate ordering (listed first in the implementation
description, last in the schema example and existing code), a misleading variable name
(`ignored` where `value` was clearer), and the A2ASemanticEnrichmentStep input change
— not "untouched" when its input is richer.

Code review after implementation caught the third rendering path. I'd applied
`vocabUriForSlot()` to `buildDescriptorPayload()` and `assembleA2aCard()` — the two
paths I was actively looking at. But `assembleMarkdownStructural()` is a third slot
renderer and still had the old `slotVocabulary != null` guard. An agent with only
`domainVocabulary` set got correct slot vocabulary context in the LLM payload and A2A
card, but raw slot string in structural markdown. No test covered that combination.
The fix was adding `vocabUriForSlot().ifPresentOrElse()` to the structural renderer and
a test that drove the domainVocabulary path specifically.

The pattern is general enough that it went into the garden: when fixing a shared
resolution method, search for all callers of the old access pattern, not just the ones
you're looking at.

The card now tells an orchestrator which Belbin role this agent plays, what vocabulary
grounds each disposition axis, and which theoretical frameworks are instantiated — all
from the card itself, without a registry query. Whether that actually enables
Belbin-based composition in the engine (#28) depends on that work proceeding. But the
eidos side is ready.
