# Cross-Vocabulary Subsumption Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Allow `VocabularyTerm.specializes()` to reference terms from different vocabularies, enabling application-tier capability hierarchies that extend foundation-tier vocabularies.

**Architecture:** Two-pass registration in `CdiVocabularyRegistry` — pass 1 registers terms, pass 2 builds a global DAG across all vocabularies and computes per-vocabulary ancestor/descendant indexes with cross-vocabulary injection. No API signature changes; existing consumers (`match()`, `subsumes()`, `expandForMatchingByVocabulary()`) work transparently.

**Tech Stack:** Java 21, Quarkus 3.32.2, AssertJ

## Global Constraints

- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- Test single module: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
- Issue: casehubio/eidos#73
- All commits reference `Refs casehubio/eidos#73`
- Use IntelliJ MCP for all code navigation — never bash grep/find for classes

---

### Task 1: Two-Pass Registration — Split `register()` into `registerTerms()` + `buildAllHierarchyIndexes()`

This task refactors `CdiVocabularyRegistry` to separate term registration from hierarchy computation. No cross-vocabulary behavior yet — this is a pure refactor that maintains identical behavior. The split is prerequisite for Task 2.

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/vocabulary/CdiVocabularyRegistry.java`
- Test: `runtime/src/test/java/io/casehub/eidos/runtime/vocabulary/CdiVocabularyRegistryTest.java`

**Interfaces:**
- Produces: `registerTerms(Class<T> vocab)` — validates and stores terms without building hierarchy
- Produces: `buildAllHierarchyIndexes(Map<String, Class<? extends Enum<?>>> vocabSnapshot)` — pure function, builds hierarchy from snapshot
- Produces: `AncestorEntry(VocabularyTerm term, int depth, String vocabUri)` — augmented record
- Produces: `DescendantEntry(VocabularyTerm term, int depth, String vocabUri)` — augmented record

- [ ] **Step 1: Verify existing tests pass before refactoring**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -f /Users/mdproctor/claude/casehub/eidos/pom.xml`
Expected: All tests PASS

- [ ] **Step 2: Add `vocabUri` field to `AncestorEntry` and `DescendantEntry`**

In `CdiVocabularyRegistry.java`, change the two private records (currently at lines 56-57):

```java
private record AncestorEntry(VocabularyTerm term, int depth, String vocabUri) {}
private record DescendantEntry(VocabularyTerm term, int depth, String vocabUri) {}
```

Update all construction sites in `buildHierarchyIndex()` to pass the vocabulary URI. Every `new AncestorEntry(term, depth)` becomes `new AncestorEntry(term, depth, uri)` where `uri` is the current vocabulary's URI (the `uri` parameter already in scope). Same for `DescendantEntry`.

- [ ] **Step 3: Extract `registerTerms()` from `register()`**

Create a new private method `registerTerms()` that contains the validation and map-population logic from `register()` lines 80-133 (everything up to and including the `byUri.put()` call), but NOT the `buildHierarchyIndex()` call. The method validates metadata, constants, duplicates, URI uniqueness, and populates `byUri`, `byClass`, `byClassOrdered`.

- [ ] **Step 4: Convert `buildHierarchyIndex()` to `buildAllHierarchyIndexes(vocabSnapshot)`**

Rename and refactor the existing `buildHierarchyIndex(vocab, uri, constants)` method. The new method:
1. Takes `Map<String, Class<? extends Enum<?>>> vocabSnapshot` (immutable snapshot of all registered vocabularies)
2. Iterates ALL vocabularies in the snapshot
3. For each vocabulary, builds edges, runs cycle detection, computes ancestor/descendant indexes — same logic as current `buildHierarchyIndex()` but iterated over the snapshot
4. Writes all indexes to class-level maps after all vocabularies are processed
5. Keep the same-vocabulary `specializes()` validation check for now (line 140 `if (!vocab.isInstance(parent))`) — Task 2 will remove it

The key difference from current code: cycle detection, BFS, and index writes happen ONCE for all vocabularies, not per-vocabulary during registration.

- [ ] **Step 5: Update `init()` and `register()` to use the two-pass flow**

```java
@PostConstruct
void init() {
    for (VocabularyRegistrar r : registrars) {
        registerTermsRaw(r.vocabulary());
    }
    buildAllHierarchyIndexes(Map.copyOf(byUri));
}
```

The public `register()` method calls `registerTerms()` then `buildAllHierarchyIndexes(Map.copyOf(byUri))`.

- [ ] **Step 6: Run all tests to verify the refactor is behavior-preserving**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -f /Users/mdproctor/claude/casehub/eidos/pom.xml`
Expected: All tests PASS — identical behavior, just restructured

- [ ] **Step 7: Run full build to catch any cross-module issues**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/eidos/pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 8: Commit**

```
feat(eidos#73): two-pass registration — split registerTerms() from buildAllHierarchyIndexes()

Refs casehubio/eidos#73
```

---

### Task 2: Cross-Vocabulary Subsumption — Global DAG with Injection

Remove the same-vocabulary constraint. Build the global DAG across vocabulary boundaries. Inject cross-vocabulary terms into per-vocabulary indexes for bidirectional matching. Implement inline collision detection. Implement late `register()` atomicity.

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/vocabulary/CdiVocabularyRegistry.java`
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/vocabulary/CdiVocabularyRegistryTest.java`

**Interfaces:**
- Consumes: `registerTerms()`, `buildAllHierarchyIndexes(vocabSnapshot)` from Task 1
- Produces: Cross-vocabulary subsumption working end-to-end via `match()`, `subsumes()`, `ancestors()`, `descendants()`

- [ ] **Step 1: Write failing test — cross-vocabulary `specializes()` edge accepted**

Replace the existing `register_cross_vocab_specializes_throws` test (line 618) with a test that asserts cross-vocabulary specialization SUCCEEDS. The `CrossRefTerm` enum already references `SourceTerm.ALPHA` via `specializes()` — flip the assertion:

```java
@Test
void register_cross_vocab_specializes_accepted() {
    @VocabularyMetadata(uri = "urn:test:cross-ref")
    enum CrossRefTerm implements VocabularyTerm {
        X("x", "X") {
            @Override public List<VocabularyTerm> specializes() {
                return List.of(SourceTerm.ALPHA);
            }
        };
        final String value, label;
        CrossRefTerm(String v, String l) { value = v; label = l; }
        @Override public String value() { return value; }
        @Override public String label() { return label; }
    }
    registry.register(SourceTerm.class);
    registry.register(CrossRefTerm.class);

    assertThat(registry.subsumes("urn:test:cross-ref", "alpha", "x")).isTrue();
    assertThat(registry.ancestors("urn:test:cross-ref", "x"))
        .extracting(VocabularyTerm::value).containsExactly("alpha");
}
```

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="CdiVocabularyRegistryTest#register_cross_vocab_specializes_accepted" -f /Users/mdproctor/claude/casehub/eidos/pom.xml`
Expected: FAIL — the same-vocabulary validation still rejects it

- [ ] **Step 2: Remove same-vocabulary constraint and implement global DAG**

In `buildAllHierarchyIndexes()`:
1. Remove the `if (!vocab.isInstance(parent))` validation check (current line ~140)
2. For cross-vocabulary parents, resolve the parent's vocabulary URI via `((Enum<?>) parent).getDeclaringClass()` → lookup in `vocabSnapshot`
3. Validate: if the parent's vocabulary is not in the snapshot, throw: `"Cross-vocabulary specializes() references unregistered vocabulary: <child> in <childVocab> specializes <parent> from <parentVocab>, but <parentVocab> is not registered"`
4. Build edge map, cycle detection, BFS ancestors/descendants across the GLOBAL DAG (all terms from all vocabularies)
5. When building per-vocabulary indexes, process vocabularies in sorted URI order (deterministic)
6. For each vocabulary V, inject terms from other vocabularies that have a transitive ancestor in V — with inline collision detection:
   - Native-vs-injected collision: key exists with a native entry → error
   - Injected-vs-injected collision: key exists with a previously injected entry → error
   - Error message: `"Value collision in index for '<vocabUri>': '<value>' from '<vocab1>' collides with '<value>' from '<vocab2>'"`
7. Write all indexes to class-level maps ONLY after all validation passes (compute-validate-write atomicity)

- [ ] **Step 3: Run the cross-vocab test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="CdiVocabularyRegistryTest#register_cross_vocab_specializes_accepted" -f /Users/mdproctor/claude/casehub/eidos/pom.xml`
Expected: PASS

- [ ] **Step 4: Write and run tests for cross-vocabulary `match()` — both directions**

```java
@Test
void match_cross_vocab_specialization() {
    registry.register(SourceTerm.class);
    @VocabularyMetadata(uri = "urn:test:app-cap")
    enum AppCap implements VocabularyTerm {
        SPECIAL("special", "Special") {
            @Override public List<VocabularyTerm> specializes() {
                return List.of(SourceTerm.ALPHA);
            }
        };
        final String value, label;
        AppCap(String v, String l) { value = v; label = l; }
        @Override public String value() { return value; }
        @Override public String label() { return label; }
    }
    registry.register(AppCap.class);

    // Specialization: declared is app-specific, requested is foundation
    assertThat(registry.match("urn:test:app-cap", "special", "alpha"))
        .isEqualTo(new MatchDegree.Specialization(1));

    // Plugin: declared is foundation, requested is app-specific (via injection)
    assertThat(registry.match("urn:test:source", "alpha", "special"))
        .isEqualTo(new MatchDegree.Plugin(1));
}
```

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="CdiVocabularyRegistryTest#match_cross_vocab_specialization" -f /Users/mdproctor/claude/casehub/eidos/pom.xml`
Expected: PASS

- [ ] **Step 5: Write and run tests for cross-vocabulary `subsumes()`, `ancestors()`, `descendants()`**

Tests covering: `subsumes()` across vocab boundaries, `ancestors()` returning cross-vocab ancestors, `descendants()` returning cross-vocab descendants, registration order independence.

- [ ] **Step 6: Write and run validation tests**

Tests covering: unregistered parent vocabulary fails, nonexistent parent term fails, cross-vocabulary cycle detection, native-vs-injected value collision, injected-vs-injected value collision, late `register()` failure atomicity (collision → exception → existing indexes unchanged).

- [ ] **Step 7: Run all runtime tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -f /Users/mdproctor/claude/casehub/eidos/pom.xml`
Expected: All tests PASS

- [ ] **Step 8: Commit**

```
feat(eidos#73): cross-vocabulary subsumption — global DAG with injection and collision detection

Refs casehubio/eidos#73
```

---

### Task 3: `expandForMatchingByVocabulary()` — Cross-Vocabulary Expansion

Update the expansion method to group cross-vocabulary entries by their declaring vocabulary URI.

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/vocabulary/CdiVocabularyRegistry.java`
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/vocabulary/CdiVocabularyRegistryTest.java`

**Interfaces:**
- Consumes: `AncestorEntry.vocabUri`, `DescendantEntry.vocabUri` from Task 1
- Produces: `expandForMatchingByVocabulary()` returns multi-vocabulary maps for cross-vocabulary hierarchies

- [ ] **Step 1: Write failing test — expansion returns cross-vocabulary terms under their declaring vocabulary URI**

```java
@Test
void expandForMatchingByVocabulary_cross_vocab_groups_by_declaring_vocab() {
    registry.register(HierarchyTerm.class);
    @VocabularyMetadata(uri = "urn:test:app-hierarchy")
    enum AppHierarchy implements VocabularyTerm {
        APP_CHILD("app-child", "App Child") {
            @Override public List<VocabularyTerm> specializes() {
                return List.of(HierarchyTerm.ROOT);
            }
        };
        final String value, label;
        AppHierarchy(String v, String l) { value = v; label = l; }
        @Override public String value() { return value; }
        @Override public String label() { return label; }
    }
    registry.register(AppHierarchy.class);

    // Expanding foundation term — should include app-tier descendant
    var expansion = registry.expandForMatchingByVocabulary("root");
    assertThat(expansion).containsKey("urn:test:hierarchy");
    assertThat(expansion).containsKey("urn:test:app-hierarchy");
    assertThat(expansion.get("urn:test:app-hierarchy")).contains("app-child");

    // Expanding app term — should include foundation ancestor
    var appExpansion = registry.expandForMatchingByVocabulary("app-child");
    assertThat(appExpansion).containsKey("urn:test:app-hierarchy");
    assertThat(appExpansion).containsKey("urn:test:hierarchy");
    assertThat(appExpansion.get("urn:test:hierarchy")).contains("root");
}
```

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="CdiVocabularyRegistryTest#expandForMatchingByVocabulary_cross_vocab_groups_by_declaring_vocab" -f /Users/mdproctor/claude/casehub/eidos/pom.xml`
Expected: FAIL — current implementation groups all expanded terms under the source vocabulary URI

- [ ] **Step 2: Implement cross-vocabulary grouping in `expandForMatchingByVocabulary()`**

Replace the current implementation. The new logic:
1. Start from `valueToVocabs.get(value)` (declaring vocabularies only)
2. For each vocabulary, iterate ancestor and descendant entries from the augmented indexes
3. Group each entry into the result map by its `entry.vocabUri()` field, NOT by the vocabulary whose index it was read from
4. The self-value goes under each vocabulary it's declared in (from `valueToVocabs`)

- [ ] **Step 3: Run the cross-vocab expansion test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="CdiVocabularyRegistryTest#expandForMatchingByVocabulary_cross_vocab_groups_by_declaring_vocab" -f /Users/mdproctor/claude/casehub/eidos/pom.xml`
Expected: PASS

- [ ] **Step 4: Write three-vocabulary chain expansion test**

Test the Foundation → Mid → App transitive chain from the spec examples.

- [ ] **Step 5: Run all runtime tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -f /Users/mdproctor/claude/casehub/eidos/pom.xml`
Expected: All tests PASS

- [ ] **Step 6: Commit**

```
feat(eidos#73): expandForMatchingByVocabulary cross-vocabulary grouping by declaring vocabulary

Refs casehubio/eidos#73
```

---

### Task 4: Integration Tests and Full Build Verification

Add integration tests using real `CasehubCapabilityTerm` and JPA registry, and verify end-to-end with the full build.

**Files:**
- Modify: `examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/CapabilityVocabularyIntegrationTest.java`
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/registry/JpaAgentRegistryTest.java`
- Test: existing test infrastructure

**Interfaces:**
- Consumes: All cross-vocabulary subsumption functionality from Tasks 1-3

- [ ] **Step 1: Add cross-vocabulary integration test to `CapabilityVocabularyIntegrationTest`**

Create a test vocabulary in the integration test module that specializes `CasehubCapabilityTerm.DOCUMENTATION`. Register it and verify `subsumes()`, `match()`, `expandForMatchingByVocabulary()` work cross-vocabulary with the real foundation vocabulary.

Note: This test uses `CasehubCapabilityTerm` which is registered by the `CasehubCapabilityVocabularyRegistrar` CDI bean at startup. The test vocabulary must be registered programmatically.

- [ ] **Step 2: Run integration tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl examples/agent-scenarios -f /Users/mdproctor/claude/casehub/eidos/pom.xml`
Expected: All tests PASS

- [ ] **Step 3: Add JPA registry cross-vocabulary test to `JpaAgentRegistryTest`**

Register an agent with a foundation capability (`documentation` in `urn:casehub:vocab:capability`) and query for an app-tier term — verify the JPA `find(AgentQuery)` path matches via Plugin. Register an agent with an app-tier capability and query for a foundation term — verify Specialization match. This verifies the JPA query path (which uses `expandForMatchingByVocabulary()`) produces the same results as the `InMemoryAgentRegistry` path (which uses `match()` per-agent).

- [ ] **Step 4: Run JPA registry tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="JpaAgentRegistryTest" -f /Users/mdproctor/claude/casehub/eidos/pom.xml`
Expected: All tests PASS

- [ ] **Step 5: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/eidos/pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```
test(eidos#73): cross-vocabulary integration tests — CapabilityVocabularyIntegrationTest + JpaAgentRegistryTest

Refs casehubio/eidos#73
```
