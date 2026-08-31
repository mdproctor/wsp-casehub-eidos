# yaml-core for Parameterized Descriptor YAML

**Issue:** casehubio/eidos#149
**Date:** 2026-08-31

## Summary

Adopt `casehub-platform-yaml-core` to add variable resolution, forEach expansion, conditional inclusion (`when`), and CSV data sources to `descriptors.yaml`. This eliminates copy-paste duplication for multi-tenant deployments, team-based agent provisioning, and environment-conditional agents.

## YAML Format

After this change, `descriptors.yaml` supports four new top-level sections alongside the existing `descriptors` list:

```yaml
variables:
  tenancy: acme-corp
  env: production

iterations:
  teams:
    as: team
    in: [frontend, backend, infra]

dataSources:
  roster:
    file: META-INF/eidos/agents.csv    # classpath resource
  inline-data:
    csv: |                              # inline CSV content
      name:STRING, role:STRING, quality:DECIMAL
      alice, reviewer, 0.95
      bob, planner, 0.80

descriptors:
  # Static descriptor (unchanged from current format)
  - agentId: global-reviewer
    name: Global Reviewer
    slot: reviewer
    tenancyId: ${var.tenancy}
    capabilities:
      - name: code-review

  # forEach expansion — stamps one descriptor per team
  - agentId: ${each.team}-reviewer
    name: ${each.team} Reviewer
    slot: reviewer
    tenancyId: ${var.tenancy}
    forEach: teams
    capabilities:
      - name: code-review
        tags: [${each.team}]

  # Conditional inclusion — only in production
  - agentId: audit-agent
    name: Audit Agent
    slot: auditor
    tenancyId: ${var.tenancy}
    when: "${var.env_production}"
    capabilities:
      - name: audit

  # CSV-driven expansion — one descriptor per roster row
  - agentId: ${each.agent.name}-agent
    name: ${each.agent.name}
    slot: ${each.agent.role}
    tenancyId: ${var.tenancy}
    forEach:
      as: agent
      in: roster
    capabilities:
      - name: ${each.agent.role}
        qualityHint: ${each.agent.quality}
```

### Variable resolution

Two prefix sources:

| Prefix | Source | Available |
|--------|--------|-----------|
| `var` | Static values from the YAML `variables` section | Always |
| `config` | Quarkus MicroProfile Config (`ConfigProvider.getConfig()`) | When Quarkus is present (i.e., always in runtime; absent in plain unit tests) |
| `each` | ForEach iteration context (managed by yaml-core) | During forEach expansion only |

Syntax: `${prefix.name}` — bare `${name}` is rejected with a clear error pointing to available prefixes.

### forEach expansion

Two forms:

**Named group** — references a top-level `iterations` entry:
```yaml
forEach: teams    # references iterations.teams
```

**Inline** — declares values directly:
```yaml
forEach:
  as: team
  in: [frontend, backend]
```

**CSV data source** — inline form where `in` references a data source name:
```yaml
forEach:
  as: agent
  in: roster    # references dataSources.roster
```

Resolution order for `in` values: if `in` is a string, check `dataSources` first, then `iterations`. This means data source names and iteration group names share a namespace — a collision is an error.

CSV rows are available as structured objects: `${each.agent.name}`, `${each.agent.role}`.

Expansion limit: 100 per template (hardcoded). ForEachExpander throws `IllegalStateException` with clear context if exceeded.

### Conditional inclusion

```yaml
when: "${var.env_production}"
```

Evaluated after variable resolution. Values: `true/false/yes/no/on/off/y/n/1/0`. Any other value throws `IllegalArgumentException`. Works on both static and forEach-expanded descriptors.

### Backward compatibility

All new sections (`variables`, `iterations`, `dataSources`) and per-descriptor fields (`forEach`, `when`) are optional. Existing `descriptors.yaml` files without them work unchanged — the preprocessing pipeline is a no-op when no preprocessing keys are present.

## Architecture

### Pipeline

```
descriptors.yaml (InputStream)
        │
        ▼
┌─────────────────────────┐
│ Plain ObjectMapper       │  Parse to Map<String, Object>
│ (no custom module)       │  (lenient — unknown top-level keys OK)
└─────────────────────────┘
        │
        ▼
┌─────────────────────────┐
│ DescriptorPreprocessor   │  Extract variables/iterations/dataSources
│                          │  Build VariableResolver + IterationGroups
│                          │  Load + parse CSV data sources
│                          │  Run ForEachExpander with DescriptorForEachAdapter
│                          │  → List<Map<String, Object>> (resolved descriptor maps)
└─────────────────────────┘
        │
        ▼
┌─────────────────────────┐
│ ObjectMapper with         │  convertValue(map, JsonNode) per descriptor
│ EidosDescriptorModule    │  → AgentDescriptor via existing deserializers
│ (FAIL_ON_UNKNOWN = true) │  (strict — catches typos in descriptor fields)
└─────────────────────────┘
        │
        ▼
    List<AgentDescriptor>
```

### Two-level strictness

- **Top level:** Lenient. The plain ObjectMapper parses the full YAML including `variables`, `iterations`, `dataSources`. `DescriptorPreprocessor` extracts and strips these keys before per-descriptor deserialization.
- **Per descriptor:** Strict. Each resolved descriptor map is deserialized through `EidosDescriptorModule` with `FAIL_ON_UNKNOWN_PROPERTIES=true`. The `forEach` and `when` fields are stripped by the adapter before deserialization.

A typo in a top-level key (e.g., `varibles`) is silently ignored at parse time, but unresolved `${var.*}` references fail loudly via `UnresolvedVariableException`.

### New classes

#### `DescriptorPreprocessor` (runtime/yaml/)

Standalone, Quarkus-agnostic. Takes:
- `Map<String, Object>` — the raw parsed YAML
- `Map<String, VariableSource>` — prefix sources (`var`, `config`)
- `ClassLoader` — for loading CSV classpath resources

Returns `List<Map<String, Object>>` — resolved descriptor maps ready for Jackson deserialization.

Steps:
1. Extract `variables` → build `var` VariableSource
2. Combine with caller-provided sources (e.g., `config`) → build `VariableResolver`
3. Extract `iterations` → build `Map<String, IterationGroup>`
4. Extract `dataSources` → load CSV files/inline content via `CsvParser`
5. Extract `descriptors` list → convert to `LinkedHashMap<String, Map<String, Object>>` keyed by raw `agentId`
6. Partition descriptors: CSV-backed forEach vs. normal (string-list/named-group forEach or no forEach)
7. Expand normal descriptors via `ForEachExpander.expand()` with `DescriptorForEachAdapter`, maxExpansion=100
8. Expand CSV-backed descriptors directly: for each CSV row, build a resolver with both `withEachContext(as, rowKey)` and `withEachRowContext(as, rowMap)`, resolve variables, evaluate `when` conditions, enforce expansion limit
9. Merge results maintaining original descriptor order (interleaved by position in the input list)
10. Strip `forEach` and `when` keys from each expanded map
11. Return the list of resolved maps

**Why CSV is handled separately:** `ForEachExpander` iterates over string values via `withEachContext()`. CSV rows require structured field access (`${each.agent.name}`) via `withEachRowContext()` — which `ForEachExpander` does not set up. Rather than shoehorning row context through the adapter's `stamp()` method (which can't extract the current iteration value from the resolver), CSV expansion is done directly using `VariableResolver`'s native row context support. The duplicated logic is ~15 lines (when check + expansion limit).

#### `DescriptorForEachAdapter` (runtime/yaml/)

Implements `ForEachAdapter<Map<String, Object>>`:

| Method | Behavior |
|--------|----------|
| `stamp(template, stampedId, resolver)` | Deep-copy the template map, resolve all `${...}` variables via `resolver.resolveMap()`, return the resolved map |
| `getForEach(element)` | Return `element.get("forEach")` — String (group ref) or Map (inline) |
| `getId(element)` | Return `element.get("agentId").toString()` |
| `getWhen(element)` | Return `(String) element.get("when")` |

### Modified classes

#### `ClasspathYamlDescriptorRegistrar`

The `loadFrom()` method changes from:
1. Parse YAML → `DescriptorFile` via `EidosDescriptorModule`

To:
1. Parse YAML → `Map<String, Object>` via plain `ObjectMapper`
2. Call `DescriptorPreprocessor.preprocess()` → `List<Map<String, Object>>`
3. For each resolved map: `eidosMapper.convertValue(map, AgentDescriptor.class)` via `EidosDescriptorModule`

The CDI bean gains an injected `org.eclipse.microprofile.config.Config` (optional) to provide the `config` variable source.

#### `DescriptorFile` (inner class)

No longer used for the primary parse path. Retained only if needed for backward compatibility — otherwise removed.

### Dependency

Add to `runtime/pom.xml`:
```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-yaml-core</artifactId>
</dependency>
```

Version managed by `casehub-platform-parent` BOM.

## Testing

### Unit tests (no Quarkus)

`DescriptorPreprocessorTest`:
- Variable resolution in all descriptor fields (agentId, name, slot, briefing, capability names)
- forEach with named iteration group — stamps correct count, resolves `${each.*}`
- forEach with inline values — stamps and resolves
- forEach with CSV data source (inline CSV content) — structured row access `${each.row.field}`
- forEach with CSV data source (classpath file) — loads and parses
- `when: true` includes, `when: false` excludes
- `when` with forEach — per-copy selective exclusion
- Mixed static + forEach descriptors — correct ordering and count
- Expansion limit exceeded → `IllegalStateException`
- Unresolved variable → `UnresolvedVariableException` with context
- No preprocessing keys → passthrough (backward compatibility)
- Nested variable resolution in capabilities, goals, constraints, templates, disposition

`DescriptorForEachAdapterTest`:
- `stamp()` resolves variables in the map
- `getForEach()`/`getId()`/`getWhen()` extract correct fields
- `stamp()` strips `forEach` and `when` from the result

### Integration tests (ClasspathYamlDescriptorRegistrarTest)

Extend existing tests:
- Parameterized YAML with variables → correct AgentDescriptor field values
- forEach expansion → correct descriptor count and resolved agentIds
- `when` condition → descriptors included/excluded correctly
- CSV data source → bulk provisioning produces correct descriptors
- `config` variable source → resolves Quarkus config properties
- Existing non-parameterized YAML → works unchanged (regression)

## Scope boundaries

**In scope:**
- `DescriptorPreprocessor` + `DescriptorForEachAdapter` in runtime/yaml/
- Modify `ClasspathYamlDescriptorRegistrar` to use the preprocessing pipeline
- `var` + `config` variable sources
- forEach (named groups, inline, CSV data sources)
- `when` conditional inclusion
- CSV loading (classpath + inline)
- Unit and integration tests

**Out of scope:**
- `templates.yaml` preprocessing (separate file, separate registrar)
- Additional variable sources (`env`, custom SPI)
- Configurable expansion limit
- YAML schema validation tooling

## References

- `ClasspathYamlDescriptorRegistrar.java` — current YAML loading path
- `EidosDescriptorModule.java` — Jackson module with custom deserializers
- `AgentDescriptorDeserializer.java` — current descriptor deserialization
- `ForEachExpander.java` — yaml-core expansion engine
- `ForEachAdapter.java` — yaml-core adapter interface
- `VariableResolver.java` — yaml-core variable resolution
- `CsvParser.java` / `CsvDataSource.java` — yaml-core CSV support
- `ForEachExpanderTest.java` — yaml-core test patterns (TestAdapter reference implementation)
- casehubio/eidos#149 — feature issue
- casehubio/platform#248 — yaml-core module
- casehubio/quarkmind#283 — personality generator wizard (downstream motivation)
