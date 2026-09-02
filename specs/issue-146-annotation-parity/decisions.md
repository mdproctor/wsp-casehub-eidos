# Design Decisions — annotation parity (#146)

## D1: Full parity — no fields excluded from annotation surface

**Choice:** Every field on `AgentDescriptor`, `AgentDisposition`, and `AgentCapability` that is expressible via Builder or YAML will also be expressible via annotations. No progressive-disclosure carve-outs.
**Alternatives:**
- Exclude "power fields" (epistemicDomains, axisVocabularies, templates) — leaves annotation surface incomplete, forces generators to special-case surface selection
**Rationale:** A generator should be able to target any of the three surfaces with equal fidelity. Humans writing complex agents will prefer Builder/YAML naturally — that's progressive disclosure at the surface level, not at the field level.
**Trade-offs:** Verbose annotations for complex agents. Acceptable — the builder/YAML paths exist for that case.
**Sources:** casehubio/eidos#146 issue body ("full parity means any descriptor expressible in YAML or the Builder DSL can also be expressed declaratively via annotations")
**Exploration:** quick
**Status:** captured

## D2: Capability metadata via new @AgentCapabilities container, not @Discoverable expansion

**Choice:** New `@AgentCapabilities(@AgentCapabilityDef[])` in `casehub-eidos-annotations`. `@Discoverable` stays name-only in `casehub-eidos-api`. Both present on the same class → build-time error.
**Alternatives:**
- Expand `@Discoverable` with nested annotation array — breaks existing consumers, bloats the API module with rich metadata types
- Inline all fields into `@Discoverable` — changes its character from pure marker to complex annotation
**Rationale:** Module layering (`@Discoverable` is Tier 1 API, rich metadata belongs in the annotations extension). Follows established container+def pattern (`@AgentGoals/@AgentGoalDef`, `@OrgRelationships/@OrgRelationshipDef`). Progressive disclosure preserved: simple agents use `@Discoverable`, rich agents use `@AgentCapabilities`.
**Trade-offs:** Two annotations for capabilities. Mitigated by build-time mutual-exclusion check — no ambiguity.
**Sources:** AgentCapability.java (11 fields), Discoverable.java (String[] only), OrgRelationships/OrgRelationshipDef pattern, issue-139 spec Known Limitations §2
**Exploration:** quick
**Status:** captured
