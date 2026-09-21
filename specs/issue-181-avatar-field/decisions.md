# Decisions — #181 Avatar Field

## D1: Avatar field type and discrimination

**Choice:** Single `String avatar` field on `AgentDescriptor`, discriminated by prefix
**Alternatives:**
- Structured record (AvatarRef with enum type + value) — type-safe but over-engineered for an opaque identifier
- Separate fields (avatarCode + avatarUrl) — splits what is conceptually one identity into two mutually exclusive fields
**Rationale:** Follows the open-String pattern (slot, archetype). The string is opaque to eidos — blocks-ui owns the semantics. URL detection via `http://` or `https://` prefix is trivial and unambiguous.
**Trade-offs:** No compile-time type safety on the avatar value. Acceptable because eidos never interprets the code beyond discrimination.
**Sources:** avatar-generator-contract.md Section 5, issue #181
**Exploration:** quick
**Status:** captured

## D2: Auto-derivation mapping

**Choice:** Sorted canonical list of `{Family}/{SubArchetype}` keys, archetype index encoded as base36 in format `mythic:P{base36(index)}`
**Alternatives:**
- Enum ordinal — shifts when terms are reordered or inserted mid-enum
- Hash-based — collision risk, less human-readable
**Rationale:** Sorted `{Family}/{SubArchetype}` is stable under enum reordering — the sort key is the archetype's identity strings, not its enum position. Base36 keeps codes compact (0-47 fits in 2 chars). Default collection is `mythic`.
**Trade-offs:** Adding a new archetype between existing ones shifts all later indices. Mitigated by the sorted-key approach: insertions only affect the alphabetically-later range, and the coding scheme is append-only on the blocks-ui side (part indices are stable).
**Sources:** User-provided avatar identity system spec, ArchetypeTerm enum
**Exploration:** quick
**Status:** captured

## D3: Rendering scope

**Choice:** A2A_CARD includes `avatar` as a top-level string field. MARKDOWN/PROSE do not render it.
**Alternatives:**
- Render in all formats — but avatar is a visual concern, not a behavioural instruction for LLMs
**Rationale:** The avatar code has no meaning to an LLM. It exists for UI consumers (agent cards, chat UIs, CLI). A2A_CARD is the machine-readable format consumed by frontends. MARKDOWN/PROSE are LLM prompts — adding `avatar: mythic:P1B` would be noise.
**Trade-offs:** An LLM-facing prompt can't tell the agent what it "looks like". Acceptable — agent self-image is not a current requirement.
**Sources:** Protocol PP-20260611-228599 (numeric routing signals in A2A_CARD only)
**Exploration:** quick
**Status:** captured

## D4: Derivation location

**Choice:** Extend `ArchetypeDeriver` in runtime — derive avatar alongside archetype in the same bootstrap post-processing step
**Alternatives:**
- Separate `AvatarDeriver` class — unnecessary indirection for a single-line derivation
- In DescriptorCollector (eidos-core) — can't reference ArchetypeTerm for index computation
**Rationale:** ArchetypeDeriver already runs at registration time and has vocab on the classpath. Avatar derivation depends on archetype being set, so it's a natural extension of the same step.
**Trade-offs:** ArchetypeDeriver gains a second responsibility. Acceptable given the derivation is a single method call.
**Sources:** ArchetypeDeriver.java (runtime/registrar)
**Exploration:** quick
**Status:** captured
