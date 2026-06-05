---
layout: post
title: "Two Vocabularies Are Better Than One"
date: 2026-06-05
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-eidos]
tags: [vocabulary, personality-frameworks, design]
---

## Two Vocabularies Are Better Than One

Before you can implement Belbin and DISC vocabulary modules, you need to know
which AgentDescriptor field they belong to. That sounds like a small question.
It isn't.

My first instinct was to treat DISC the same as Belbin: both give agents named
types, both are personality-adjacent, both would end up in `casehub-eidos-vocab`.
So DISC as a slot vocabulary — `slot="dominance"` the way you'd write
`slot="shaper"`. That's wrong, and the error matters architecturally.

Belbin roles are assigned. A team decides you're the Co-ordinator or the Shaper
— that's the role you fill in this team context. DISC types are measured. You
*are* a D-type wherever you go, regardless of the team. The slot field is for
"what role do you play here?" DISC answers a different question: "how do you
behave anywhere?" That's a disposition field, not a slot.

The correction unlocked something better. AgentDescriptor has a `slotVocabulary`
field for role assignment and a `dispositionVocabulary` field for behavioral style.
An agent can hold both simultaneously:

```
slotVocabulary        = "urn:casehub:vocab:belbin"
dispositionVocabulary = "urn:casehub:vocab:disc"
slot                  = "co-ordinator"
disposition.socialOrient  = "dominance"
```

A Co-ordinator who is also a D-type is genuinely interesting — the Belbin role
implies `facilitative` on socialOrient; the DISC type implies `independent`. They
diverge. That divergence is information: the role describes what the team expects
of this agent, the DISC type describes how the agent actually behaves. If they
match, the agent fits the role cleanly. If they don't, you know where to look
for friction.

Then the API gap showed up.

`VocabularyRegistry.equivalentValues(fromVocab, value, toVocab)` takes no axis
parameter. A Dominance type maps to `independent` on socialOrient, `flexible` on
ruleFollowing, `bold` on riskAppetite, `autonomous` on autonomy — four different
Conscientiousness terms for the same DISC type, depending which axis you're
resolving. The current signature can't express that. The Belbin half of eidos#26
can proceed; the DISC half is blocked until eidos#40 resolves the API extension.

A few other things surfaced during the eleven-framework mapping exercise. The
Completer Finisher maps to `autonomy: semi-autonomous`, not `directed` — which
was my initial call. The CF role is defined by internal perfectionism, not
instruction-following. They're driven by their own exacting quality judgment,
not by what they're told to do. `directed` was the wrong model. Similarly, the
KAI instrument scores 32–160 with *higher* scores indicating the Innovator pole.
Counter-intuitive if you expect "high score = more conventional," and the draft
had it backwards.

The document that came out of this (`docs/personality-frameworks.md`) is 828
lines covering eleven frameworks, a cross-reference table, ten compatibility pair
ratings, three combination patterns, seven anti-patterns, and vocabulary draft
tables for Belbin — tables structured so the #26 implementor can read a row and
write a `Map.entry(key, new VocabularyTerm(...))` call directly from it. The DISC
draft table is there too, annotated with eidos#40 warnings on every axis-resolution
column.

The document is the prerequisite. eidos#40 is the next unblock. Once the API
shape is decided, eidos#26 can start.
