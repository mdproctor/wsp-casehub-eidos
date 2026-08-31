## D1: Preprocessing pipeline integration point

**Choice:** Two-pass generic maps — parse YAML to Map<String, Object>, preprocess with yaml-core (variables, forEach, when, CSV), then convertValue through existing EidosDescriptorModule
**Alternatives:**
- String-level preprocessing — only works for variables, breaks down for forEach/when expansion
- Embed in Jackson deserializer — tangles yaml-core into Jackson module, harder to test
**Rationale:** Respects yaml-core's design (operate on generic maps), cleanly separates preprocessing from deserialization, reuses EidosDescriptorModule unchanged
**Trade-offs:** Two ObjectMapper instances (one plain for generic parse, one with EidosDescriptorModule for typed deserialization) — minor complexity
**Sources:** ClasspathYamlDescriptorRegistrar.java, ForEachExpander.java, ForEachExpanderTest.java, EidosDescriptorModule.java
**Exploration:** quick
**Status:** captured

## D2: Scope — all four yaml-core features

**Choice:** Variables, forEach, when, and CSV data sources all in scope for this issue
**Alternatives:**
- Phase: vars+forEach+when first, CSV follow-up — creates more integration work than it saves
- Variables+forEach only — misses conditional inclusion
**Rationale:** All four share the same preprocessing pipeline; yaml-core handles them as a unit
**Trade-offs:** Slightly larger PR, but the features are cohesive
**Sources:** casehubio/eidos#149 issue body
**Exploration:** quick
**Status:** captured

## D3: yaml-core dependency placement — runtime only

**Choice:** Add casehub-platform-yaml-core to runtime/pom.xml only
**Alternatives:**
- API module — YAGNI, API is pure Java records not a YAML processing layer
**Rationale:** ClasspathYamlDescriptorRegistrar is the only consumer, lives in runtime
**Trade-offs:** Consumers can't build custom preprocessing pipelines using eidos-api alone — but that's not a use case
**Sources:** runtime/pom.xml, api module structure
**Exploration:** quick
**Status:** captured

## D4: ForEach element keying — agentId

**Choice:** Key descriptors by raw agentId string for ForEachExpander input map
**Alternatives:**
- Positional index — produces meaningless stamped IDs
- Optional explicit key field — adds YAML surface area for no benefit
**Rationale:** agentId is already required and unique within a file; natural key for expansion tracking
**Trade-offs:** Template agentId containing ${each.*} is used as-is for the map key — the resolved agentId comes from the adapter's stamp() method
**Sources:** ForEachExpander API, AgentDescriptor.agentId requirement
**Exploration:** quick
**Status:** captured

## D5: Variable sources — var + config

**Choice:** Two prefix sources: `var` (static values from YAML `variables` section) and `config` (Quarkus MicroProfile Config via ConfigProvider)
**Alternatives:**
- var only — misses environment-specific configuration that Quarkus already provides
- var + config + env — env vars are already accessible via Quarkus config (`${config.ENV_VAR}`), so a dedicated `env` prefix is redundant
**Rationale:** DescriptorPreprocessor takes Map<String, VariableSource> — pure Java, Quarkus-agnostic. ClasspathYamlDescriptorRegistrar (already @ApplicationScoped) wires in both sources. Unit tests without Quarkus pass var only.
**Trade-offs:** None — config source is a one-liner VariableSource lambda in the registrar
**Sources:** VariableResolver.java, VariableSource.java, ClasspathYamlDescriptorRegistrar.java
**Exploration:** quick
**Status:** captured

## D6: CSV data source format — both classpath and inline

**Choice:** dataSources section supports both `file:` (classpath resource) and `csv:` (inline content)
**Alternatives:** None — this is completeness, not a choice
**Rationale:** Both produce a CSV string for CsvParser.parse(); the YAML format should accept whichever the user provides
**Trade-offs:** None
**Sources:** CsvParser.java, CsvDataSource.java
**Exploration:** quick
**Status:** captured

## D7: Preprocessing orchestration — new DescriptorPreprocessor class

**Choice:** New standalone `DescriptorPreprocessor` class in `runtime/yaml/` that takes a raw Map + variable sources, runs the yaml-core pipeline, returns `List<Map<String, Object>>`
**Alternatives:**
- Inline in ClasspathYamlDescriptorRegistrar — bloats the registrar with CSV loading, variable source building
- Separate Maven module — overkill for one class with one consumer
**Rationale:** Clean single-responsibility, independently testable. ClasspathYamlDescriptorRegistrar calls it before Jackson deserialization.
**Trade-offs:** One more class — trivial
**Sources:** ClasspathYamlDescriptorRegistrar.java
**Exploration:** quick
**Status:** captured

## D8: Two-level strictness for unknown properties

**Choice:** Top-level parse is lenient (extract and strip preprocessing keys). Per-descriptor deserialization remains strict via EidosDescriptorModule.
**Alternatives:**
- All lenient — loses safety net for descriptor field typos
- Explicit allowlist of top-level keys — most strict but requires updating for future extensions
**Rationale:** Catches typos in descriptor fields (agentId, slot, disposition) while allowing variables/iterations/dataSources at the top level
**Trade-offs:** A typo in a top-level preprocessing key (e.g., `varibles` instead of `variables`) would silently be ignored — but the unresolved `${var.*}` references would fail loudly
**Sources:** EidosDescriptorModule.java (FAIL_ON_UNKNOWN_PROPERTIES=true)
**Exploration:** quick
**Status:** captured

## D9: Expansion limit — 100 per template

**Choice:** maxExpansion = 100 per forEach template, hardcoded constant
**Alternatives:**
- 1000 — very permissive, risk of silent startup performance issues
- Configurable via Quarkus config — adds configuration surface for a rarely-tuned value
**Rationale:** 100 agents per template covers team/tenancy provisioning generously. ForEachExpander enforces the limit with a clear error message.
**Trade-offs:** Rare use cases needing >100 agents per template would require a code change — but at that scale, questions about registry performance should be answered first
**Sources:** ForEachExpander.java maxExpansion parameter
**Exploration:** quick
**Status:** captured
