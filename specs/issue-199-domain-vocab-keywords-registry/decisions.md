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

## D2: No dedicated keywords() method on VocabularyRegistry

**Choice:** VocabularyRegistry does NOT get a dedicated `keywords()` method. Consumers compose via `resolve(vocabUri, value).map(VocabularyTerm::keywords).orElse(List.of())`. `DomainKeywords` utility (D6) provides the convenience composition layer.
**Alternatives:**
- `VocabularyRegistry.keywords(String vocabUri, String value)` convenience method — redundant with resolve() + accessor, inflates API surface with no internal logic
- Cross-vocabulary `keywords(value)` — breaks the vocabulary-scoped pattern
**Rationale:** VocabularyRegistry provides orthogonal primitives (resolve, allTerms, ancestors, subsumes, match) that each involve non-trivial internal operations (index lookup, hierarchy traversal, cross-vocabulary logic). `keywords()` would be the first pure delegation method — `resolve()` + accessor. This sets a bad precedent. DomainKeywords.expanded() (D6) is the right home for convenience composition.
**Trade-offs:** Consumers need the Optional chain for raw keyword lookup. Acceptable — DomainKeywords handles the common case.
**Sources:** VocabularyRegistry.java (existing method patterns), R1-06 reviewer analysis
**Exploration:** quick → revised after review
**Status:** revised (was: convenience method on registry; now: no registry method, compose via resolve + accessor)

## D3: Explicitly declared keywords only (no auto-include of identity)

**Choice:** `keywords()` returns only what the term author declares. Does not auto-include `value()`, `label()`, or `aliases()`.
**Alternatives:**
- Auto-include value + label + aliases — conflates identity with descriptive keywords
**Rationale:** Clean separation. Identity is structural; keywords are semantic associations. Consumers compose when they need both. The composition mechanism is `DomainKeywords.expanded()` (D6).
**Trade-offs:** Assertion consumers need an extra step to include identity terms. Trivial — D6 provides the utility.
**Depends on:** D6 (DomainKeywords provides the composition utility that makes the "extra step" trivial)
**Sources:** First-principles analysis
**Exploration:** quick
**Status:** captured

## D4: Data-driven domain vocabulary via YAML

**Choice:** Domain vocabulary defined entirely in YAML (`META-INF/eidos/vocabularies/epistemic-domains.yaml`), NOT as a Java enum. ~20 terms with hierarchy and keywords. Optional JSON Schema (`vocabulary.schema.json`) for IDE completion — a new artifact, not following an existing project pattern.
**Alternatives:**
- Java enum with hardcoded terms and keywords — domain terms are pure data (value, label, description, keywords, hierarchy) with no behavioral logic (`axisExactMatch`, `shadow`, `opposite`). Hardcoding data in Java violates data-driven principle.
- Java enum for structure, YAML for keywords only — still hardcodes the term list. Half measure.
**Rationale:** Domain terms have no cross-vocabulary matching logic. They're data — value + label + description + keywords + hierarchy. Follows the established classpath YAML registrar pattern (ClasspathYamlDescriptorRegistrar, ClasspathYamlTemplateRegistrar).
**Trade-offs:** No compile-time type safety for specializes() references — resolved at load time (string → term within the vocabulary). Acceptable — same resolution approach used by YAML descriptor deserialization. Hierarchy errors surface at startup, not compile time.
**Depends on:** D1 (keywords on VocabularyTerm — DataDrivenTerm implements it)
**Sources:** ClasspathYamlDescriptorRegistrar.java, ClasspathYamlTemplateRegistrar.java, SKOS minimal ontological commitment principle
**Exploration:** deep-analysis
**Status:** captured

## D5: VocabularyRegistry extended for non-enum vocabularies

**Choice:** New `register(String uri, String name, String version, String description, List<? extends VocabularyTerm> terms)` method on VocabularyRegistry. CdiVocabularyRegistry refactored to URI-keyed primary storage. Class-keyed `byClass` map retained as secondary index for backward-compat typed `resolve(Class, value)`. Hierarchy builder restructured for non-enum terms.

**Structural changes required:**
1. `byUri: Map<String, Class<? extends Enum<?>>>` → new URI-keyed `termsByUri: Map<String, Map<String, VocabularyTerm>>` for direct term lookup. `byUri` class map retained for enum vocabs only.
2. New `orderedByUri: Map<String, List<? extends VocabularyTerm>>` for URI-keyed ordered term lists. `byClassOrdered` retained for enum vocabs.
3. `vocabularyMetadata()` return type changes from `Optional<VocabularyMetadata>` (annotation) to `Optional<VocabularyInfo>` (record). For enum vocabs, populated from `@VocabularyMetadata` annotation; for data-driven, from registration parameters. See D13.
4. **Hierarchy builder**: `buildAllHierarchyIndexes()` parameter type changes from `Map<String, Class<? extends Enum<?>>>` to a unified representation covering both vocab types. The `((Enum<?>) parent).getDeclaringClass()` cast (for term→vocabulary lookup) replaced by a `termToVocabUri` map populated during term collection. The `vocab.getEnumConstants()` iteration replaced by unified iteration over enum constants and data-driven term lists.
5. All string-based `resolve()`, `allTerms()`, `vocabularyMetadata()` methods updated to check URI-keyed storage first, with class-keyed maps as secondary index for enum vocabs.
6. `byClass` and `byClassOrdered` retained as secondary indexes for typed `resolve(Class, value)` and `allTerms(Class)` — backward-compatible for enum consumers.

**Alternatives:**
- Keep enum-only registration, convert YAML to synthetic enum at load time — fragile, hacky, defeats the purpose
- Separate registry for data-driven vocabularies — parallel system, duplication
**Rationale:** Single registry for all vocabulary types. URI-keyed storage is the natural primary index. The enum-specific casts and iterations in the hierarchy builder are implementation coupling, not architectural necessity — they can be replaced by the `termToVocabUri` map pattern that already exists conceptually in the hierarchy builder's local `termToUri` variable.
**Trade-offs:** Substantial CdiVocabularyRegistry refactoring — touches every internal map and every read-path method in a 574-line class. External API backward-compatible for enum consumers. Worth the cost to achieve a unified vocabulary system.
**Depends on:** D4 (data-driven vocabularies need non-enum registration), D11 (VocabularyTerm contract opens), D13 (VocabularyInfo replaces annotation return type)
**Sources:** CdiVocabularyRegistry.java (byUri, byClass, byClassOrdered maps, buildAllHierarchyIndexes), R1-03/R1-04 scope analysis
**Exploration:** deep-analysis
**Status:** revised (scope characterization corrected from "moderate" to explicit structural change list; hierarchy builder changes detailed)

## D6: No keyword inheritance — composition via DomainKeywords

**Choice:** Term keyword lookup returns only declared keywords, with no automatic ancestor keyword inclusion. `DomainKeywords.expanded()` static utility in eidos-api composes `resolve()` + `VocabularyTerm.keywords()` + `ancestors()` for scope-inclusive keyword lookup. There is no dedicated keyword method on VocabularyRegistry (see D2).
**Alternatives:**
- Automatic ancestor keyword inheritance — hides what's specific vs inherited
- Two registry methods — more API surface for a 3-line utility
**Rationale:** Keeps the registry primitives composable and predictable. Consumers compose with `resolve()` + `VocabularyTerm.keywords()` for direct lookup, or use `DomainKeywords.expanded()` when they need ancestor-inclusive scope.
**Trade-offs:** Assertion consumer uses a utility instead of a single call. Explicit > implicit.
**Sources:** VocabularyRegistry.ancestors(), D2 (no registry method)
**Exploration:** deep-analysis
**Status:** revised (phrasing updated to align with D2 revision removing registry method)

## D7: No validation of epistemicDomains keys

**Choice:** `epistemicDomains` keys on `AgentCapability` remain freeform strings. Vocabulary enriches; does not constrain.
**Alternatives:**
- Validate against registered vocabularies — forces custom vocabulary registration
**Rationale:** Consistent with eidos's open-string principle.
**Trade-offs:** Typos not caught. Same trade-off as slot values and disposition terms. Implementors should log at WARN when a descriptor's epistemicDomains key has no matching vocabulary registration, so silent degradation is visible in diagnostics without constraining the open-string contract.
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

**Choice:** `ClasspathYamlVocabularyRegistrar` in eidos-core, reads `META-INF/eidos/vocabularies/*.yaml`. Implements `DataDrivenVocabularyRegistrar` (D12). Follows the same classpath scanning pattern as `ClasspathYamlDescriptorRegistrar` and `ClasspathYamlTemplateRegistrar`.
**Alternatives:**
- In runtime module — breaks the pattern; other YAML registrars are in eidos-core
**Rationale:** Consistent with existing YAML registrar placement. Discovered by `CdiVocabularyRegistry` at `@PostConstruct` via `Instance<DataDrivenVocabularyRegistrar>` alongside `Instance<VocabularyRegistrar>` for enum vocabs.
**Trade-offs:** eidos-core gains a new registrar. Consistent with existing pattern.
**Depends on:** D4 (YAML vocabulary format), D5 (non-enum registration), D9 (DataDrivenTerm), D12 (DataDrivenVocabularyRegistrar SPI)
**Sources:** ClasspathYamlDescriptorRegistrar.java, ClasspathYamlTemplateRegistrar.java
**Exploration:** quick → updated after review to specify SPI mechanism
**Status:** revised (SPI mechanism specified: implements DataDrivenVocabularyRegistrar)

## D11: VocabularyTerm contract opens to non-enum implementations

**Choice:** VocabularyTerm interface contract changes from "enum only" to "any implementation." The Javadoc "Implemented by enum constants" is updated. DataDrivenTerm (D9) is the first non-enum implementation.
**Alternatives:**
- Keep VocabularyTerm as enum-only, create a separate interface for data-driven terms — duplication, parallel type hierarchies, every consumer handles both
- Keep VocabularyTerm as enum-only, synthesize enum at load time — fragile, defeats the purpose
**Rationale:** VocabularyTerm defines the contract for vocabulary terms: value, label, description, aliases, specializes. None of these methods are inherently enum-specific. The enum restriction was an implementation assumption, not a design invariant. The enum-specific behaviors (exactMatch, axisExactMatch, opposite, defaultProfile) return empty defaults — data-driven terms inherit these safely.
**Trade-offs:** Consumers that cast VocabularyTerm to Enum<?> will break. CdiVocabularyRegistry is the primary offender — addressed in D5 refactoring. External consumers using typed `resolve(Class, value)` are unaffected.
**Sources:** VocabularyTerm.java:14 (Javadoc), CdiVocabularyRegistry enum casts, R1-11 reviewer analysis
**Exploration:** implicit → surfaced during review
**Status:** captured

## D12: DataDrivenVocabularyRegistrar SPI

**Choice:** New `DataDrivenVocabularyRegistrar` interface in eidos-api. Method: `List<DataDrivenVocabulary> vocabularies()` where `DataDrivenVocabulary` is a record carrying URI, name, version, description, and `List<DataDrivenTerm>`. CdiVocabularyRegistry discovers `Instance<DataDrivenVocabularyRegistrar>` at `@PostConstruct` alongside `Instance<VocabularyRegistrar>`.
**Alternatives:**
- Widen existing VocabularyRegistrar to return a union type — breaks all 15 existing registrar implementations for no architectural benefit; wrong abstraction level
- ClasspathYamlVocabularyRegistrar directly injects VocabularyRegistry and calls register() in @PostConstruct — creates initialization ordering dependency that the CDI discovery pattern avoids
- No SPI, VocabularyRegistrar stays enum-only — leaves no CDI-discoverable mechanism for data-driven vocabularies
**Rationale:** Follows the existing SPI pattern: `AgentDescriptorRegistrar` returns `List<AgentDescriptor>`, `TemplateRegistrar` returns `List<DescriptorTemplate>`. Separate SPI for separate vocabulary types keeps concerns clean. CDI discovery is consistent with the existing `@PostConstruct` pattern in CdiVocabularyRegistry.init().
**Trade-offs:** New interface and record in eidos-api. CdiVocabularyRegistry.init() adds a second loop for data-driven providers.
**Depends on:** D9 (DataDrivenTerm), D11 (VocabularyTerm contract opens)
**Sources:** AgentDescriptorRegistrar.java, TemplateRegistrar.java, CdiVocabularyRegistry.init(), R1-05 and R1-12 reviewer analysis
**Exploration:** implicit → surfaced during review
**Status:** captured

## D13: VocabularyInfo record replaces annotation return type

**Choice:** New `VocabularyInfo` record in eidos-api with fields: `uri`, `name`, `version`, `description`. `VocabularyRegistry.vocabularyMetadata(String uri)` return type changes from `Optional<VocabularyMetadata>` (annotation) to `Optional<VocabularyInfo>` (record). For enum vocabs, `VocabularyInfo` is populated from the `@VocabularyMetadata` annotation at registration time. For data-driven vocabs, populated from YAML metadata. The `@VocabularyMetadata` annotation is unchanged — it remains the declaration mechanism for enum vocab classes.
**Alternatives:**
- Return `Optional.empty()` for data-driven vocabs — inconsistent; domain vocabularies have metadata but can't surface it
- Create synthetic VocabularyMetadata annotation proxy at runtime — hacky; annotation dynamic proxies are fragile and don't play well with CDI
- Keep annotation return type, add new method for data-driven — dual API, consumers need to know which to call
**Rationale:** Clean separation: annotation is for declaration (on enum classes), record is for runtime queries (from any vocab type). Migration is mechanical — callers use `meta.name()`, `meta.description()` which have identical signatures on the record.
**Trade-offs:** Breaking API change on `vocabularyMetadata()` return type. ~14 sites total: 8 callers (6 in EidosRenderPipeline, 2 in CdiVocabularyRegistryTest) plus ~6 stub/test implementations of VocabularyRegistry that override `vocabularyMetadata()` (TestVocabularyRegistry, StubVocabularyRegistry variants in CapabilityResolverTest, DefaultCapabilityHealthBehavioralViolationTest, etc.). All updates are mechanical — method signatures on the record match the annotation accessor names.
**Sources:** VocabularyMetadata.java, VocabularyRegistry.vocabularyMetadata(), EidosRenderPipeline usage analysis, R1-13 reviewer analysis
**Exploration:** implicit → surfaced during review
**Status:** captured

## D14: DataDrivenTerm hierarchy is vocabulary-internal only

**Choice:** DataDrivenTerm `specializes()` references are resolved within the same vocabulary at YAML load time. Cross-vocabulary specialization (a data-driven term specializing a term from a different vocabulary) is not supported for data-driven terms.
**Alternatives:**
- Support cross-vocabulary specialization for data-driven terms — requires the registrar to access already-registered vocabularies during YAML loading, creating initialization ordering dependencies. DataDrivenTerm would need to hold unresolved string references resolved later by the hierarchy builder, changing the `specializes()` contract.
- Defer all resolution to hierarchy builder — changes `specializes()` from "returns resolved term instances" to "returns identifiers to be resolved"; leaks resolution responsibility out of the vocabulary provider
**Rationale:** The epistemic domains use case is vocabulary-internal: "engineering" specializes "stem", "clinical" specializes "healthcare". Cross-vocabulary specialization between a domain term and a capability term is semantically incoherent. Vocabulary-internal resolution keeps YAML files self-contained and avoids initialization ordering dependencies. The hierarchy builder already handles cross-vocabulary injection (terms injected into a vocabulary's index via ancestor relationships), which covers the matching use case without requiring cross-vocabulary specialization at the term level.
**Trade-offs:** If a future data-driven vocabulary genuinely needs cross-vocabulary specialization, it can be migrated to an enum vocabulary (which supports it via Java references) or the mechanism can be extended. Current use case doesn't need it.
**Sources:** CdiVocabularyRegistry.buildAllHierarchyIndexes(), epistemic domains design, R1-14 reviewer analysis
**Exploration:** implicit → surfaced during review
**Status:** captured
