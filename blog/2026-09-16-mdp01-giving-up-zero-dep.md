---
title: "Giving Up Zero-Dep for Model Selection"
date: 2026-09-16
author: mdp
entry_type: note
subtype: diary
tags: [model-selection, platform-api, architectural-boundary, vocabulary]
projects: [casehubio/eidos]
series: issue-179-model-selection-schema
---

Issue #172 landed model tier and model capabilities on `AgentCapability` as flat
strings — `modelTier` and `modelCapabilities`. That was the right call at the
time. Eidos declared requirements as plain values, platform resolved them, and
the API module stayed zero-dependency. Clean boundary, no coupling.

Then platform#335 landed a proper agent configuration manifest with
`model-selection.schema.json` — a union type where model selection is either a
string shorthand (`model: reasoning-heavy`) or an inline constraints object
(`model: {tier: FAST, capabilities: [reasoning], max-cost: LOW}`). Nine fields
on `ModelQuery` instead of two flat strings. And platform#342 added direct
`ModelQuery` dispatch on `AgentSessionConfig` — no string serialisation
round-trip needed.

The question became: does eidos create its own `ModelRequirements` type with
just the two fields #172 approved, or does it use `ModelQuery` directly from
`casehub-platform-api`?

I chose `ModelQuery` directly. This breaks the zero-dep quality goal that has
been in ARC42STORIES since the project started. The decision review flagged
this hard — three HIGH-priority findings about reversing #172's boundary
architecture, leaking operational fields into agent identity, and violating
the first quality goal in the document. All valid concerns.

The reasoning for overriding them: `ModelQuery` is the canonical type. Creating
a thin eidos-owned wrapper carries the same two fields but needs a conversion
layer, and the agent author *should* be able to express richer constraints
than just tier and capabilities. If an agent needs a model with 128k context
and text+vision, that's identity — it defines what the agent can do. Restricting
the descriptor to tier-only means the engine has to infer the rest from context,
which is less precise.

The implementation is a dual-field pattern — `String modelRef` for string
shorthands (aliases, tier refs, model IDs) and `ModelQuery model` for inline
constraints. Mutually exclusive. This mirrors `AgentSessionConfig` on the
platform side, which already has both `String model` and `ModelQuery modelQuery`.
The YAML deserializer dispatches on node type: text node goes to `modelRef`,
object node gets parsed field by field into `ModelQuery`.

One thing the decision review surfaced that I'm still thinking about:
`ModelQuery.authMethod` is now structurally present on agent descriptors even
though it's an infrastructure concern. Prevention by absence (the #172 approach)
is stronger than prevention by validation. For now, eidos ignores it — the
recorder never sets it, the validator doesn't check it, the renderer doesn't
render it. But the type signature invites it. If this becomes a problem, the
fix is a `ModelQuery.forDescriptor()` factory on the platform side that strips
infrastructure fields. Not needed yet.

The doc audit afterwards was revealing — ARC42STORIES had fifteen stale reactive
references from before parent#384 retired the reactive tier. The contributor
guide was missing seven modules (eidos-core, annotations, routing, all four org
modules). These accumulated silently because doc updates don't have a forcing
function the way code changes do. The #179 change forced a pass through all
three documents, and the drift was worse than expected.
