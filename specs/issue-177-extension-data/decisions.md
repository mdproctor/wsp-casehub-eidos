# Decisions — extensionData field (#177)

## D1: Field type

**Choice:** `Map<String, Object>` — full Jackson-compatible tree (nested maps, lists, primitives)
**Alternatives:**
- `Map<String, String>` — simpler but forces double-serialization of nested config
- `JsonNode` — maximum fidelity but leaks Jackson into the api module (currently pure Java)
- Raw JSON `String` — keeps api fully typed but pushes deserialization to every consumer
**Rationale:** wacky-manor's SocialConfig has nested structures. Applications should embed natural config shapes without encoding gymnastics. The "untyped hole" concern (R1-02) is intentional — extension data IS untyped by design. Applications know their own config shape and cast accordingly. This is the first deliberately opaque field on the descriptor; all others are structured identity data.
**Trade-offs:** Introduces the first `Object`-typed values into the descriptor's type system. Accepted — the alternative (raw JSON String or JsonNode) just moves the type unsafety to a different layer.
**Sources:** AgentDescriptor.java (existing Map<DispositionAxis, String> pattern), casehubio/examples#56
**Exploration:** quick
**Status:** captured

## D2: Rendering

**Choice:** Never rendered — extensionData is excluded from all prompt formats (MARKDOWN, PROSE, A2A_CARD)
**Alternatives:**
- A2A_CARD only — machine-readable cards might benefit, but the data is application-specific, not agent identity
- All formats — opaque maps don't produce meaningful prompt text
**Rationale:** extensionData is application-level configuration, not agent identity for LLM consumption. Applications read it directly from the descriptor object.
**Protocol guard (R1-06):** extensionData does not participate in the A2A structural hash (protocol PP-20260613-608684 is not implicated). Any future change that renders extensionData in any format MUST also add it to `buildDescriptorPayload()` per protocol PP-20260613-608684. Additionally, extensionData changes do NOT invalidate the prompt cache for any format — descriptors differing only in extensionData produce identical rendered prompts.
**Trade-offs:** If a future use case needs extensionData in A2A cards for cross-agent routing, a separate field or explicit rendering opt-in would be needed.
**Sources:** EidosSystemPromptRenderer.java, protocol PP-20260613-608684 (a2a-structural-assembly-hash-coverage)
**Exploration:** quick
**Status:** captured

## D3: Validation — size limit

**Choice:** Construction-time validation via recursive estimated size calculation. `MAX_EXTENSION_DATA_SIZE = 65536` characters (estimated). A zero-dependency utility method walks the map tree, summing `String.length()` for all string keys and values plus structural overhead estimates per entry. Validated entirely in the compact constructor — no two-layer split.
**Alternatives:**
- Key count limit proxy (max 100 top-level keys) — weak correlation with actual size, reviewed R1-03
- No limit — TEXT column has no inherent cap, but no guardrail against huge payloads
- Byte-exact via Jackson serialization — requires Jackson in api module
**Rationale:** Revised per R1-03. The original D3 split validation across api (key count) and JPA (byte size), violating the "no invalid record" invariant — every other field is fully validated at construction time. Recursive character counting is a pure-Java operation handling Map, List, String, Number, Boolean — the types YAML/JSON deserialization produces. The estimate is conservative (slightly over-counts) so some valid maps near the boundary may be rejected, which is acceptable.
**Trade-offs:** Estimated size is not byte-exact (String.length() ≠ UTF-8 byte count for non-ASCII, structural overhead is approximate). Acceptable — the purpose is to prevent abuse, not enforce an exact wire format limit.
**Sources:** AgentDescriptorValidator.java (MAX_BRIEFING pattern), R1-03 review finding
**Exploration:** quick
**Status:** revised

## D4: Annotation and YAML support

**Choice:** Both paths supported. YAML: full nested `Map<String, Object>`. Annotations: repeatable `@ExtensionEntry(key, value)` on a new `@ExtensionData` container annotation (separate from `@Identity`).
**Alternatives:**
- YAML only — applications using annotations would need to set extensionData programmatically via Builder
- Nesting `@ExtensionEntry` inside `@Identity` — makes extension data a first-class identity concern
**Rationale:** Consistency with how capabilities, goals, and constraints support both annotation and YAML declaration paths. `@ExtensionData` as a separate container (not nested in `@Identity`) positions extension data as secondary metadata, which matches its nature — it's application config, not core identity (R1-05). The annotation path handles flat key-value config; complex nested structures use YAML.
**Trade-offs:** Annotation path is limited to flat String-String pairs. Complex nested structures must use YAML. This is a natural limitation of Java annotation syntax. Applications starting with annotations that need nested structures later must migrate to YAML — but this is a progressive disclosure pattern, not a trap.
**Sources:** EidosAnnotationsProcessor.java, @AgentCapabilityDef (repeatable annotation pattern), R1-05 review finding
**Exploration:** quick
**Status:** revised

## D5: Namespacing

**Choice:** Convention only — document reverse-domain key convention (e.g. `io.casehub.manor.socialConfig`) but don't enforce structurally
**Alternatives:**
- Enforce structurally — require dots/URI pattern in keys, validated in compact constructor
- No namespacing — flat keys with no convention
**Rationale:** Applications own their descriptors, so collisions are unlikely in practice. Structural enforcement adds friction for simple use cases.
**Assumption (R1-07):** This decision is load-bearing on the single-application-per-descriptor invariant. If descriptor composition across application boundaries is introduced, structural namespacing enforcement must be revisited. Adding enforcement retroactively requires migrating all existing keys.
**Trade-offs:** No compile-time or runtime protection against key collisions. Acceptable for the current usage pattern.
**Sources:** Java system properties convention, R1-07 review finding
**Exploration:** quick
**Status:** revised

## D6: Immutability semantics

**Choice:** Recursive deep copy via a zero-dependency utility method in the api module. The utility walks the map tree, copying all nested `Map` and `List` instances. Only handles the types YAML/JSON deserialization produces: `Map`, `List`, `String`, `Number`, `Boolean`, `null`. Unknown types throw `IllegalArgumentException`.
**Alternatives:**
- Shallow `Map.copyOf()` — leaves nested mutable values (LinkedHashMap, ArrayList), reviewed R1-04
- Deep copy via Jackson round-trip — adds Jackson dependency to api module
- `Collections.unmodifiableMap()` only — doesn't detach from caller's map
**Rationale:** Revised per R1-04. The `axisVocabularies` analogy was false — that map has immutable `String` values. `Map<String, Object>` from YAML deserialization contains mutable `ArrayList` and `LinkedHashMap` instances. Every other field on AgentDescriptor guarantees immutability; extension data must too. The utility is ~20 lines and zero-dependency.
**Trade-offs:** Unknown value types (not Map/List/String/Number/Boolean/null) cause `IllegalArgumentException` at construction time. This is intentional — it prevents arbitrary mutable objects from entering the record.
**Sources:** AgentDescriptor.java compact constructor, R1-04 review finding
**Exploration:** quick
**Status:** revised

## D7: Record parameter ordering

**Choice:** `extensionData` goes after `constraints` (last position) in the record constructor
**Alternatives:**
- After `briefing` — conceptually similar freeform data, but breaks positional convention
- After `templates` — groups "extras" together
**Rationale:** New fields go at the end of the record constructor. This minimizes disruption to existing constructor call sites (AgentDescriptorMapper.toRecord(), tests). The Builder and toBuilder() patterns mean callers rarely use the positional constructor directly.
**Trade-offs:** None significant — the positional constructor is an internal detail; all external usage goes through Builder.
**Sources:** AgentDescriptor.java (22 existing parameters), R1-09 review finding
**Exploration:** quick
**Status:** captured

## D8: Comparator participation

**Choice:** `extensionData` participates in `AgentDescriptorComparator.compare()` via `Objects.equals()` in `compareSimpleFields`. The `COMPARED_FIELD_COUNT` constant increments from 20 to 21.
**Alternatives:**
- Exclude from comparison — two descriptors differing only in extensionData are considered equal. Registry wouldn't detect the change on re-registration.
**Rationale:** `Map.equals()` works recursively for Jackson-produced types (LinkedHashMap, ArrayList, String, Number, Boolean). After D6's deep copy, both sides are guaranteed to be composed of these types. Extension data changes should trigger re-registration — they represent meaningful configuration changes.
**Trade-offs:** None — `Objects.equals()` on the deep-copied map is correct and efficient.
**Sources:** AgentDescriptorComparator.java (COMPARED_FIELD_COUNT=20), R1-10 review finding
**Exploration:** quick
**Status:** captured
