# Decisions — extensionData field (#177)

## D1: Field type

**Choice:** `Map<String, Object>` — full Jackson-compatible tree (nested maps, lists, primitives)
**Alternatives:**
- `Map<String, String>` — simpler but forces double-serialization of nested config
- `JsonNode` — maximum fidelity but leaks Jackson into the api module (currently pure Java)
**Rationale:** wacky-manor's SocialConfig has nested structures. Applications should embed natural config shapes without encoding gymnastics.
**Trade-offs:** Shallow `Map.copyOf()` in the compact constructor doesn't guarantee deep immutability of nested values. Accepted — deserialized config trees are effectively immutable in practice.
**Sources:** AgentDescriptor.java (existing Map<DispositionAxis, String> pattern), casehubio/examples#56
**Exploration:** quick
**Status:** captured

## D2: Rendering

**Choice:** Never rendered — extensionData is excluded from all prompt formats (MARKDOWN, PROSE, A2A_CARD)
**Alternatives:**
- A2A_CARD only — machine-readable cards might benefit, but the data is application-specific, not agent identity
- All formats — opaque maps don't produce meaningful prompt text
**Rationale:** extensionData is application-level configuration, not agent identity for LLM consumption. Applications read it directly from the descriptor object. This also means it doesn't participate in the A2A structural hash (protocol PP-20260613-608684 is not implicated).
**Trade-offs:** If a future use case needs extensionData in A2A cards for cross-agent routing, a separate field or explicit rendering opt-in would be needed.
**Sources:** EidosSystemPromptRenderer.java, protocol PP-20260613-608684 (a2a-structural-assembly-hash-coverage)
**Exploration:** quick
**Status:** captured

## D3: Validation — size limit

**Choice:** Cap serialized JSON size at 64KB (MAX_EXTENSION_DATA = 65536 bytes)
**Alternatives:**
- No limit — TEXT column has no inherent cap, but no guardrail against huge payloads
- Key count limit only — weaker than a byte limit, doesn't prevent large values
**Rationale:** Follows the existing pattern where briefing has MAX_BRIEFING. 64KB is generous for config data while preventing abuse. Validated in the compact constructor by serializing to JSON and checking byte length — but since we don't have Jackson in api, we validate key count (max 100 top-level keys) as a proxy, and the JPA layer enforces the byte limit.
**Trade-offs:** Validation split across two layers (api validates key count, JPA validates byte size). Acceptable — the api validation catches most problems early.
**Sources:** AgentDescriptorValidator.java (MAX_BRIEFING pattern)
**Exploration:** quick
**Status:** captured

## D4: Annotation and YAML support

**Choice:** Both paths supported — annotations for flat String key-value pairs, YAML for full nested `Map<String, Object>`
**Alternatives:**
- YAML only — applications using annotations would need to set extensionData programmatically via Builder
**Rationale:** Consistency with how capabilities, goals, and constraints support both annotation and YAML declaration paths. Annotation syntax: repeatable `@ExtensionEntry(key, value)` nested in `@Identity` or a new `@ExtensionData` container annotation.
**Trade-offs:** Annotation path is limited to flat String-String pairs. Complex nested structures must use YAML. This is a natural limitation of Java annotation syntax, not a design gap.
**Sources:** EidosAnnotationsProcessor.java, @Identity annotation, @AgentCapabilityDef (repeatable annotation pattern)
**Exploration:** quick
**Status:** captured

## D5: Namespacing

**Choice:** Convention only — document reverse-domain key convention (e.g. `io.casehub.manor.socialConfig`) but don't enforce structurally
**Alternatives:**
- Enforce structurally — require dots/URI pattern in keys, validated in compact constructor
- No namespacing — flat keys with no convention
**Rationale:** Follows how Java system properties and OSGi bundle headers handle namespacing. Applications own their descriptors, so collisions are unlikely in practice. Structural enforcement adds friction for simple use cases.
**Trade-offs:** No compile-time or runtime protection against key collisions. Acceptable — documented convention is sufficient for the expected usage pattern.
**Sources:** Java system properties convention, OSGi bundle headers
**Exploration:** quick
**Status:** captured

## D6: Immutability semantics

**Choice:** Shallow `Map.copyOf()` in the compact constructor
**Alternatives:**
- Deep copy via Jackson round-trip — guarantees full immutability but adds Jackson dependency to api module and overhead
- `Collections.unmodifiableMap()` only — doesn't detach from caller's map
**Rationale:** Consistent with how `axisVocabularies` (`Map<DispositionAxis, String>`) is handled. Values from YAML/JSON deserialization are effectively immutable. Nested mutation is theoretically possible but practically a non-issue.
**Trade-offs:** No deep-immutability guarantee. Callers who mutate nested values get what they deserve.
**Sources:** AgentDescriptor.java compact constructor (axisVocabularies pattern)
**Exploration:** quick
**Status:** captured
