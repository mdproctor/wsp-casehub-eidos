# Avatar Field — Agent Identity Code

**Issue:** #181
**Branch:** issue-181-avatar-field

## Purpose

Add an optional `avatar` field to `AgentDescriptor` that carries a compact visual identity code. The code is opaque to eidos — blocks-ui owns the rendering semantics. Eidos stores it, derives a default when archetype is set, and surfaces it in A2A_CARD for frontend consumption.

## Avatar Code Format

Three modes, discriminated by prefix:

| Mode | Example | When |
|------|---------|------|
| Preset | `mythic:P1B` | Archetype-derived, no customisation |
| Custom | `mythic:Co/grLNBF` | User tweaked parts in the wizard |
| External | `https://example.com/img.png` | User-provided image URL |

Detection: `http://` or `https://` prefix → external URL. Otherwise → collection code (preset or custom). Eidos does not distinguish preset from custom — both are opaque collection codes.

## Field Design

`String avatar` on `AgentDescriptor`, nullable. Follows the open-String pattern (slot, archetype).

- Max length: 500 (accommodates long external URLs)
- Standard banned-character validation (same as other String fields)
- No structural validation of the code format — eidos is not the owner
- Empty adjective-style constraint: none (avatar is independent of other fields)

## Auto-Derivation

When `archetype` is set and `avatar` is null, derive a default preset code.

**Canonical archetype index:** sorted list of `{Family}/{SubArchetype}` keys (alphabetical by family name, then by sub-archetype value within family):

```
 0 = Caregiver/angel
 1 = Caregiver/guardian
 2 = Caregiver/healer
 3 = Caregiver/samaritan
 4 = Creator/artist
 5 = Creator/entrepreneur
 6 = Creator/storyteller
 7 = Creator/visionary
 ...
44 = Sovereign/judge
45 = Sovereign/patriarch
46 = Sovereign/ruler
47 = (reserved)
```

**Default code:** `mythic:P{base36(index)}`

Base36 encoding keeps codes compact (0-47 fits in 1-2 characters: `0`-`1B`).

**Default collection:** `mythic` (hardcoded constant). When blocks-ui adds more collections, a configuration property can override the default.

**Derivation location:** `ArchetypeDeriver` in runtime — extends the existing archetype derivation step. Runs at registration time in `AgentDescriptorBootstrap`.

**Index computation:** Static method on a new `AvatarCodec` utility in vocab/:

```java
public static int archetypeIndex(String archetypeValue) {
    // Binary search the sorted canonical list
}

public static String defaultCode(String collection, String archetypeValue) {
    int idx = archetypeIndex(archetypeValue);
    return idx >= 0 ? collection + ":P" + Integer.toString(idx, 36).toUpperCase() : null;
}
```

The canonical list is computed once from `ArchetypeTerm.values()` sorted by `family().name() + "/" + value()`. This is stable under enum reordering — the sort key is the identity strings, not enum ordinal.

## YAML Support

```yaml
agentId: inspector-morse
name: Inspector Morse
slot: analyst
archetype: detective
avatar: mythic:P1B
```

Or with external override:

```yaml
avatar: https://example.com/morse.png
```

Or omitted entirely (auto-derived from archetype at registration time).

Parsed in `AgentDescriptorDeserializer` via `ifString(root, "avatar", builder::avatar)`.

## Rendering

**A2A_CARD:** top-level `avatar` string field on the card JSON. Included when non-null, omitted when null.

```json
{
  "name": "Inspector Morse",
  "agentId": "inspector-morse",
  "avatar": "mythic:P1B",
  "slot": { ... },
  "archetype": { ... }
}
```

**MARKDOWN/PROSE:** not rendered. The avatar code has no meaning to an LLM — it's a visual concern for UI consumers only.

**Descriptor payload (cache key):** included via `addIfPresent(node, "avatar", descriptor.avatar())` so different avatars produce different cache keys.

## JPA Persistence

`AgentDescriptorEntity`: `String avatar` column, `VARCHAR(500)`.

`AgentDescriptorMapper`:
- `toRecord`: pass `e.avatar` directly (String → String)
- `toEntity`: `e.avatar = d.avatar()` (null passthrough)

Schema (in V1, no existing installations): `avatar VARCHAR(500)` after `archetype_adjectives`.

## AgentDescriptorComparator

Add `compareField(drifts, "avatar", desired.avatar(), actual.avatar())` and bump `COMPARED_FIELD_COUNT` by 1.

## Files Changed

| File | Change |
|------|--------|
| `api/.../AgentDescriptor.java` | Add `String avatar` field, builder, toBuilder, validation |
| `api/.../AgentDescriptorValidator.java` | Add `MAX_AVATAR = 500` |
| `api/.../AgentDescriptorComparator.java` | Add avatar comparison, bump count |
| `vocab/.../AvatarCodec.java` | New: canonical index computation + default code generation |
| `eidos-core/.../AgentDescriptorDeserializer.java` | Parse `avatar` from YAML |
| `eidos-core/.../EidosRenderPipeline.java` | Add avatar to A2A_CARD + descriptor payload |
| `runtime/.../ArchetypeDeriver.java` | Extend: derive avatar when archetype set and avatar null |
| `runtime/.../AgentDescriptorEntity.java` | Add `String avatar` column |
| `runtime/.../AgentDescriptorMapper.java` | Map avatar in both directions |
| `runtime/...V1__initial_schema.sql` | Add `avatar VARCHAR(500)` column |
| `docs/guides/consumer-guide.md` | Document avatar field |

## References

- `specs/avatar-generator-contract.md` Section 5 — avatar generation payload
- `ArchetypeTerm.java` — the 48 sub-archetypes that drive index computation
- `ArchetypeDeriver.java` — derivation location (runtime/registrar)
- blocks-ui#167 — archetype avatar system (SVG templates, collections, compact codes)
- blocks-ui#166 — agent setup wizard (writes avatar field during creation)
