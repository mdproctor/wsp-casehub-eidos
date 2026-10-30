## D1: YAML module location

**Choice:** vocab/ (META-INF/eidos/archetypes.yaml)
**Alternatives:**
- api/ — makes data available without vocab dep, but loader needs vocab types
- New module — maximum decoupling but overkill for a single YAML file
**Rationale:** Keep data with its consumers — ArchetypeCompatibility, ArchetypeTerm, ArchetypeFamily all live in vocab/
**Trade-offs:** Consumers outside vocab need a vocab dependency to access the YAML
**Sources:** ArchetypeCompatibility.java, ArchetypeTerm.java (both in vocab/)
**Exploration:** quick
**Status:** captured

## D2: Source of truth split

**Choice:** Java enums own structural truth (what exists), YAML owns data truth (what it means, how things relate)
**Alternatives:**
- YAML fully authoritative — enums generated or removed. Eliminates dual maintenance but loses compile-time type safety
- Java fully authoritative — YAML is validated copy. Simpler but two copies of data
**Rationale:** Enums provide compile-time type safety (ArchetypeTerm.SAGE); YAML provides the mappings, weights, and compatibility data that changes independently
**Trade-offs:** Bidirectional validation test required to keep them in sync
**Sources:** ArchetypeTerm.java, ArchetypeFamily.java, archetypes.yaml
**Exploration:** quick
**Status:** captured

## D3: ArchetypeCompatibility loading mechanism

**Choice:** Stay static — load YAML in a static initializer block
**Alternatives:**
- CDI-managed @ApplicationScoped — more testable but changes all calling patterns
- Hybrid lazy-load — static API with lazy YAML parse on first access
**Rationale:** No API change for callers (ArchetypeResolver, etc). Simplest migration — same pattern, different data source
**Trade-offs:** Harder to test with alternative data; static initializer errors surface at class load time
**Sources:** ArchetypeCompatibility.java (static FAMILY_MAP/AFFINITY_MAP), ArchetypeResolver.java (static caller)
**Exploration:** quick
**Status:** captured

## D4: MBTI selection for auto-derivation

**Choice:** Use the first MBTI type in the archetype's affinity list
**Alternatives:**
- Blend all — average profiles from all compatible MBTI types. More nuanced but harder to debug
- Best match via ArchetypeResolver — full resolution chain with fallback. More sophisticated, more complex
**Rationale:** YAML already orders affinities by relevance (most natural fit first). Simple, deterministic, tunable via data file ordering
**Trade-offs:** Loses nuance from secondary compatible types
**Sources:** archetypes.yaml families.*.subArchetypes.*.affinities.mbti ordering, PersonalityTypeDeriver.java
**Exploration:** quick
**Status:** captured

## D5: Auto-briefing composition hook

**Choice:** Render-time in EidosRenderPipeline
**Alternatives:**
- Registration-time in DescriptorCollector — pre-populate briefing field. Simpler rendering but stores derived data and can't adapt per format
**Rationale:** Descriptor stays minimal (no derived briefing stored). Different formats can compose differently. Consistent with how archetype personality section is already rendered at lines 590+
**Trade-offs:** Briefing not available via descriptor.briefing() — only in rendered output
**Sources:** EidosRenderPipeline.java lines 754-758 (briefing hook), assembleMarkdownArchetype() line 590
**Exploration:** quick
**Status:** captured

## D6: Schema-YAML sync mechanism

**Choice:** Validation test — reads both files, asserts enum arrays match
**Alternatives:**
- Build-time generation — Maven plugin generates schema from YAML. Fully automated but adds build complexity
- Test-generated + committed — test generates, diffs against committed version. Fails if outdated
**Rationale:** Manual schema updates with automated drift detection. Simplest approach, no build pipeline changes
**Trade-offs:** Developer must manually update schema when YAML values change (test catches the drift)
**Sources:** descriptor.schema.json, archetypes.yaml, ArchetypeDataValidationTest.java
**Exploration:** quick
**Status:** captured
