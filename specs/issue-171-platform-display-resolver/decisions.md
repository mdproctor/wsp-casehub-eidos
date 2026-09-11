# Decisions — Issue #171: Bridge to Platform DisplayTermResolver SPI

## D1: Dual interface implementation — single class implements both SPIs

**Choice:** `DefaultDisplayTermResolver implements io.casehub.platform.api.display.DisplayTermResolver, io.casehub.eidos.api.DisplayTermResolver`. One class, both interfaces. Platform's `resolveLabel(value, vocabUri)` and eidos's default `resolveLabel(value, sourceVocabUri)` share the same signature — one implementation satisfies both.
**Alternatives:**
- Separate adapter class — `PlatformDisplayTermResolverAdapter` wrapping `DefaultDisplayTermResolver`. Unnecessary indirection for identical semantics.
- Remove eidos SPI, use platform SPI only — breaks eidos-internal consumers that need typed `DispositionAxis` parameter. Eidos SPI stays for type-safe axis-aware resolution.
**Rationale:** Both SPIs have compatible `resolveLabel(String, String)` signatures. Java resolves the collision naturally. No adapter boilerplate. CDI sees one bean satisfying both types.
**Trade-offs:** Class implements two interfaces from different packages with the same name — requires fully-qualified imports. Acceptable for a bridge that will be simplified when eidos extracts to framework-neutral core (#169).
**Sources:** Platform DisplayTermResolver.java (platform-api), eidos DisplayTermResolver.java (eidos-api)
**Exploration:** quick
**Status:** captured

## D2: mappingContext → DispositionAxis via jsonKey()

**Choice:** Parse `String mappingContext` to `DispositionAxis` using `jsonKey()` matching (camelCase: `"riskAppetite"` → `RISK_APPETITE`). Null or unrecognized context → axis-unaware mapping (falls back to `exactMatch`).
**Alternatives:**
- Match by enum name (SCREAMING_CASE) — unfriendly to API callers who don't have the enum
- Fail on unrecognized context — breaks graceful degradation principle
**Rationale:** `jsonKey()` is what appears in JSON/YAML — the natural external representation. Platform consumers won't have `DispositionAxis` on their classpath, so the string must match the serialized form.
**Trade-offs:** Adds a linear scan over 5 enum values per call when mappingContext is non-null. Negligible — display-time only.
**Sources:** DispositionAxis.java (jsonKey()), platform DisplayTermResolver.java (mappingContext parameter)
**Exploration:** quick
**Status:** captured

## D3: mapTerm exposes the cross-vocab translation step

**Choice:** Platform's `mapTerm(value, src, tgt)` → `Optional<String>` delegates to `registry.equivalentValues(src, value, tgt)`. The axis-aware variant parses context and delegates to `registry.equivalentValues(src, value, tgt, axis)`. Returns the target *value*, not the label — `mapTerm` is the translation step, `resolveLabel` is the full chain.
**Alternatives:**
- Have mapTerm return the label instead of the value — diverges from platform SPI contract which specifies Optional<String> of the mapped value
**Rationale:** Platform SPI defines `mapTerm` as term mapping (value → value), distinct from `resolveLabel` (value → display label). The eidos implementation already has this logic inside `resolveLabel` — `mapTerm` just exposes the intermediate step.
**Trade-offs:** None — straightforward delegation to existing VocabularyRegistry methods.
**Sources:** DefaultDisplayTermResolver.java:47-50 (existing equivalentValues calls)
**Exploration:** quick
**Status:** captured
