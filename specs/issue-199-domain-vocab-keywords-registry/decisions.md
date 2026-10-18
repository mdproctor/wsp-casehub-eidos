# Decisions — eidos#199 Domain Vocabulary Keywords

## D1: keywords() as a general VocabularyTerm method

**Choice:** Add `keywords()` as a default method on `VocabularyTerm` returning `List<String>`, empty by default. Any vocabulary term can declare keywords — data-driven terms populate it from YAML, enum terms leave it empty by default.
**Alternatives:**
- No `keywords()` on VocabularyTerm, keywords only as registry-level data — breaks pattern where term metadata lives on the term (alongside `description()`, `label()`, `aliases()`)
- Standalone DomainKeywordProvider SPI — parallel system with no vocabulary integration
**Rationale:** `keywords()` is natural metadata about a term. Consistent with VocabularyTerm's role as the universal term interface. For data-driven terms, `DataDrivenTerm.keywords()` returns YAML-sourced data. Enum terms inherit the empty default.
**Trade-offs:** Enum vocabs CAN hardcode keywords in Java. Project governance (not the API) directs keyword data to YAML files instead.
**Sources:** VocabularyTerm.java:14-71, SKOS concept model
**Exploration:** quick
**Status:** captured

## D2: Vocabulary-scoped registry lookup

**Choice:** `VocabularyRegistry.keywords(String vocabUri, String value)` — resolves term, delegates to `keywords()`. Returns `List<String>`. Empty list if not found.
**Alternatives:**
- Cross-vocabulary `keywords(value)` — breaks the vocabulary-scoped pattern
- Both scoped and cross-vocabulary — unnecessary API surface
**Rationale:** Consistent with existing `resolve(vocabUri, value)`, `allTerms(vocabUri)`, `ancestors(vocabUri, value)` patterns.
**Trade-offs:** Consumer must know the vocabulary URI. Acceptable — URI constant provided.
**Sources:** VocabularyRegistry.java:30-31 (resolve pattern)
**Exploration:** quick
**Status:** captured

## D3: Explicitly declared keywords only (no auto-include of identity)

**Choice:** `keywords()` returns only what the term author declares. Does not auto-include `value()`, `label()`, or `aliases()`.
**Alternatives:**
- Auto-include value + label + aliases — conflates identity with descriptive keywords
**Rationale:** Clean separation. Identity is structural; keywords are semantic associations. Consumers compose when they need both.
**Trade-offs:** Assertion consumers need an extra step to include identity terms. Trivial.
**Sources:** First-principles analysis
**Exploration:** quick
**Status:** captured

## D4: Data-driven domain vocabulary via YAML

**Choice:** Domain vocabulary defined entirely in YAML (`META-INF/eidos/vocabularies/epistemic-domains.yaml`), NOT as a Java enum. ~20 terms with hierarchy and keywords. JSON Schema (`vocabulary.schema.json`) for IDE completion.
**Alternatives:**
- Java enum with hardcoded terms and keywords — domain terms are pure data (value, label, description, keywords, hierarchy) with no behavioral logic (`axisExactMatch`, `shadow`, `opposite`). Hardcoding data in Java violates data-driven principle.
- Java enum for structure, YAML for keywords only — still hardcodes the term list. Half measure.
**Rationale:** Domain terms have no cross-vocabulary matching logic. They're data — value + label + description + keywords + hierarchy. YAML with JSON Schema gives IDE completion (same pattern as descriptor.schema.json). Follows the established classpath YAML registrar pattern (descriptors, templates, organizations).
**Trade-offs:** No compile-time type safety for specializes() references — resolved at load time (string → term within the vocabulary). Acceptable — same resolution approach used by YAML descriptor deserialization. Hierarchy errors surface at startup, not compile time.
**Depends on:** D1 (keywords on VocabularyTerm — DataDrivenTerm implements it)
**Sources:** ClasspathYamlDescriptorRegistrar.java, ClasspathYamlTemplateRegistrar.java, descriptor.schema.json, SKOS minimal ontological commitment principle
**Exploration:** deep-analysis
**Status:** captured

## D5: VocabularyRegistry extended for non-enum vocabularies

**Choice:** New `register(String uri, String name, String version, String description, List<? extends VocabularyTerm> terms)` method on VocabularyRegistry. CdiVocabularyRegistry refactored to URI-keyed primary storage. Class-keyed `byClass` map retained as secondary index for backward-compat typed `resolve(Class, value)`. Hierarchy builder adapted for non-enum terms.
**Alternatives:**
- Keep enum-only registration, convert YAML to synthetic enum at load time — fragile, hacky, defeats the purpose
- Separate registry for data-driven vocabularies — parallel system, duplication
**Rationale:** Single registry for all vocabulary types. URI-keyed storage is the natural primary index (URI is already the unique identifier). Hierarchy builder works with VocabularyTerm instances — the enum cast is an implementation detail that can be removed.
**Trade-offs:** Moderate CdiVocabularyRegistry refactor. The internal maps change from class-keyed to URI-keyed. External API backward-compatible.
**Depends on:** D4 (data-driven vocabularies need non-enum registration)
**Sources:** CdiVocabularyRegistry.java (byUri, byClass, byClassOrdered maps), hierarchy builder
**Exploration:** deep-analysis
**Status:** captured

## D6: No keyword inheritance in registry method

**Choice:** `keywords(vocabUri, value)` returns the term's declared keywords only. No automatic ancestor keyword inclusion. `DomainKeywords.expanded()` static utility in api provides convenience for scope-inclusive keyword lookup.
**Alternatives:**
- Automatic ancestor keyword inheritance — hides what's specific vs inherited
- Two registry methods — more API surface for a 3-line utility
**Rationale:** Keeps the registry primitive composable and predictable. Consumers compose with `ancestors()` when they need expanded scope.
**Trade-offs:** Assertion consumer uses a utility instead of a single registry call. Explicit > implicit.
**Sources:** VocabularyRegistry.ancestors()
**Exploration:** deep-analysis
**Status:** captured

## D7: No validation of epistemicDomains keys

**Choice:** `epistemicDomains` keys on `AgentCapability` remain freeform strings. Vocabulary enriches; does not constrain.
**Alternatives:**
- Validate against registered vocabularies — forces custom vocabulary registration
**Rationale:** Consistent with eidos's open-string principle.
**Trade-offs:** Typos not caught. Same trade-off as slot values and disposition terms.
**Sources:** CLAUDE.md — "Slot is an open String"
**Exploration:** quick
**Status:** captured

## D8: No renderer changes in this issue

**Choice:** `epistemicDomains` rendering stays as-is. Domain keyword enrichment in rendered output is a future concern.
**Alternatives:**
- Include domain keywords in A2A_CARD — expands scope
**Rationale:** Issue targets vocabulary data model and assertion consumer. Rendering enrichment is a separate concern with its own protocol.
**Trade-offs:** A2A_CARD consumers query VocabularyRegistry directly until a follow-up issue.
**Sources:** eidos#199 issue body, capability-metadata-rendering.md protocol
**Exploration:** quick
**Status:** captured

## D9: DataDrivenTerm record in eidos-api

**Choice:** `DataDrivenTerm` record in eidos-api implementing `VocabularyTerm`. Fields: value, label, description, aliases, keywords, resolvedParents. Immutable, Tier 1, pure Java.
**Alternatives:**
- DataDrivenTerm in eidos-core — would prevent api-level utilities from using it
**Rationale:** Tier 1 placement lets `DomainKeywords` utility (also in api) work with the type. No runtime dependencies.
**Trade-offs:** api module gains a new type. Acceptable — it's a simple record with no dependencies beyond the existing api types.
**Depends on:** D1 (implements VocabularyTerm with keywords())
**Sources:** Module tier structure
**Exploration:** quick
**Status:** captured

## D10: ClasspathYamlVocabularyRegistrar in eidos-core

**Choice:** `ClasspathYamlVocabularyRegistrar` in eidos-core, reads `META-INF/eidos/vocabularies/*.yaml`. Follows the same classpath scanning pattern as `ClasspathYamlDescriptorRegistrar` and `ClasspathYamlTemplateRegistrar`.
**Alternatives:**
- In runtime module — breaks the pattern; other YAML registrars are in eidos-core
**Rationale:** Consistent with existing YAML registrar placement. Discovered by `CdiVocabularyRegistry` at `@PostConstruct` alongside enum registrars.
**Trade-offs:** eidos-core gains a new registrar. Consistent with existing pattern.
**Depends on:** D4 (YAML vocabulary format), D5 (non-enum registration), D9 (DataDrivenTerm)
**Sources:** ClasspathYamlDescriptorRegistrar.java, ClasspathYamlTemplateRegistrar.java
**Exploration:** quick
**Status:** captured
