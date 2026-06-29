# Design: Minor Fixes Batch — eidos#2, #3, #9

**Branch:** `issue-2-3-9-minor-fixes`
**Date:** 2026-05-25
**Issues:** casehubio/eidos#2, #3, #9

---

## eidos#2 — Null-safe comparisons in `InMemoryAgentRegistry`

### Problem

`InMemoryAgentRegistry.find()` has two comparisons that are not defensively null-safe:

- **Slot filter** (`query.slot().equals(d.slot())`): `String.equals(null)` is safe in Java, but
  the idiom `Objects.equals` is cleaner and makes intent explicit.
- **Capability filter** (`c.name().equals(query.capabilityName())`): if a registered
  `AgentCapability` has a null `name`, this throws NPE. `AgentCapabilityEntity` enforces
  `nullable=false` at the JPA layer, but `InMemoryAgentRegistry` has no such enforcement —
  a descriptor built in pure Java and registered directly can carry a null-named capability.

### Fix

Replace both comparisons with `Objects.equals`:

```java
.filter(d -> query.slot() == null || Objects.equals(d.slot(), query.slot()))
.filter(d -> query.capabilityName() == null
    || d.capabilities().stream().anyMatch(c -> Objects.equals(c.name(), query.capabilityName())))
```

### Tests (TDD — write first)

- `find_by_capability_with_null_slot_descriptor` — register a descriptor with `slot=null`,
  query by capability only (`AgentQuery.byCapability`), assert result contains the descriptor
  and no NPE is thrown.
- `find_by_slot_with_null_slot_descriptor` — register a descriptor with `slot=null`, query
  by a specific slot, assert the null-slot descriptor is excluded without NPE.

---

## eidos#3 — Phase 1 minor findings (5 fixes)

### Minor 1 — Entity field visibility comment

`AgentDescriptorEntity` and `AgentCapabilityEntity` are `public` with package-private fields.
This is intentional: Hibernate Reactive requires public class for bytecode enhancement;
package-private fields are the correct access level when enhancement handles field access
directly (no getters needed).

**Fix:** Add a class-level Javadoc comment to both entity classes explaining this invariant.

### Minor 2 — Reactive register threading template

`InMemoryReactiveAgentRegistry.register()` calls `delegate.register(descriptor)` synchronously
before wrapping in `Uni.createFrom().voidItem()`. This works for in-memory but establishes a
bad template — a future implementer adding real I/O would do it wrong by example.

**Fix:** Change to the idiomatic Mutiny pipeline:

```java
return Uni.createFrom().item(descriptor)
    .invoke(delegate::register)
    .replaceWithVoid();
```

No behaviour change for in-memory; safe template for I/O-bearing implementers.

### Minor 3 — `equivalentValues` Javadoc + test

`CdiVocabularyRegistry.equivalentValues()` collects matches across ALL terms that match
the input (by value or alias), not just the first. This can return multiple exact-match
values if multiple terms share the same alias pointing to different targets.

**Fix:**
1. Add Javadoc to `equivalentValues()` explaining multi-term semantics.
2. Add test: two distinct terms with the same alias, each with an exact-match value for the
   target vocabulary — assert both values are returned.

### Minor 4 — Test isolation in `InMemoryAgentRegistryTest`

Tests currently avoid pollution via unique agent IDs per test. This is fragile — a future test
that forgets to use a unique ID silently pollutes others. Explicit isolation is better.

**Fix:**
1. Add package-private `void clear()` to `InMemoryAgentRegistry` (clears the store map).
2. In `InMemoryAgentRegistryTest`: add a second injection `@Inject InMemoryAgentRegistry store`
   alongside the existing `@Inject AgentRegistry registry`, and add
   `@BeforeEach void clearStore() { store.clear(); }`.

### Minor 5 — Flyway consumer documentation

`quarkus-flyway` is `test`-scoped in `runtime/pom.xml`. Consumers who add `casehub-eidos`
must know to add `quarkus-flyway` and configure `quarkus.flyway.locations`. There is no
in-code signal pointing them to this requirement.

**Fix:** Add a comment block at the top of `V1__initial_schema.sql` documenting the required
consumer configuration.

---

## eidos#9 — Phase 3 renderer minor findings (3 fixes)

### Finding 1 — sha256 truncation documentation

`ClaudeMarkdownRenderer.sha256()` truncates the SHA-256 digest to 16 hex characters (64 bits)
via `substring(0, 16)`. The truncation is arbitrary-looking without context.

**Decision:** Keep 64-bit truncation — collision probability (1 in 2^64) is negligible for
cache-invalidation fingerprinting. The hash is compared for equality only; no cryptographic
trust is involved.

**Fix:** Add an inline comment above `substring(0, 16)` explaining why 64 bits is sufficient
for this use case.

### Finding 2 — Extract `renderAndCaptureYaml` test helper

Four tests in `ClaudeMarkdownRendererTest` share identical boilerplate (~14 lines each):
create a capturing `ChatModel` via anonymous class, construct a `ClaudeMarkdownRenderer`,
call `render()`, and return the captured YAML input string.

**Fix:** Extract `private String renderAndCaptureYaml(AgentDescriptor desc, AgentPromptContext ctx)`
that encapsulates the capture wiring and returns the captured prompt string. Each test body
becomes a single assertion line.

### Finding 3 — StubStateStore comment

`DefaultCapabilityHealthDegradedTest.StubStateStore.record()` accepts `expiresAt` but ignores
it. A reader encountering this wonders if TTL is silently broken.

**Fix:** Add `// TTL enforcement tested at the store level; intentionally ignored here` inside
`record()`.

---

## Scope

All changes are within `casehub-eidos`. No API changes, no cross-module impact, no Flyway
migration changes (the Minor 5 fix is a comment in an existing migration file). No PLATFORM.md
update needed.