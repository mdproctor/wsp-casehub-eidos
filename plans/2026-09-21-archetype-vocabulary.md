# Archetype Vocabulary Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #180 — Archetype vocabulary: 60-archetype personality system with faceted framework intersection
**Issue group:** #180

**Goal:** Add a 60-archetype vocabulary derived from Hartwell & Chen that sits above existing personality frameworks, providing human/LLM-readable personality labels with a faceted intersection algorithm.

**Architecture:** New `ArchetypeTerm` and `ArchetypeFamily` enums in casehub-eidos-vocab. `ArchetypeResolver` utility in api/ provides the set-intersection algorithm. `AgentDescriptor` gains optional archetype fields. The renderer surfaces archetype identity as the primary personality section, with source vocabulary descriptions for all frameworks (fixing the Jungian-only rendering gap).

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-eidos-vocab, casehub-eidos-api, casehub-eidos runtime

## Global Constraints

- Java 21 source (Java 26 JVM)
- All new public API goes in `casehub-eidos-api` (Tier 1, pure Java — no CDI, no Quarkus)
- Vocabulary enums go in `casehub-eidos-vocab`
- Renderer changes in `runtime/` (`io.casehub.eidos.core.renderer`)
- No existing installations — schema changes go directly in base migrations
- Each vocabulary enum implements `VocabularyTerm` and is annotated with `@VocabularyMetadata`
- Tests use `@QuarkusTest` for CDI-dependent code, plain JUnit for pure Java
- Protocol PP-20260611-228599: numeric routing signals in A2A_CARD only, not MARKDOWN/PROSE

---

## Batch 1: Archetype Vocabulary Foundation

### Task 1: ArchetypeFamily enum and ArchetypeTerm enum

**Files:**
- Create: `vocab/src/main/java/io/casehub/eidos/vocab/ArchetypeFamily.java`
- Create: `vocab/src/main/java/io/casehub/eidos/vocab/ArchetypeTerm.java`
- Modify: `vocab/src/main/java/io/casehub/eidos/vocab/EidosVocabRegistrar.java` — add archetype registration
- Test: `vocab/src/test/java/io/casehub/eidos/vocab/ArchetypeTermTest.java`

**Interfaces:**
- Produces: `ArchetypeFamily` enum (12 constants: SAGE, HERO, CREATOR, EXPLORER, SOVEREIGN, MAGICIAN, CAREGIVER, INNOCENT, JESTER, REBEL, LOVER, EVERYMAN), each with `quadrant()`, `drive()`, `label()`
- Produces: `ArchetypeTerm` enum (60 constants), each with `family()`, `value()`, `label()`, `description()`, `validAdjectives()`, `invalidAdjectives()`. Implements `VocabularyTerm`.

- [ ] **Step 1: Write the failing test for ArchetypeFamily**

```java
package io.casehub.eidos.vocab;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class ArchetypeTermTest {

    @Test
    void familyCountIs12() {
        assertEquals(12, ArchetypeFamily.values().length);
    }

    @Test
    void eachFamilyHasQuadrant() {
        for (ArchetypeFamily f : ArchetypeFamily.values()) {
            assertNotNull(f.quadrant(), f.name() + " missing quadrant");
            assertNotNull(f.drive(), f.name() + " missing drive");
        }
    }

    @Test
    void sageFamilyHasIndependenceQuadrant() {
        assertEquals(ArchetypeFamily.Quadrant.INDEPENDENCE, ArchetypeFamily.SAGE.quadrant());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl vocab -Dtest=ArchetypeTermTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: Compilation error — ArchetypeFamily not found

- [ ] **Step 3: Implement ArchetypeFamily**

```java
package io.casehub.eidos.vocab;

public enum ArchetypeFamily {
    CAREGIVER(Quadrant.STABILITY, "Service, compassion, generosity"),
    EVERYMAN(Quadrant.BELONGING, "Belonging, empathy, realism"),
    CREATOR(Quadrant.STABILITY, "Innovation, expression, imagination"),
    INNOCENT(Quadrant.INDEPENDENCE, "Safety, optimism, simplicity"),
    EXPLORER(Quadrant.INDEPENDENCE, "Freedom, discovery, self-sufficiency"),
    HERO(Quadrant.MASTERY, "Mastery, courage, achievement"),
    JESTER(Quadrant.BELONGING, "Joy, spontaneity, humor"),
    LOVER(Quadrant.BELONGING, "Intimacy, passion, commitment"),
    MAGICIAN(Quadrant.MASTERY, "Transformation, vision, catalyst"),
    REBEL(Quadrant.MASTERY, "Liberation, disruption, revolution"),
    SAGE(Quadrant.INDEPENDENCE, "Knowledge, truth, understanding"),
    SOVEREIGN(Quadrant.STABILITY, "Control, order, leadership");

    public enum Quadrant {
        INDEPENDENCE, MASTERY, BELONGING, STABILITY
    }

    private final Quadrant quadrant;
    private final String drive;

    ArchetypeFamily(Quadrant quadrant, String drive) {
        this.quadrant = quadrant;
        this.drive = drive;
    }

    public Quadrant quadrant() { return quadrant; }
    public String drive() { return drive; }
    public String label() { return name().charAt(0) + name().substring(1).toLowerCase(); }
}
```

- [ ] **Step 4: Write ArchetypeTerm failing tests**

```java
@Test
void termCountIs60() {
    // 12 families × 4-5 sub-archetypes = ~48-60
    assertTrue(ArchetypeTerm.values().length >= 48);
    assertTrue(ArchetypeTerm.values().length <= 60);
}

@Test
void detectiveBelongsToSageFamily() {
    assertEquals(ArchetypeFamily.SAGE, ArchetypeTerm.DETECTIVE.family());
    assertEquals("detective", ArchetypeTerm.DETECTIVE.value());
    assertEquals("Detective", ArchetypeTerm.DETECTIVE.label());
    assertFalse(ArchetypeTerm.DETECTIVE.description().isBlank());
}

@Test
void eachTermHasValidAdjectives() {
    for (ArchetypeTerm t : ArchetypeTerm.values()) {
        assertFalse(t.validAdjectives().isEmpty(),
            t.name() + " has no valid adjectives");
        assertFalse(t.invalidAdjectives().isEmpty(),
            t.name() + " has no invalid adjectives");
    }
}

@Test
void validAndInvalidAdjectivesDoNotOverlap() {
    for (ArchetypeTerm t : ArchetypeTerm.values()) {
        var overlap = new java.util.HashSet<>(t.validAdjectives());
        overlap.retainAll(t.invalidAdjectives());
        assertTrue(overlap.isEmpty(),
            t.name() + " has overlapping adjectives: " + overlap);
    }
}

@Test
void vocabularyMetadataPresent() {
    var meta = ArchetypeTerm.class.getAnnotation(
        io.casehub.eidos.api.VocabularyMetadata.class);
    assertNotNull(meta);
    assertEquals("urn:casehub:vocab:archetype", meta.uri());
}

@Test
void implementsVocabularyTerm() {
    assertTrue(io.casehub.eidos.api.VocabularyTerm.class
        .isAssignableFrom(ArchetypeTerm.class));
}
```

- [ ] **Step 5: Implement ArchetypeTerm**

Create the enum with all 60 constants. Follow the BelbinTerm pattern. Each constant has: value, label, description, family, validAdjectives, invalidAdjectives.

Use the data from `specs/archetype-compatibility-matrix.md` (Section 2) and `specs/avatar-generator-contract.md` (Section 4 adjective catalog) as the source of truth for all 60 terms.

Pattern for each constant (showing 4 from Sage family):

```java
package io.casehub.eidos.vocab;

import io.casehub.eidos.api.VocabularyMetadata;
import io.casehub.eidos.api.VocabularyTerm;
import java.util.List;

@VocabularyMetadata(uri = "urn:casehub:vocab:archetype",
                    name = "Hartwell & Chen Archetypes", version = "1.0",
                    description = "60 sub-archetypes in 12 families from Archetypes in Branding (2012). Four motivation quadrants: Independence, Mastery, Belonging, Stability.")
public enum ArchetypeTerm implements VocabularyTerm {

    // --- Sage family (Independence) ---
    DETECTIVE("detective", "Detective",
        "Uncovers truth through systematic investigation and evidence",
        ArchetypeFamily.SAGE,
        List.of("analytical", "meticulous", "persistent", "methodical", "observant", "evidence-driven", "skeptical"),
        List.of("impulsive", "gullible", "reckless", "superficial")),

    MENTOR("mentor", "Mentor",
        "Shares accumulated wisdom to develop others' potential",
        ArchetypeFamily.SAGE,
        List.of("patient", "wise", "developmental", "guiding", "experienced", "nurturing", "insightful"),
        List.of("dismissive", "withholding", "impatient", "competitive")),

    SHAMAN("shaman", "Shaman",
        "Accesses insight from unconventional or unseen sources",
        ArchetypeFamily.SAGE,
        List.of("intuitive", "mystical", "deep", "liminal", "unconventional", "contemplative", "perceptive"),
        List.of("superficial", "conventional", "rigid", "literal")),

    TRANSLATOR("translator", "Translator",
        "Makes the complex accessible; bridges disciplines and audiences",
        ArchetypeFamily.SAGE,
        List.of("clear", "synthesizing", "accessible", "bridging", "articulate", "patient", "versatile"),
        List.of("obscure", "jargon-heavy", "narrow", "exclusive")),

    // --- Hero family (Mastery) ---
    ATHLETE("athlete", "Athlete", ...),
    // ... remaining 56 constants following same pattern ...
    ;

    public static final String URI = "urn:casehub:vocab:archetype";

    private final String value;
    private final String label;
    private final String description;
    private final ArchetypeFamily family;
    private final List<String> validAdj;
    private final List<String> invalidAdj;

    ArchetypeTerm(String value, String label, String description,
                  ArchetypeFamily family,
                  List<String> validAdj, List<String> invalidAdj) {
        this.value = value;
        this.label = label;
        this.description = description;
        this.family = family;
        this.validAdj = validAdj;
        this.invalidAdj = invalidAdj;
    }

    @Override public String value() { return value; }
    @Override public String label() { return label; }
    @Override public String description() { return description; }
    public ArchetypeFamily family() { return family; }
    public List<String> validAdjectives() { return validAdj; }
    public List<String> invalidAdjectives() { return invalidAdj; }

    public static List<ArchetypeTerm> byFamily(ArchetypeFamily family) {
        return java.util.Arrays.stream(values())
            .filter(t -> t.family == family)
            .toList();
    }
}
```

Populate all 60 constants from the specs. Group by family with comment separators.

- [ ] **Step 6: Register archetype vocabulary**

Add to `EidosVocabRegistrar` (or create a new registrar bean if one doesn't exist per vocab):

```java
registry.register(ArchetypeTerm.class);
```

- [ ] **Step 7: Run tests and verify**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl vocab -Dtest=ArchetypeTermTest`
Expected: All pass

- [ ] **Step 8: Commit**

```bash
git -C $PROJECT add vocab/
git -C $PROJECT commit -m "feat(vocab): add ArchetypeFamily and ArchetypeTerm — 60 archetypes in 12 families

Hartwell & Chen archetype model from Archetypes in Branding (2012).
12 families grouped by 4 motivation quadrants (Independence, Mastery,
Belonging, Stability). Each term carries valid/invalid adjectives for
faceted personality refinement.

Refs #180"
```

---

## Batch 2: Compatibility Data & Intersection Algorithm

### Task 2: ArchetypeCompatibility mapping data

**Files:**
- Create: `api/src/main/java/io/casehub/eidos/api/ArchetypeCompatibility.java`
- Test: `api/src/test/java/io/casehub/eidos/api/ArchetypeCompatibilityTest.java`

**Interfaces:**
- Consumes: `ArchetypeFamily`, `ArchetypeTerm` from vocab module
- Produces: `compatibleFamilies(String frameworkUri, String termValue) → Set<ArchetypeFamily>`, `subArchetypeAffinity(ArchetypeTerm, String frameworkUri, String termValue) → boolean`

- [ ] **Step 1: Write failing tests**

```java
package io.casehub.eidos.api;

import io.casehub.eidos.vocab.ArchetypeFamily;
import io.casehub.eidos.vocab.ArchetypeTerm;
import org.junit.jupiter.api.Test;
import java.util.Set;
import static org.junit.jupiter.api.Assertions.*;

class ArchetypeCompatibilityTest {

    static final String MBTI_URI = "urn:casehub:vocab:mbti";
    static final String ENNEAGRAM_URI = "urn:casehub:vocab:enneagram";
    static final String DISC_URI = "urn:casehub:vocab:disc";
    static final String BELBIN_URI = "urn:casehub:vocab:belbin";

    @Test
    void intjMapsToSageMagicianSovereign() {
        Set<ArchetypeFamily> families = ArchetypeCompatibility
            .compatibleFamilies(MBTI_URI, "intj");
        assertTrue(families.contains(ArchetypeFamily.SAGE));
        assertTrue(families.contains(ArchetypeFamily.MAGICIAN));
        assertTrue(families.contains(ArchetypeFamily.SOVEREIGN));
        assertEquals(3, families.size());
    }

    @Test
    void enneagram5MapsToSageExplorerMagician() {
        Set<ArchetypeFamily> families = ArchetypeCompatibility
            .compatibleFamilies(ENNEAGRAM_URI, "5");
        assertTrue(families.contains(ArchetypeFamily.SAGE));
        assertTrue(families.contains(ArchetypeFamily.EXPLORER));
        assertTrue(families.contains(ArchetypeFamily.MAGICIAN));
    }

    @Test
    void discCMapsToSageSovereignCreatorExplorer() {
        Set<ArchetypeFamily> families = ArchetypeCompatibility
            .compatibleFamilies(DISC_URI, "conscientiousness");
        assertTrue(families.contains(ArchetypeFamily.SAGE));
        assertTrue(families.contains(ArchetypeFamily.SOVEREIGN));
    }

    @Test
    void belbinPlantMapsToCreatorMagician() {
        Set<ArchetypeFamily> families = ArchetypeCompatibility
            .compatibleFamilies(BELBIN_URI, "plant");
        assertTrue(families.contains(ArchetypeFamily.CREATOR));
        assertTrue(families.contains(ArchetypeFamily.MAGICIAN));
    }

    @Test
    void detectiveHasIntjAffinity() {
        assertTrue(ArchetypeCompatibility
            .subArchetypeAffinity(ArchetypeTerm.DETECTIVE, MBTI_URI, "intj"));
    }

    @Test
    void detectiveDoesNotHaveEnfpAffinity() {
        assertFalse(ArchetypeCompatibility
            .subArchetypeAffinity(ArchetypeTerm.DETECTIVE, MBTI_URI, "enfp"));
    }

    @Test
    void unknownFrameworkReturnsAllFamilies() {
        Set<ArchetypeFamily> families = ArchetypeCompatibility
            .compatibleFamilies("urn:casehub:vocab:unknown", "x");
        assertEquals(12, families.size());
    }
}
```

- [ ] **Step 2: Implement ArchetypeCompatibility**

Static utility class in api/. Contains the mapping data from `specs/archetype-compatibility-matrix.md` Section 1 as static `Map<String, Map<String, Set<ArchetypeFamily>>>` keyed by framework URI → term value → compatible families. Sub-archetype affinity from Section 2 as `Map<ArchetypeTerm, Map<String, Set<String>>>` keyed by archetype → framework URI → affinity values.

```java
package io.casehub.eidos.api;

import io.casehub.eidos.vocab.ArchetypeFamily;
import io.casehub.eidos.vocab.ArchetypeTerm;
import java.util.*;

public final class ArchetypeCompatibility {

    private static final Map<String, Map<String, Set<ArchetypeFamily>>> FAMILY_MAP = new HashMap<>();
    private static final Map<ArchetypeTerm, Map<String, Set<String>>> AFFINITY_MAP = new HashMap<>();
    private static final Set<ArchetypeFamily> ALL_FAMILIES = Set.of(ArchetypeFamily.values());

    static {
        // MBTI mappings
        var mbti = new HashMap<String, Set<ArchetypeFamily>>();
        mbti.put("intj", Set.of(ArchetypeFamily.SAGE, ArchetypeFamily.MAGICIAN, ArchetypeFamily.SOVEREIGN));
        mbti.put("intp", Set.of(ArchetypeFamily.SAGE, ArchetypeFamily.EXPLORER, ArchetypeFamily.CREATOR));
        mbti.put("entj", Set.of(ArchetypeFamily.SOVEREIGN, ArchetypeFamily.HERO, ArchetypeFamily.MAGICIAN));
        mbti.put("entp", Set.of(ArchetypeFamily.MAGICIAN, ArchetypeFamily.REBEL, ArchetypeFamily.EXPLORER));
        // ... all 16 MBTI types from compatibility matrix ...
        FAMILY_MAP.put(MbtiTypeTerm.URI, mbti);

        // Enneagram, DISC, Belbin, Big Five, SDI — same pattern
        // ... populated from specs/archetype-compatibility-matrix.md ...

        // Sub-archetype affinities
        var detectiveAff = new HashMap<String, Set<String>>();
        detectiveAff.put(MbtiTypeTerm.URI, Set.of("istj", "intj", "istp"));
        detectiveAff.put("urn:casehub:vocab:enneagram", Set.of("5", "6"));
        AFFINITY_MAP.put(ArchetypeTerm.DETECTIVE, detectiveAff);
        // ... all 60 sub-archetypes from compatibility matrix Section 2 ...
    }

    private ArchetypeCompatibility() {}

    public static Set<ArchetypeFamily> compatibleFamilies(String frameworkUri, String termValue) {
        var framework = FAMILY_MAP.get(frameworkUri);
        if (framework == null) return ALL_FAMILIES;
        var families = framework.get(termValue.toLowerCase(Locale.ROOT));
        return families != null ? families : ALL_FAMILIES;
    }

    public static boolean subArchetypeAffinity(ArchetypeTerm archetype,
                                                String frameworkUri, String termValue) {
        var affinities = AFFINITY_MAP.get(archetype);
        if (affinities == null) return false;
        var values = affinities.get(frameworkUri);
        return values != null && values.contains(termValue.toLowerCase(Locale.ROOT));
    }
}
```

Populate all mappings from `specs/archetype-compatibility-matrix.md`.

- [ ] **Step 3: Run tests, verify pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=ArchetypeCompatibilityTest`

- [ ] **Step 4: Commit**

```bash
git -C $PROJECT add api/
git -C $PROJECT commit -m "feat(api): add ArchetypeCompatibility — framework-to-archetype mapping data

Static mapping tables for MBTI, Enneagram, DISC, Belbin, Big Five, SDI
to archetype families and sub-archetype affinities. Powers the set-
intersection algorithm.

Refs #180"
```

### Task 3: ArchetypeResolver — intersection algorithm

**Files:**
- Create: `api/src/main/java/io/casehub/eidos/api/ArchetypeResolver.java`
- Test: `api/src/test/java/io/casehub/eidos/api/ArchetypeResolverTest.java`

**Interfaces:**
- Consumes: `ArchetypeCompatibility.compatibleFamilies()`, `ArchetypeCompatibility.subArchetypeAffinity()`
- Produces: `resolve(Map<String, String> frameworkValues) → ArchetypeResolution` — sealed interface: `Converged(ArchetypeTerm)`, `Narrowed(List<ArchetypeTerm> candidates)`, `Conflict(String frameworkA, String frameworkB)`

- [ ] **Step 1: Write failing tests**

```java
package io.casehub.eidos.api;

import io.casehub.eidos.vocab.ArchetypeTerm;
import org.junit.jupiter.api.Test;
import java.util.Map;
import static org.junit.jupiter.api.Assertions.*;

class ArchetypeResolverTest {

    @Test
    void intjAndEnneagram5NarrowsToSageOrMagician() {
        var result = ArchetypeResolver.resolve(Map.of(
            "urn:casehub:vocab:mbti", "intj",
            "urn:casehub:vocab:enneagram", "5"));
        assertInstanceOf(ArchetypeResolver.Narrowed.class, result);
        var narrowed = (ArchetypeResolver.Narrowed) result;
        assertTrue(narrowed.candidates().stream()
            .anyMatch(t -> t == ArchetypeTerm.DETECTIVE));
    }

    @Test
    void intjEnneagram5DiscCConvergesToSageFamily() {
        var result = ArchetypeResolver.resolve(Map.of(
            "urn:casehub:vocab:mbti", "intj",
            "urn:casehub:vocab:enneagram", "5",
            "urn:casehub:vocab:disc", "conscientiousness"));
        assertInstanceOf(ArchetypeResolver.Narrowed.class, result);
        var narrowed = (ArchetypeResolver.Narrowed) result;
        assertTrue(narrowed.candidates().stream()
            .allMatch(t -> t.family() == io.casehub.eidos.vocab.ArchetypeFamily.SAGE));
    }

    @Test
    void isfjAndEnneagram8Conflicts() {
        var result = ArchetypeResolver.resolve(Map.of(
            "urn:casehub:vocab:mbti", "isfj",
            "urn:casehub:vocab:enneagram", "8"));
        assertInstanceOf(ArchetypeResolver.Conflict.class, result);
    }

    @Test
    void singleFrameworkNarrows() {
        var result = ArchetypeResolver.resolve(Map.of(
            "urn:casehub:vocab:mbti", "estp"));
        assertInstanceOf(ArchetypeResolver.Narrowed.class, result);
        var narrowed = (ArchetypeResolver.Narrowed) result;
        assertTrue(narrowed.candidates().size() < 60);
    }

    @Test
    void emptyInputReturnsAll60() {
        var result = ArchetypeResolver.resolve(Map.of());
        assertInstanceOf(ArchetypeResolver.Narrowed.class, result);
        assertEquals(60, ((ArchetypeResolver.Narrowed) result).candidates().size());
    }
}
```

- [ ] **Step 2: Implement ArchetypeResolver**

```java
package io.casehub.eidos.api;

import io.casehub.eidos.vocab.ArchetypeFamily;
import io.casehub.eidos.vocab.ArchetypeTerm;
import java.util.*;
import java.util.stream.Collectors;

public final class ArchetypeResolver {

    public sealed interface Resolution permits Converged, Narrowed, Conflict {}
    public record Converged(ArchetypeTerm archetype) implements Resolution {}
    public record Narrowed(List<ArchetypeTerm> candidates) implements Resolution {}
    public record Conflict(String frameworkUriA, String termValueA,
                           String frameworkUriB, String termValueB) implements Resolution {}

    private ArchetypeResolver() {}

    public static Resolution resolve(Map<String, String> frameworkValues) {
        if (frameworkValues.isEmpty()) {
            return new Narrowed(List.of(ArchetypeTerm.values()));
        }

        Set<ArchetypeFamily> candidateFamilies = EnumSet.allOf(ArchetypeFamily.class);

        for (var entry : frameworkValues.entrySet()) {
            var families = ArchetypeCompatibility.compatibleFamilies(
                entry.getKey(), entry.getValue());
            var before = Set.copyOf(candidateFamilies);
            candidateFamilies.retainAll(families);

            if (candidateFamilies.isEmpty()) {
                return findConflictingPair(frameworkValues);
            }
        }

        List<ArchetypeTerm> candidates = Arrays.stream(ArchetypeTerm.values())
            .filter(t -> candidateFamilies.contains(t.family()))
            .toList();

        if (candidates.size() <= 1) {
            return candidates.isEmpty()
                ? findConflictingPair(frameworkValues)
                : new Converged(candidates.get(0));
        }

        // Sub-archetype affinity refinement
        List<ArchetypeTerm> affinityMatched = candidates.stream()
            .filter(t -> frameworkValues.entrySet().stream()
                .anyMatch(e -> ArchetypeCompatibility
                    .subArchetypeAffinity(t, e.getKey(), e.getValue())))
            .toList();

        if (affinityMatched.size() == 1) {
            return new Converged(affinityMatched.get(0));
        }
        if (!affinityMatched.isEmpty()) {
            return new Narrowed(affinityMatched);
        }

        return new Narrowed(candidates);
    }

    private static Conflict findConflictingPair(Map<String, String> frameworkValues) {
        var entries = new ArrayList<>(frameworkValues.entrySet());
        for (int i = 0; i < entries.size(); i++) {
            for (int j = i + 1; j < entries.size(); j++) {
                var a = ArchetypeCompatibility.compatibleFamilies(
                    entries.get(i).getKey(), entries.get(i).getValue());
                var b = ArchetypeCompatibility.compatibleFamilies(
                    entries.get(j).getKey(), entries.get(j).getValue());
                var intersection = new HashSet<>(a);
                intersection.retainAll(b);
                if (intersection.isEmpty()) {
                    return new Conflict(
                        entries.get(i).getKey(), entries.get(i).getValue(),
                        entries.get(j).getKey(), entries.get(j).getValue());
                }
            }
        }
        return new Conflict(entries.get(0).getKey(), entries.get(0).getValue(),
                           entries.get(1).getKey(), entries.get(1).getValue());
    }
}
```

- [ ] **Step 3: Run tests, verify pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=ArchetypeResolverTest`

- [ ] **Step 4: Commit**

```bash
git -C $PROJECT add api/
git -C $PROJECT commit -m "feat(api): add ArchetypeResolver — set-intersection archetype derivation

Given framework values (MBTI, Enneagram, DISC, etc.), intersects
compatible archetype families and refines to sub-archetypes via
affinity matching. Returns Converged, Narrowed, or Conflict.

Refs #180"
```

---

## Batch 3: Descriptor Integration

### Task 4: Add archetype fields to AgentDescriptor and YAML support

**Files:**
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentDescriptor.java` — add `archetype`, `archetypeAdjectives` fields
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/yaml/AgentDescriptorDeserializer.java` — parse archetype from YAML
- Modify: `runtime/src/main/java/io/casehub/eidos/core/registrar/DescriptorCollector.java` — derive archetype from framework values when not explicitly set
- Test: `api/src/test/java/io/casehub/eidos/api/AgentDescriptorArchetypeTest.java`
- Test: `runtime/src/test/java/io/casehub/eidos/runtime/yaml/ArchetypeYamlTest.java`

**Interfaces:**
- Consumes: `ArchetypeTerm`, `ArchetypeResolver`
- Produces: `AgentDescriptor.archetype()` → `Optional<ArchetypeTerm>`, `AgentDescriptor.archetypeAdjectives()` → `List<String>`

- [ ] **Step 1: Write failing test for AgentDescriptor archetype field**

```java
package io.casehub.eidos.api;

import io.casehub.eidos.vocab.ArchetypeTerm;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class AgentDescriptorArchetypeTest {

    @Test
    void archetypeFieldDefaultsToEmpty() {
        var d = AgentDescriptor.builder()
            .agentId("test").name("Test").tenancyId("t1")
            .build();
        assertTrue(d.archetype().isEmpty());
        assertTrue(d.archetypeAdjectives().isEmpty());
    }

    @Test
    void archetypeCanBeSet() {
        var d = AgentDescriptor.builder()
            .agentId("test").name("Test").tenancyId("t1")
            .archetype(ArchetypeTerm.DETECTIVE)
            .archetypeAdjectives(java.util.List.of("meticulous", "persistent"))
            .build();
        assertEquals(ArchetypeTerm.DETECTIVE, d.archetype().orElseThrow());
        assertEquals(java.util.List.of("meticulous", "persistent"), d.archetypeAdjectives());
    }

    @Test
    void invalidAdjectiveRejected() {
        assertThrows(IllegalArgumentException.class, () ->
            AgentDescriptor.builder()
                .agentId("test").name("Test").tenancyId("t1")
                .archetype(ArchetypeTerm.DETECTIVE)
                .archetypeAdjectives(java.util.List.of("impulsive"))
                .build());
    }
}
```

- [ ] **Step 2: Add archetype fields to AgentDescriptor**

Add to AgentDescriptor record: `ArchetypeTerm archetype` (nullable), `List<String> archetypeAdjectives`. Add builder methods. Validate adjectives against `archetype.validAdjectives()` in compact constructor when archetype is non-null. Use `ide_insert_member` to add the fields and builder methods.

- [ ] **Step 3: Write failing YAML deserialization test**

```java
package io.casehub.eidos.runtime.yaml;

// @QuarkusTest with descriptors.yaml containing:
// archetype: detective
// archetypeAdjectives: [meticulous, persistent]
```

- [ ] **Step 4: Update AgentDescriptorDeserializer for archetype fields**

Parse `archetype` as a string, resolve to `ArchetypeTerm` by value. Parse `archetypeAdjectives` as a list of strings.

- [ ] **Step 5: Add auto-derivation in DescriptorCollector**

When `archetype` is null but framework values are present (dispositionVocabulary, dispositionProfile, slotVocabulary), call `ArchetypeResolver.resolve()` with the framework values. If result is `Converged`, set the archetype. If `Narrowed` with single-family candidates, set the archetype to the first affinity match. Otherwise leave null.

- [ ] **Step 6: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api,runtime`

- [ ] **Step 7: Commit**

```bash
git -C $PROJECT add api/ runtime/
git -C $PROJECT commit -m "feat: add archetype field to AgentDescriptor with YAML and auto-derivation

AgentDescriptor gains optional archetype (ArchetypeTerm) and
archetypeAdjectives (List<String>). Adjectives validated against
archetype's valid set. YAML supports 'archetype: detective'.
DescriptorCollector auto-derives archetype from framework values
when not explicitly set.

Refs #180"
```

---

## Batch 4: Rendering — Archetype Identity + Source Vocabulary Fix

### Task 5: Render archetype in MARKDOWN/PROSE and fix source vocabulary rendering

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/core/renderer/EidosRenderPipeline.java` — add archetype rendering section, generalize cognitive profile beyond Jungian
- Test: `runtime/src/test/java/io/casehub/eidos/runtime/renderer/EidosRenderPipelineTest.java` — add archetype rendering tests

**Interfaces:**
- Consumes: `AgentDescriptor.archetype()`, `AgentDescriptor.archetypeAdjectives()`, `ArchetypeTerm.family()`, `ArchetypeTerm.description()`, `VocabularyRegistry` for source vocabulary resolution

- [ ] **Step 1: Write failing test for archetype rendering in MARKDOWN**

```java
@Test
void markdownRendersArchetypeSection() {
    var descriptor = AgentDescriptor.builder()
        .agentId("test").name("Test Agent").tenancyId("t1")
        .archetype(ArchetypeTerm.DETECTIVE)
        .archetypeAdjectives(List.of("meticulous", "persistent"))
        .build();

    var rendered = renderer.render(descriptor,
        AgentPromptContext.of(RenderFormat.MARKDOWN));

    var text = rendered.text();
    assertTrue(text.contains("Detective"), "Should contain archetype name");
    assertTrue(text.contains("Sage"), "Should contain family name");
    assertTrue(text.contains("meticulous"), "Should contain adjectives");
    assertTrue(text.contains("Uncovers truth"), "Should contain description");
}
```

- [ ] **Step 2: Add assembleArchetypeSection() to EidosRenderPipeline**

Insert a new method that renders the archetype section before the disposition section. Use `ide_insert_member`.

```java
private void assembleArchetypeSection(AgentDescriptor d, StringBuilder sb) {
    d.archetype().ifPresent(archetype -> {
        sb.append("\n## Personality\n\n");
        sb.append("**").append(archetype.label()).append("** (")
          .append(archetype.family().label()).append(" family)");
        if (!d.archetypeAdjectives().isEmpty()) {
            sb.append(" — ").append(String.join(", ", d.archetypeAdjectives()));
        }
        sb.append("\n\n");
        sb.append(archetype.description()).append("\n");
    });
}
```

Wire it into the render pipeline before `assembleMarkdownDisposition()`.

- [ ] **Step 3: Write failing test for source vocabulary rendering (non-Jungian)**

```java
@Test
void markdownRendersBelbinSlotDescription() {
    var descriptor = AgentDescriptor.builder()
        .agentId("test").name("Test Agent").tenancyId("t1")
        .slot("plant")
        .slotVocabulary("urn:casehub:vocab:belbin")
        .build();

    var rendered = renderer.render(descriptor,
        AgentPromptContext.of(RenderFormat.MARKDOWN));

    var text = rendered.text();
    assertTrue(text.contains("Plant"), "Should contain resolved slot label");
    assertTrue(text.contains("Creative, unorthodox"), "Should contain slot description");
}

@Test
void markdownRendersDiscDispositionProfileDescription() {
    var descriptor = AgentDescriptor.builder()
        .agentId("test").name("Test Agent").tenancyId("t1")
        .dispositionVocabulary("urn:casehub:vocab:disc")
        .disposition(AgentDisposition.builder()
            .dispositionProfile(List.of(new DispositionValue("dominance", 1.0)))
            .build())
        .build();

    var rendered = renderer.render(descriptor,
        AgentPromptContext.of(RenderFormat.MARKDOWN));

    var text = rendered.text();
    assertTrue(text.contains("Dominance"), "Should contain DISC term label");
    assertTrue(text.contains("Results-driven"), "Should contain DISC term description");
}
```

- [ ] **Step 4: Generalize assembleMarkdownCognitiveProfile() for all vocabularies**

Remove the Jungian-only gate (`if (!JUNGIAN_VOCAB_URI.equals(vocabUri)) { return; }`). Instead:
- If Jungian: existing rich rendering (cognitive functions, orientation, anti-patterns)
- If other registered vocabulary with dispositionProfile terms: render framework name + term labels + descriptions in a "Behavioral Profile" section
- If slotVocabulary is set: resolve slot term for label + description in a "Role" section

- [ ] **Step 5: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=EidosRenderPipelineTest`

- [ ] **Step 6: Commit**

```bash
git -C $PROJECT add runtime/
git -C $PROJECT commit -m "feat(renderer): surface archetype identity and source vocabulary descriptions

Adds Personality section rendering archetype name, family, adjectives,
and description in MARKDOWN/PROSE. Generalizes cognitive profile
rendering beyond Jungian — all source vocabulary frameworks now
surface their term labels and descriptions. Fixes the disposition
vocabulary fidelity loss where Belbin, DISC, Big Five, etc.
collapsed to canonical axes without source identity.

Refs #180"
```

### Task 6: A2A_CARD archetype rendering

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/core/renderer/EidosRenderPipeline.java` — add archetype to A2A_CARD JSON
- Test: `runtime/src/test/java/io/casehub/eidos/runtime/renderer/EidosRenderPipelineTest.java`

**Interfaces:**
- Consumes: same as Task 5

- [ ] **Step 1: Write failing test**

```java
@Test
void a2aCardIncludesArchetype() {
    var descriptor = AgentDescriptor.builder()
        .agentId("test").name("Test Agent").tenancyId("t1")
        .archetype(ArchetypeTerm.DETECTIVE)
        .archetypeAdjectives(List.of("meticulous"))
        .build();

    var rendered = renderer.render(descriptor,
        AgentPromptContext.of(RenderFormat.A2A_CARD));

    var text = rendered.text();
    assertTrue(text.contains("\"archetype\""));
    assertTrue(text.contains("\"detective\""));
    assertTrue(text.contains("\"Sage\""));
    assertTrue(text.contains("\"meticulous\""));
}
```

- [ ] **Step 2: Add archetype node to A2A_CARD assembly**

In the A2A_CARD rendering path, add an `archetype` object node:
```json
{
  "archetype": {
    "family": "Sage",
    "subArchetype": "detective",
    "quadrant": "Independence",
    "label": "Detective",
    "description": "Uncovers truth through systematic investigation and evidence",
    "adjectives": ["meticulous"]
  }
}
```

Also add `sourceVocabulary` node for slot and disposition profile source terms.

- [ ] **Step 3: Run tests, verify pass**

- [ ] **Step 4: Commit**

```bash
git -C $PROJECT add runtime/
git -C $PROJECT commit -m "feat(renderer): add archetype and source vocabulary to A2A_CARD

A2A_CARD now includes archetype object (family, subArchetype, quadrant,
label, description, adjectives) and sourceVocabulary entries for
slot and disposition profile terms. Machine consumers can distinguish
archetype identity without parsing prose.

Refs #180"
```

---

## Batch 5: Integration Testing & Documentation

### Task 7: End-to-end integration test and consumer guide update

**Files:**
- Create: `examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/ArchetypeScenarioTest.java`
- Modify: `docs/guides/consumer-guide.md` — add archetype vocabulary section
- Modify: `docs/personality-frameworks.md` — add archetype system section

**Interfaces:**
- Consumes: all previous tasks

- [ ] **Step 1: Write integration test**

```java
@QuarkusTest
class ArchetypeScenarioTest {

    @Inject AgentRegistry registry;
    @Inject VocabularyRegistry vocabRegistry;
    @Inject SystemPromptRenderer renderer;

    @Test
    void archetypeFromYamlRendersFullPipeline() {
        // Register a descriptor with archetype: detective
        // Render it in MARKDOWN
        // Verify archetype section, canonical axes, and framework context all appear
    }

    @Test
    void archetypeAutoDerivesFromMbtiAndEnneagram() {
        // Register a descriptor with mbtiType: INTJ, enneagramType: 5
        // Verify archetype is auto-derived
        // Render and verify archetype section appears
    }

    @Test
    void archetypePlusBelbinSlotRendersAllLayers() {
        // Register descriptor with archetype: detective + slot: monitor-evaluator
        // Render in MARKDOWN
        // Verify: Personality section + Role section + How You Operate section
    }
}
```

- [ ] **Step 2: Run integration test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl examples/agent-scenarios -Dtest=ArchetypeScenarioTest`

- [ ] **Step 3: Update consumer guide**

Add section documenting:
- The archetype vocabulary and how to use it in YAML descriptors
- The faceted selection concept (archetype + adjectives)
- Framework-to-archetype auto-derivation
- Rendered output format for each RenderFormat

- [ ] **Step 4: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`

- [ ] **Step 5: Commit**

```bash
git -C $PROJECT add examples/ docs/
git -C $PROJECT commit -m "feat: archetype integration tests and consumer guide documentation

Adds end-to-end scenario tests for archetype rendering, auto-derivation,
and multi-layer (archetype + slot) rendering. Documents archetype
vocabulary usage in consumer guide.

Closes #180"
```

---

## References

- [specs/archetype-compatibility-matrix.md] — compatibility matrix: framework-to-archetype mappings
- [specs/avatar-generator-contract.md] — avatar generator contract: faceted selection + visual generation
- [vocab/src/main/java/io/casehub/eidos/vocab/BelbinTerm.java] — vocabulary enum pattern
- [api/src/main/java/io/casehub/eidos/api/VocabularyTerm.java] — vocabulary interface
- [runtime/src/main/java/io/casehub/eidos/core/renderer/EidosRenderPipeline.java] — renderer
- [runtime/src/main/java/io/casehub/eidos/core/registrar/DescriptorCollector.java] — descriptor processing
- [docs/protocols/renderer/capability-metadata-rendering.md] — PP-20260611-228599
- [GitHub #180] — focal issue
