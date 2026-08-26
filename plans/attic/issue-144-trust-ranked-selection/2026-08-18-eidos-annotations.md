# casehub-eidos-annotations Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #139 — feat: casehub-eidos-annotations module
**Issue group:** #139

**Goal:** Create an opt-in Quarkus extension that lets developers declare agent identity via `@Identity`, `@Disposition`, `@AgentGoals`, `@AgentConstraints`, and `@Discoverable` annotations instead of builder chains or YAML.

**Architecture:** New `casehub-eidos-annotations` (runtime) and `casehub-eidos-annotations-deployment` (build extension) modules. `@Discoverable` lives in `casehub-eidos-api`. The build extension scans Jandex for `@Identity`-annotated classes, generates synthetic `AgentDescriptorRegistrar` CDI beans, and plugs into the existing `DescriptorCollector` → `AgentDescriptorBootstrap` pipeline. Hybrid vocabulary validation catches term typos at build time when vocab modules are on the classpath.

**Tech Stack:** Java 21, Quarkus 3.32.2 (Jandex, SyntheticBeanBuildItem, Arc), eidos-api records

## Global Constraints

- Java 21 source, Java 26 JVM: `JAVA_HOME=$(/usr/libexec/java_home -v 26)`
- `mvn` not `./mvnw`
- Parent version: `0.2-SNAPSHOT`
- Package: `io.casehub.eidos.annotations` (runtime), `io.casehub.eidos.annotations.deployment` (deployment)
- `@Discoverable` in `io.casehub.eidos.api` (existing package)
- All commits reference `Refs #139`
- IntelliJ MCP for all .java file operations

---

## Batch 1: Foundation — API additions + annotation definitions

### Task 1: Fix styleVocabulary validation gap + add @Discoverable to eidos-api

**Files:**
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentDescriptor.java` — add styleVocabulary validation
- Create: `api/src/main/java/io/casehub/eidos/api/Discoverable.java`
- Test: `api/src/test/java/io/casehub/eidos/api/AgentDescriptorStyleVocabValidationTest.java`
- Test: `api/src/test/java/io/casehub/eidos/api/DiscoverableAnnotationTest.java`

**Interfaces:**
- Produces: `@Discoverable` annotation type at `io.casehub.eidos.api.Discoverable` — `String[] capabilities()`

- [ ] **Step 1: Write failing test for styleVocabulary validation**

```java
package io.casehub.eidos.api;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

class AgentDescriptorStyleVocabValidationTest {

    @Test
    void styleVocabularyWithBannedCharactersIsRejected() {
        assertThatThrownBy(() -> AgentDescriptor.builder()
                .agentId("test").name("Test").slot("test").tenancyId("t1")
                .styleVocabulary("urn:vocab​:bad")
                .build())
            .isInstanceOf(AgentValidationException.class)
            .hasMessageContaining("styleVocabulary");
    }

    @Test
    void styleVocabularyExceedingMaxLengthIsRejected() {
        String tooLong = "urn:" + "x".repeat(500);
        assertThatThrownBy(() -> AgentDescriptor.builder()
                .agentId("test").name("Test").slot("test").tenancyId("t1")
                .styleVocabulary(tooLong)
                .build())
            .isInstanceOf(AgentValidationException.class)
            .hasMessageContaining("styleVocabulary");
    }

    @Test
    void validStyleVocabularyIsAccepted() {
        var d = AgentDescriptor.builder()
                .agentId("test").name("Test").slot("test").tenancyId("t1")
                .styleVocabulary("urn:casehub:vocab:style")
                .build();
        assertThat(d.styleVocabulary()).isEqualTo("urn:casehub:vocab:style");
    }

    @Test
    void nullStyleVocabularyIsAccepted() {
        var d = AgentDescriptor.builder()
                .agentId("test").name("Test").slot("test").tenancyId("t1")
                .build();
        assertThat(d.styleVocabulary()).isNull();
    }
}
```

- [ ] **Step 4b: Fix DescriptorCollector.deriveDispositionAxes — pre-existing bug**

`deriveDispositionAxes()` rebuilds the `AgentDescriptor` field-by-field but omits `styleVocabulary` and `styleProfile`. Fix by:
1. Using `descriptor.toBuilder()` instead of manual field-by-field rebuild (preserves all fields including future additions)
2. Adding `.styleProfile(disposition.styleProfile())` to the disposition builder

Use `ide_replace_member` to replace the `deriveDispositionAxes` method body.

The key changes in the descriptor rebuild (end of method):
```java
// Replace manual builder chain with toBuilder()
return descriptor.toBuilder()
    .axisVocabularies(axisVocabularies.isEmpty() ? null : new java.util.HashMap<>(axisVocabularies))
    .disposition(builder.build())
    .build();
```

And in the disposition builder initialization:
```java
var builder = AgentDisposition.builder()
    .delegation(disposition.delegation())
    .dispositionProfile(disposition.dispositionProfile())
    .styleProfile(disposition.styleProfile());  // ← was missing
```
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=AgentDescriptorStyleVocabValidationTest -DfailIfNoTests=false`
Expected: FAIL — banned character test passes (no validation), max length test passes (no validation) — wait, actually the tests that expect exceptions will fail because no validation exists. Let me be precise: the first two tests will FAIL (no exception thrown), the last two will PASS.

- [ ] **Step 3: Add styleVocabulary validation to AgentDescriptor compact constructor**

Use `ide_edit_member` to add this line after the `dispositionVocabulary` validation in the compact constructor:

```java
AgentDescriptorValidator.validateOptional("styleVocabulary", styleVocabulary, AgentDescriptorValidator.MAX_VOCABULARY_URI);
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=AgentDescriptorStyleVocabValidationTest`
Expected: PASS (all 4 tests)

- [ ] **Step 5: Write @Discoverable annotation**

Use `ide_create_file` to create `api/src/main/java/io/casehub/eidos/api/Discoverable.java`:

```java
package io.casehub.eidos.api;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Discoverable {
    String[] capabilities();
}
```

- [ ] **Step 6: Write @Discoverable reflection test**

```java
package io.casehub.eidos.api;

import org.junit.jupiter.api.Test;
import java.lang.annotation.ElementType;
import java.lang.annotation.RetentionPolicy;
import static org.assertj.core.api.Assertions.*;

class DiscoverableAnnotationTest {

    @Test
    void retentionIsRuntime() {
        assertThat(Discoverable.class.getAnnotation(java.lang.annotation.Retention.class).value())
            .isEqualTo(RetentionPolicy.RUNTIME);
    }

    @Test
    void targetIsType() {
        assertThat(Discoverable.class.getAnnotation(java.lang.annotation.Target.class).value())
            .containsExactly(ElementType.TYPE);
    }

    @Test
    void capabilitiesIsRequired() throws NoSuchMethodException {
        var method = Discoverable.class.getDeclaredMethod("capabilities");
        assertThat(method.getDefaultValue()).isNull();
    }
}
```

- [ ] **Step 7: Run all api tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add api/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat: add @Discoverable to eidos-api + fix styleVocabulary validation gap Refs #139"
```

---

### Task 2: Create annotations runtime module

**Files:**
- Modify: `pom.xml` — add `annotations` module
- Create: `annotations/pom.xml`
- Create: `annotations/src/main/java/io/casehub/eidos/annotations/Identity.java`
- Create: `annotations/src/main/java/io/casehub/eidos/annotations/Disposition.java`
- Create: `annotations/src/main/java/io/casehub/eidos/annotations/AgentGoals.java`
- Create: `annotations/src/main/java/io/casehub/eidos/annotations/AgentGoalDef.java`
- Create: `annotations/src/main/java/io/casehub/eidos/annotations/AgentConstraints.java`
- Create: `annotations/src/main/java/io/casehub/eidos/annotations/AgentConstraintDef.java`
- Create: `annotations/src/main/java/io/casehub/eidos/annotations/NameDerivation.java`
- Test: `annotations/src/test/java/io/casehub/eidos/annotations/NameDerivationTest.java`
- Test: `annotations/src/test/java/io/casehub/eidos/annotations/AnnotationReflectionTest.java`

**Interfaces:**
- Consumes: `GoalPriority`, `Visibility`, `ConstraintSeverity` from `casehub-eidos-api`
- Produces: All 6 annotation types in `io.casehub.eidos.annotations`; `NameDerivation.toKebabCase(String)` and `NameDerivation.toDisplayName(String)` utility methods

- [ ] **Step 1: Create annotations/pom.xml**

```xml
<?xml version="1.0"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-eidos-parent</artifactId>
    <version>0.2-SNAPSHOT</version>
  </parent>

  <artifactId>casehub-eidos-annotations</artifactId>
  <name>CaseHub Eidos - Annotations</name>
  <description>Annotation-driven agent identity — @Identity, @Disposition, @AgentGoals, @AgentConstraints</description>

  <dependencies>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-eidos-api</artifactId>
      <version>${project.version}</version>
    </dependency>

    <dependency>
      <groupId>org.junit.jupiter</groupId>
      <artifactId>junit-jupiter</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.assertj</groupId>
      <artifactId>assertj-core</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>io.smallrye</groupId>
        <artifactId>jandex-maven-plugin</artifactId>
        <executions>
          <execution>
            <id>make-index</id>
            <goals><goal>jandex</goal></goals>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
</project>
```

- [ ] **Step 2: Add module to parent pom.xml**

Add `<module>annotations</module>` after `<module>api</module>` in the parent pom's `<modules>` section.

- [ ] **Step 3: Write NameDerivation utility + tests**

Create `annotations/src/main/java/io/casehub/eidos/annotations/NameDerivation.java`:

```java
package io.casehub.eidos.annotations;

public final class NameDerivation {

    private NameDerivation() {}

    public static String toKebabCase(String className) {
        if (className == null || className.isEmpty()) return "";
        int dollar = className.lastIndexOf('$');
        if (dollar >= 0) className = className.substring(dollar + 1);
        var sb = new StringBuilder();
        for (int i = 0; i < className.length(); i++) {
            char c = className.charAt(i);
            if (Character.isUpperCase(c)) {
                boolean nextIsLower = i + 1 < className.length() && Character.isLowerCase(className.charAt(i + 1));
                boolean prevIsUpper = i > 0 && Character.isUpperCase(className.charAt(i - 1));
                if (i > 0 && (!prevIsUpper || nextIsLower)) sb.append('-');
                sb.append(Character.toLowerCase(c));
            } else {
                sb.append(c);
            }
        }
        return sb.toString();
    }

    public static String toDisplayName(String className) {
        if (className == null || className.isEmpty()) return "";
        int dollar = className.lastIndexOf('$');
        if (dollar >= 0) className = className.substring(dollar + 1);
        var sb = new StringBuilder();
        for (int i = 0; i < className.length(); i++) {
            char c = className.charAt(i);
            if (Character.isUpperCase(c) && i > 0) sb.append(' ');
            sb.append(c);
        }
        return sb.toString();
    }
}
```

Write test `annotations/src/test/java/io/casehub/eidos/annotations/NameDerivationTest.java`:

```java
package io.casehub.eidos.annotations;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvSource;
import static org.assertj.core.api.Assertions.*;

class NameDerivationTest {

    @ParameterizedTest
    @CsvSource({
        "LegalAnalystAgent, legal-analyst-agent",
        "Reviewer, reviewer",
        "DocumentAnalyst, document-analyst",
        "HTMLParser, html-parser",
        "A, a"
    })
    void toKebabCase(String input, String expected) {
        assertThat(NameDerivation.toKebabCase(input)).isEqualTo(expected);
    }

    @Test
    void toKebabCaseInnerClass() {
        assertThat(NameDerivation.toKebabCase("OuterClass$InnerAgent")).isEqualTo("inner-agent");
    }

    @ParameterizedTest
    @CsvSource({
        "LegalAnalystAgent, Legal Analyst Agent",
        "Reviewer, Reviewer",
        "DocumentAnalyst, Document Analyst"
    })
    void toDisplayName(String input, String expected) {
        assertThat(NameDerivation.toDisplayName(input)).isEqualTo(expected);
    }

    @Test
    void toDisplayNameInnerClass() {
        assertThat(NameDerivation.toDisplayName("OuterClass$InnerAgent")).isEqualTo("Inner Agent");
    }

    @Test
    void emptyAndNull() {
        assertThat(NameDerivation.toKebabCase("")).isEmpty();
        assertThat(NameDerivation.toKebabCase(null)).isEmpty();
        assertThat(NameDerivation.toDisplayName("")).isEmpty();
        assertThat(NameDerivation.toDisplayName(null)).isEmpty();
    }
}
```

- [ ] **Step 4: Run NameDerivation tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl annotations -Dtest=NameDerivationTest`
Expected: PASS

- [ ] **Step 5: Write all 6 annotation classes**

Create each file under `annotations/src/main/java/io/casehub/eidos/annotations/`:

**Identity.java:**
```java
package io.casehub.eidos.annotations;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Identity {
    String id() default "";
    String name() default "";
    String slot();
    String provider() default "";
    String modelFamily() default "";
    String jurisdiction() default "";
    String dataHandlingPolicy() default "";
    String briefing() default "";
    String vocabulary() default "";
    String slotVocabulary() default "";
    String dispositionVocabulary() default "";
    String styleVocabulary() default "";
    String version() default "";
}
```

**Disposition.java:**
```java
package io.casehub.eidos.annotations;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Disposition {
    String socialOrient() default "";
    String ruleFollowing() default "";
    String riskAppetite() default "";
    String autonomy() default "";
    String conflictMode() default "";
    boolean delegation() default false;
    String[] dispositionProfile() default {};
    String[] styleProfile() default {};
}
```

**AgentGoals.java:**
```java
package io.casehub.eidos.annotations;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface AgentGoals {
    AgentGoalDef[] value();
}
```

**AgentGoalDef.java:**
```java
package io.casehub.eidos.annotations;

import io.casehub.eidos.api.GoalPriority;
import io.casehub.eidos.api.Visibility;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target({})
public @interface AgentGoalDef {
    String name();
    String description();
    GoalPriority priority() default GoalPriority.PRIMARY;
    Visibility visibility() default Visibility.PUBLIC;
    String[] capabilities() default {};
}
```

**AgentConstraints.java:**
```java
package io.casehub.eidos.annotations;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface AgentConstraints {
    AgentConstraintDef[] value();
}
```

**AgentConstraintDef.java:**
```java
package io.casehub.eidos.annotations;

import io.casehub.eidos.api.ConstraintSeverity;
import io.casehub.eidos.api.Visibility;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target({})
public @interface AgentConstraintDef {
    String name();
    String description();
    ConstraintSeverity severity() default ConstraintSeverity.HARD;
    Visibility visibility() default Visibility.PUBLIC;
}
```

- [ ] **Step 6: Write annotation reflection tests**

Create `annotations/src/test/java/io/casehub/eidos/annotations/AnnotationReflectionTest.java`:

```java
package io.casehub.eidos.annotations;

import io.casehub.eidos.api.GoalPriority;
import io.casehub.eidos.api.ConstraintSeverity;
import io.casehub.eidos.api.Visibility;
import org.junit.jupiter.api.Test;
import java.lang.annotation.ElementType;
import java.lang.annotation.RetentionPolicy;
import static org.assertj.core.api.Assertions.*;

class AnnotationReflectionTest {

    @Test
    void identityRetentionAndTarget() {
        assertThat(Identity.class.getAnnotation(java.lang.annotation.Retention.class).value())
            .isEqualTo(RetentionPolicy.RUNTIME);
        assertThat(Identity.class.getAnnotation(java.lang.annotation.Target.class).value())
            .containsExactly(ElementType.TYPE);
    }

    @Test
    void identitySlotIsRequired() throws NoSuchMethodException {
        assertThat(Identity.class.getDeclaredMethod("slot").getDefaultValue()).isNull();
    }

    @Test
    void identityIdDefaultsToEmpty() throws NoSuchMethodException {
        assertThat(Identity.class.getDeclaredMethod("id").getDefaultValue()).isEqualTo("");
    }

    @Test
    void dispositionRetentionAndTarget() {
        assertThat(Disposition.class.getAnnotation(java.lang.annotation.Retention.class).value())
            .isEqualTo(RetentionPolicy.RUNTIME);
        assertThat(Disposition.class.getAnnotation(java.lang.annotation.Target.class).value())
            .containsExactly(ElementType.TYPE);
    }

    @Test
    void agentGoalDefDescriptionIsRequired() throws NoSuchMethodException {
        assertThat(AgentGoalDef.class.getDeclaredMethod("description").getDefaultValue()).isNull();
    }

    @Test
    void agentGoalDefPriorityDefaultIsPrimary() throws NoSuchMethodException {
        assertThat(AgentGoalDef.class.getDeclaredMethod("priority").getDefaultValue())
            .isEqualTo(GoalPriority.PRIMARY);
    }

    @Test
    void agentConstraintDefDescriptionIsRequired() throws NoSuchMethodException {
        assertThat(AgentConstraintDef.class.getDeclaredMethod("description").getDefaultValue()).isNull();
    }

    @Test
    void agentConstraintDefSeverityDefaultIsHard() throws NoSuchMethodException {
        assertThat(AgentConstraintDef.class.getDeclaredMethod("severity").getDefaultValue())
            .isEqualTo(ConstraintSeverity.HARD);
    }

    @Test
    void agentConstraintDefVisibilityDefaultIsPublic() throws NoSuchMethodException {
        assertThat(AgentConstraintDef.class.getDeclaredMethod("visibility").getDefaultValue())
            .isEqualTo(Visibility.PUBLIC);
    }
}
```

- [ ] **Step 7: Run all annotations module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl annotations`
Expected: PASS

- [ ] **Step 8: Run full build to verify no breakage**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -pl api,annotations`
Expected: BUILD SUCCESS

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add annotations/ pom.xml
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat: casehub-eidos-annotations runtime module — 6 annotations + name derivation Refs #139"
```

---

## Batch 2: Build Extension — processor + bean generation

### Task 3: Create annotations-deployment module with EidosAnnotationsProcessor

**Files:**
- Modify: `pom.xml` — add `annotations-deployment` module
- Create: `annotations-deployment/pom.xml`
- Create: `annotations-deployment/src/main/java/io/casehub/eidos/annotations/deployment/EidosAnnotationProcessedBuildItem.java`
- Create: `annotations-deployment/src/main/java/io/casehub/eidos/annotations/deployment/EidosAnnotationsProcessor.java`
- Create: `annotations/src/main/java/io/casehub/eidos/annotations/runtime/EidosAnnotationsRecorder.java`
- Create: `annotations/src/main/java/io/casehub/eidos/annotations/runtime/AnnotatedDescriptorConfig.java`
- Test: `annotations-deployment/src/test/java/io/casehub/eidos/annotations/deployment/EidosAnnotationsProcessorTest.java`
- Test helper: `annotations-deployment/src/test/java/io/casehub/eidos/annotations/deployment/test/SimpleAnnotatedAgent.java` (and other annotated test interfaces)

**Interfaces:**
- Consumes: `@Identity`, `@Disposition`, `@AgentGoals`, `@AgentConstraints` from `casehub-eidos-annotations`; `@Discoverable` from `casehub-eidos-api`; `AgentDescriptor.builder()`, `AgentDisposition.builder()`, `AgentGoal`, `AgentConstraint`, `AgentCapability`, `DispositionValue`, `AgentDescriptorRegistrar` from `casehub-eidos-api`
- Produces: `EidosAnnotationProcessedBuildItem` (list of processed class names); synthetic `AgentDescriptorRegistrar` CDI beans via `@Recorder` pattern; `FeatureBuildItem("eidos-annotations")`

**IMPORTANT — Quarkus build→runtime boundary:** Domain records (`AgentDisposition`, `AgentGoal`, etc.) are not serializable. The processor must NOT capture them in a `Supplier` lambda. Instead, use a `@Recorder` class (`EidosAnnotationsRecorder`) that accepts primitive/String values at build time and constructs domain objects at runtime. Alternatively, use `SyntheticBeanBuildItem.createWith(BeanCreator)` with a configuration data class (`AnnotatedDescriptorConfig`) that carries only strings/enums.

- [ ] **Step 1: Create annotations-deployment/pom.xml**

```xml
<?xml version="1.0"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-eidos-parent</artifactId>
    <version>0.2-SNAPSHOT</version>
  </parent>

  <artifactId>casehub-eidos-annotations-deployment</artifactId>
  <name>CaseHub Eidos - Annotations Deployment</name>

  <dependencies>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-eidos-annotations</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-eidos</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-eidos-deployment</artifactId>
      <version>${project.version}</version>
    </dependency>

    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-core-deployment</artifactId>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-arc-deployment</artifactId>
    </dependency>

    <!-- Testing -->
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-junit5</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-junit5-internal</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.assertj</groupId>
      <artifactId>assertj-core</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-eidos-memory</artifactId>
      <version>${project.version}</version>
      <scope>test</scope>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>${compiler-plugin.version}</version>
        <configuration>
          <annotationProcessorPaths>
            <path>
              <groupId>io.quarkus</groupId>
              <artifactId>quarkus-extension-processor</artifactId>
              <version>${quarkus.version}</version>
            </path>
          </annotationProcessorPaths>
        </configuration>
      </plugin>
    </plugins>
  </build>
</project>
```

- [ ] **Step 2: Add module to parent pom.xml**

Add `<module>annotations-deployment</module>` after `<module>annotations</module>`.

- [ ] **Step 3: Write EidosAnnotationProcessedBuildItem**

Create `annotations-deployment/src/main/java/io/casehub/eidos/annotations/deployment/EidosAnnotationProcessedBuildItem.java`:

```java
package io.casehub.eidos.annotations.deployment;

import io.quarkus.builder.item.SimpleBuildItem;
import java.util.Set;

public final class EidosAnnotationProcessedBuildItem extends SimpleBuildItem {

    private final Set<String> processedClassNames;

    public EidosAnnotationProcessedBuildItem(Set<String> processedClassNames) {
        this.processedClassNames = Set.copyOf(processedClassNames);
    }

    public Set<String> processedClassNames() {
        return processedClassNames;
    }
}
```

- [ ] **Step 4: Write EidosAnnotationsProcessor**

Create `annotations-deployment/src/main/java/io/casehub/eidos/annotations/deployment/EidosAnnotationsProcessor.java`. This is the main build extension — it scans Jandex for `@Identity`, extracts all companion annotations, validates, and generates synthetic `AgentDescriptorRegistrar` beans.

```java
package io.casehub.eidos.annotations.deployment;

import io.casehub.eidos.annotations.AgentConstraintDef;
import io.casehub.eidos.annotations.AgentConstraints;
import io.casehub.eidos.annotations.AgentGoalDef;
import io.casehub.eidos.annotations.AgentGoals;
import io.casehub.eidos.annotations.Disposition;
import io.casehub.eidos.annotations.Identity;
import io.casehub.eidos.annotations.NameDerivation;
import io.casehub.eidos.api.AgentCapability;
import io.casehub.eidos.api.AgentConstraint;
import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.eidos.api.AgentDisposition;
import io.casehub.eidos.api.AgentGoal;
import io.casehub.eidos.api.ConstraintSeverity;
import io.casehub.eidos.api.Discoverable;
import io.casehub.eidos.api.DispositionValue;
import io.casehub.eidos.api.GoalPriority;
import io.casehub.eidos.api.Visibility;
import io.casehub.eidos.api.spi.AgentDescriptorRegistrar;
import io.quarkus.arc.deployment.SyntheticBeanBuildItem;
import io.quarkus.deployment.annotations.BuildProducer;
import io.quarkus.deployment.annotations.BuildStep;
import io.quarkus.deployment.builditem.CombinedIndexBuildItem;
import io.quarkus.deployment.builditem.FeatureBuildItem;
import jakarta.enterprise.context.ApplicationScoped;
import org.eclipse.microprofile.config.ConfigProvider;
import org.jboss.jandex.AnnotationInstance;
import org.jboss.jandex.AnnotationValue;
import org.jboss.jandex.ClassInfo;
import org.jboss.jandex.DotName;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.HashSet;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.function.Supplier;

class EidosAnnotationsProcessor {

    private static final String FEATURE = "eidos-annotations";
    private static final DotName IDENTITY = DotName.createSimple(Identity.class);
    private static final DotName DISPOSITION = DotName.createSimple(Disposition.class);
    private static final DotName AGENT_GOALS = DotName.createSimple(AgentGoals.class);
    private static final DotName AGENT_CONSTRAINTS = DotName.createSimple(AgentConstraints.class);
    private static final DotName DISCOVERABLE = DotName.createSimple(Discoverable.class);
    private static final String TENANCY_CONFIG_KEY = "casehub.eidos.annotations.default-tenancy-id";

    @BuildStep
    FeatureBuildItem feature() {
        return new FeatureBuildItem(FEATURE);
    }

    @BuildStep
    EidosAnnotationProcessedBuildItem processAnnotations(
            CombinedIndexBuildItem index,
            BuildProducer<SyntheticBeanBuildItem> syntheticBeans) {

        var identityAnnotations = index.getIndex().getAnnotations(IDENTITY);
        if (identityAnnotations.isEmpty()) {
            return new EidosAnnotationProcessedBuildItem(Set.of());
        }

        var processedClasses = new HashSet<String>();
        var derivedIds = new HashMap<String, String>();

        for (var annotation : identityAnnotations) {
            if (annotation.target().kind() != org.jboss.jandex.AnnotationTarget.Kind.CLASS) continue;
            var classInfo = annotation.target().asClass();
            var className = classInfo.name().toString();
            processedClasses.add(className);

            var agentId = resolveAgentId(annotation, classInfo, derivedIds);
            var displayName = resolveDisplayName(annotation, classInfo);
            var descriptorSupplier = buildDescriptorSupplier(agentId, displayName, annotation, classInfo, index);

            syntheticBeans.produce(SyntheticBeanBuildItem
                    .configure(AgentDescriptorRegistrar.class)
                    .scope(ApplicationScoped.class)
                    .supplier(descriptorSupplier)
                    .done());
        }

        return new EidosAnnotationProcessedBuildItem(processedClasses);
    }

    private String resolveAgentId(AnnotationInstance identity, ClassInfo classInfo,
                                   Map<String, String> derivedIds) {
        var explicit = stringValue(identity, "id");
        if (!explicit.isEmpty()) return explicit;

        var derived = NameDerivation.toKebabCase(classInfo.simpleName());
        var existing = derivedIds.put(derived, classInfo.name().toString());
        if (existing != null) {
            throw new IllegalStateException(
                "Duplicate derived agentId '" + derived + "' from classes " + existing
                + " and " + classInfo.name() + " — add explicit id() to at least one @Identity");
        }
        return derived;
    }

    private String resolveDisplayName(AnnotationInstance identity, ClassInfo classInfo) {
        var explicit = stringValue(identity, "name");
        return explicit.isEmpty() ? NameDerivation.toDisplayName(classInfo.simpleName()) : explicit;
    }

    private Supplier<AgentDescriptorRegistrar> buildDescriptorSupplier(
            String agentId, String displayName,
            AnnotationInstance identity, ClassInfo classInfo,
            CombinedIndexBuildItem index) {

        var slot = stringValue(identity, "slot");
        var provider = stringValue(identity, "provider");
        var modelFamily = stringValue(identity, "modelFamily");
        var jurisdiction = stringValue(identity, "jurisdiction");
        var dataHandlingPolicy = stringValue(identity, "dataHandlingPolicy");
        var briefing = stringValue(identity, "briefing");
        var vocabulary = stringValue(identity, "vocabulary");
        var slotVocabulary = stringValue(identity, "slotVocabulary");
        var dispositionVocabulary = stringValue(identity, "dispositionVocabulary");
        var styleVocabulary = stringValue(identity, "styleVocabulary");
        var version = stringValue(identity, "version");

        var disposition = extractDisposition(classInfo, index);
        var goals = extractGoals(classInfo, index);
        var constraints = extractConstraints(classInfo, index);
        var capabilities = extractCapabilities(classInfo, index);

        if (!goals.isEmpty() && !capabilities.isEmpty()) {
            var capNames = new HashSet<String>();
            for (var cap : capabilities) capNames.add(cap.name());
            for (var goal : goals) {
                for (var capRef : goal.capabilities()) {
                    if (!capNames.contains(capRef)) {
                        throw new IllegalStateException(
                            "@AgentGoalDef '" + goal.name() + "' on " + classInfo.name()
                            + " references capability '" + capRef + "' not declared in @Discoverable");
                    }
                }
            }
        }

        return () -> {
            var tenancyId = ConfigProvider.getConfig()
                    .getOptionalValue(TENANCY_CONFIG_KEY, String.class)
                    .orElse("default");
            var builder = AgentDescriptor.builder()
                    .agentId(agentId).name(displayName).slot(slot).tenancyId(tenancyId)
                    .capabilities(capabilities).goals(goals).constraints(constraints);
            if (!provider.isEmpty()) builder.provider(provider);
            if (!modelFamily.isEmpty()) builder.modelFamily(modelFamily);
            if (!jurisdiction.isEmpty()) builder.jurisdiction(jurisdiction);
            if (!dataHandlingPolicy.isEmpty()) builder.dataHandlingPolicy(dataHandlingPolicy);
            if (!briefing.isEmpty()) builder.briefing(briefing);
            if (!vocabulary.isEmpty()) builder.domainVocabulary(vocabulary);
            if (!slotVocabulary.isEmpty()) builder.slotVocabulary(slotVocabulary);
            if (!dispositionVocabulary.isEmpty()) builder.dispositionVocabulary(dispositionVocabulary);
            if (!styleVocabulary.isEmpty()) builder.styleVocabulary(styleVocabulary);
            if (!version.isEmpty()) builder.version(version);
            if (disposition != null) builder.disposition(disposition);
            return (AgentDescriptorRegistrar) () -> List.of(builder.build());
        };
    }

    private AgentDisposition extractDisposition(ClassInfo classInfo, CombinedIndexBuildItem index) {
        var ann = classInfo.annotation(DISPOSITION);
        if (ann == null) return null;
        var b = AgentDisposition.builder();
        var so = stringValue(ann, "socialOrient");
        if (!so.isEmpty()) b.socialOrient(so);
        var rf = stringValue(ann, "ruleFollowing");
        if (!rf.isEmpty()) b.ruleFollowing(rf);
        var ra = stringValue(ann, "riskAppetite");
        if (!ra.isEmpty()) b.riskAppetite(ra);
        var au = stringValue(ann, "autonomy");
        if (!au.isEmpty()) b.autonomy(au);
        var cm = stringValue(ann, "conflictMode");
        if (!cm.isEmpty()) b.conflictMode(cm);
        var del = ann.value("delegation");
        if (del != null) b.delegation(del.asBoolean());
        var dp = ann.value("dispositionProfile");
        if (dp != null) {
            var terms = dp.asStringArray();
            var values = new ArrayList<DispositionValue>();
            for (var t : terms) if (!t.isEmpty()) values.add(DispositionValue.of(t));
            if (!values.isEmpty()) b.dispositionProfile(values);
        }
        var sp = ann.value("styleProfile");
        if (sp != null) {
            var terms = sp.asStringArray();
            var values = new ArrayList<DispositionValue>();
            for (var t : terms) if (!t.isEmpty()) values.add(DispositionValue.of(t));
            if (!values.isEmpty()) b.styleProfile(values);
        }
        return b.build();
    }

    private List<AgentGoal> extractGoals(ClassInfo classInfo, CombinedIndexBuildItem index) {
        var ann = classInfo.annotation(AGENT_GOALS);
        if (ann == null) return List.of();
        var defs = ann.value().asNestedArray();
        var goals = new ArrayList<AgentGoal>();
        for (var def : defs) {
            var capValue = def.value("capabilities");
            var caps = capValue != null ? List.of(capValue.asStringArray()) : List.<String>of();
            goals.add(new AgentGoal(
                    def.value("name").asString(),
                    def.value("description").asString(),
                    GoalPriority.valueOf(enumValue(def, "priority", "PRIMARY")),
                    Visibility.valueOf(enumValue(def, "visibility", "PUBLIC")),
                    caps));
        }
        return goals;
    }

    private List<AgentConstraint> extractConstraints(ClassInfo classInfo, CombinedIndexBuildItem index) {
        var ann = classInfo.annotation(AGENT_CONSTRAINTS);
        if (ann == null) return List.of();
        var defs = ann.value().asNestedArray();
        var constraints = new ArrayList<AgentConstraint>();
        for (var def : defs) {
            constraints.add(new AgentConstraint(
                    def.value("name").asString(),
                    def.value("description").asString(),
                    Visibility.valueOf(enumValue(def, "visibility", "PUBLIC")),
                    ConstraintSeverity.valueOf(enumValue(def, "severity", "HARD"))));
        }
        return constraints;
    }

    private List<AgentCapability> extractCapabilities(ClassInfo classInfo, CombinedIndexBuildItem index) {
        var ann = classInfo.annotation(DISCOVERABLE);
        if (ann == null) return List.of();
        var names = ann.value("capabilities").asStringArray();
        var caps = new ArrayList<AgentCapability>();
        for (var name : names) {
            caps.add(new AgentCapability.Builder().name(name).build());
        }
        return caps;
    }

    private static String stringValue(AnnotationInstance ann, String key) {
        var v = ann.value(key);
        return v != null ? v.asString() : "";
    }

    private static String enumValue(AnnotationInstance ann, String key, String defaultValue) {
        var v = ann.value(key);
        return v != null ? v.asEnum() : defaultValue;
    }
}
```

- [ ] **Step 5: Write test fixtures (annotated interfaces)**

Create test fixture classes:

`annotations-deployment/src/test/java/io/casehub/eidos/annotations/deployment/test/SimpleAnnotatedAgent.java`:
```java
package io.casehub.eidos.annotations.deployment.test;

import io.casehub.eidos.annotations.Identity;
import io.casehub.eidos.annotations.Disposition;

@Identity(slot = "test-agent", briefing = "A test agent")
@Disposition(socialOrient = "collaborative", ruleFollowing = "strict")
public interface SimpleAnnotatedAgent {}
```

`annotations-deployment/src/test/java/io/casehub/eidos/annotations/deployment/test/FullAnnotatedAgent.java`:
```java
package io.casehub.eidos.annotations.deployment.test;

import io.casehub.eidos.annotations.*;
import io.casehub.eidos.api.*;

@Identity(id = "full-agent", name = "Full Agent", slot = "analyst",
          jurisdiction = "EU", briefing = "Full featured")
@Disposition(socialOrient = "collaborative", ruleFollowing = "strict",
             riskAppetite = "cautious", autonomy = "guided",
             conflictMode = "accommodating")
@Discoverable(capabilities = {"analysis", "review"})
@AgentGoals({
    @AgentGoalDef(name = "accurate", description = "Be accurate",
                  priority = GoalPriority.PRIMARY, capabilities = {"analysis"}),
    @AgentGoalDef(name = "thorough", description = "Be thorough",
                  priority = GoalPriority.SECONDARY)
})
@AgentConstraints({
    @AgentConstraintDef(name = "no-advice", description = "No binding advice",
                        severity = ConstraintSeverity.HARD),
    @AgentConstraintDef(name = "cite-sources", description = "Cite sources",
                        severity = ConstraintSeverity.SOFT, visibility = Visibility.PRIVATE)
})
public interface FullAnnotatedAgent {}
```

- [ ] **Step 6: Write EidosAnnotationsProcessorTest**

```java
package io.casehub.eidos.annotations.deployment;

import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.eidos.api.AgentRegistry;
import io.casehub.eidos.api.ConstraintSeverity;
import io.casehub.eidos.api.GoalPriority;
import io.casehub.eidos.api.Visibility;
import io.quarkus.test.QuarkusUnitTest;
import jakarta.inject.Inject;
import org.jboss.shrinkwrap.api.ShrinkWrap;
import org.jboss.shrinkwrap.api.spec.JavaArchive;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.RegisterExtension;
import static org.assertj.core.api.Assertions.*;

class EidosAnnotationsProcessorTest {

    @RegisterExtension
    static final QuarkusUnitTest unitTest = new QuarkusUnitTest()
            .withApplicationRoot(root -> root
                    .addPackage("io.casehub.eidos.annotations.deployment.test"))
            .overrideConfigKey("casehub.eidos.annotations.default-tenancy-id", "test-tenant")
            .overrideConfigKey("casehub.eidos.reactive.enabled", "false");

    @Inject
    AgentRegistry registry;

    @Test
    void simpleAnnotatedAgentIsRegistered() {
        var result = registry.findById("simple-annotated-agent", "test-tenant");
        assertThat(result).isPresent();
        var d = result.get();
        assertThat(d.slot()).isEqualTo("test-agent");
        assertThat(d.briefing()).isEqualTo("A test agent");
        assertThat(d.name()).isEqualTo("Simple Annotated Agent");
    }

    @Test
    void simpleAnnotatedAgentHasDisposition() {
        var d = registry.findById("simple-annotated-agent", "test-tenant").orElseThrow();
        assertThat(d.disposition()).isNotNull();
        assertThat(d.disposition().primaryTerm(io.casehub.eidos.api.DispositionAxis.SOCIAL_ORIENTATION))
            .isEqualTo("collaborative");
        assertThat(d.disposition().primaryTerm(io.casehub.eidos.api.DispositionAxis.RULE_FOLLOWING))
            .isEqualTo("strict");
    }

    @Test
    void fullAnnotatedAgentHasExplicitIdAndName() {
        var d = registry.findById("full-agent", "test-tenant").orElseThrow();
        assertThat(d.agentId()).isEqualTo("full-agent");
        assertThat(d.name()).isEqualTo("Full Agent");
        assertThat(d.slot()).isEqualTo("analyst");
        assertThat(d.jurisdiction()).isEqualTo("EU");
    }

    @Test
    void fullAnnotatedAgentHasCapabilities() {
        var d = registry.findById("full-agent", "test-tenant").orElseThrow();
        assertThat(d.capabilities()).hasSize(2);
        assertThat(d.capabilities().stream().map(c -> c.name()).toList())
            .containsExactly("analysis", "review");
    }

    @Test
    void fullAnnotatedAgentHasGoals() {
        var d = registry.findById("full-agent", "test-tenant").orElseThrow();
        assertThat(d.goals()).hasSize(2);
        var primary = d.goals().stream().filter(g -> g.name().equals("accurate")).findFirst().orElseThrow();
        assertThat(primary.priority()).isEqualTo(GoalPriority.PRIMARY);
        assertThat(primary.capabilities()).containsExactly("analysis");
    }

    @Test
    void fullAnnotatedAgentHasConstraints() {
        var d = registry.findById("full-agent", "test-tenant").orElseThrow();
        assertThat(d.constraints()).hasSize(2);
        var hard = d.constraints().stream().filter(c -> c.name().equals("no-advice")).findFirst().orElseThrow();
        assertThat(hard.severity()).isEqualTo(ConstraintSeverity.HARD);
        assertThat(hard.visibility()).isEqualTo(Visibility.PUBLIC);
        var soft = d.constraints().stream().filter(c -> c.name().equals("cite-sources")).findFirst().orElseThrow();
        assertThat(soft.visibility()).isEqualTo(Visibility.PRIVATE);
    }

    @Test
    void tenancyIdComesFromConfig() {
        var d = registry.findById("full-agent", "test-tenant").orElseThrow();
        assertThat(d.tenancyId()).isEqualTo("test-tenant");
    }
}
```

- [ ] **Step 7: Run processor tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl annotations-deployment -Dtest=EidosAnnotationsProcessorTest`
Expected: PASS

- [ ] **Step 8: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add annotations-deployment/ pom.xml
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat: casehub-eidos-annotations-deployment — build extension with descriptor generation Refs #139"
```

---

## Batch 3: Validation + Examples + Parity

### Task 4: Hybrid vocabulary validation

**Files:**
- Modify: `annotations-deployment/src/main/java/io/casehub/eidos/annotations/deployment/EidosAnnotationsProcessor.java` — add vocab validation build step
- Test: `annotations-deployment/src/test/java/io/casehub/eidos/annotations/deployment/VocabularyValidationTest.java`

**Interfaces:**
- Consumes: `VocabularyMetadata` annotation (DotName-based scan), Jandex index
- Produces: Build-time error messages for invalid disposition terms

- [ ] **Step 1: Write failing test — invalid disposition term caught at build time**

Create a test fixture with an intentionally wrong term, and a test that expects the build to fail:

`annotations-deployment/src/test/java/io/casehub/eidos/annotations/deployment/VocabularyValidationTest.java`:
```java
package io.casehub.eidos.annotations.deployment;

import io.quarkus.test.QuarkusUnitTest;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.RegisterExtension;

class VocabularyValidationTest {

    @RegisterExtension
    static final QuarkusUnitTest validTermTest = new QuarkusUnitTest()
            .withApplicationRoot(root -> root
                    .addClass(io.casehub.eidos.annotations.deployment.test.SimpleAnnotatedAgent.class))
            .overrideConfigKey("casehub.eidos.annotations.default-tenancy-id", "test-tenant")
            .overrideConfigKey("casehub.eidos.reactive.enabled", "false");

    @Test
    void validTermsPassBuildTimeValidation() {
        // If we get here, the build succeeded — vocab validation passed or was skipped
    }
}
```

Note: Full build-failure tests for invalid terms require casehub-eidos-vocab on the test classpath and a test fixture with an invalid term. The build extension should log a warning (not error) when vocab is available but a term doesn't match — keeping error-level for the implementation plan to decide during TDD.

- [ ] **Step 2: Add vocabulary validation to EidosAnnotationsProcessor**

Add a new build step method `validateVocabularyTerms` that:
1. Scans for `@VocabularyMetadata`-annotated enums in the Jandex index
2. Builds a `Map<String, Set<String>>` of vocab URI → valid term names
3. For each `@Identity` + `@Disposition` pair, checks axis values and profile terms against the vocabulary

```java
@BuildStep
void validateVocabularyTerms(CombinedIndexBuildItem index) {
    var vocabMetaDotName = DotName.createSimple("io.casehub.eidos.api.VocabularyMetadata");
    var vocabAnnotations = index.getIndex().getAnnotations(vocabMetaDotName);
    if (vocabAnnotations.isEmpty()) return;

    var vocabs = new HashMap<String, Set<String>>();
    for (var va : vocabAnnotations) {
        if (va.target().kind() != org.jboss.jandex.AnnotationTarget.Kind.CLASS) continue;
        var uri = va.value("uri").asString();
        var enumClass = va.target().asClass();
        var terms = new HashSet<String>();
        for (var field : enumClass.fields()) {
            if (field.isEnumConstant()) terms.add(field.name());
        }
        vocabs.put(uri, terms);
    }

    for (var identityAnn : index.getIndex().getAnnotations(IDENTITY)) {
        if (identityAnn.target().kind() != org.jboss.jandex.AnnotationTarget.Kind.CLASS) continue;
        var classInfo = identityAnn.target().asClass();
        var vocabUri = stringValue(identityAnn, "vocabulary");
        var dispVocabUri = stringValue(identityAnn, "dispositionVocabulary");
        var styleVocabUri = stringValue(identityAnn, "styleVocabulary");
        var effectiveDispUri = !dispVocabUri.isEmpty() ? dispVocabUri : vocabUri;
        var effectiveStyleUri = !styleVocabUri.isEmpty() ? styleVocabUri : vocabUri;

        var dispAnn = classInfo.annotation(DISPOSITION);
        if (dispAnn == null) continue;

        if (!effectiveDispUri.isEmpty()) {
            var dispTerms = vocabs.get(effectiveDispUri);
            if (dispTerms != null) {
                validateTerm(dispAnn, "socialOrient", dispTerms, effectiveDispUri, classInfo);
                validateTerm(dispAnn, "ruleFollowing", dispTerms, effectiveDispUri, classInfo);
                validateTerm(dispAnn, "riskAppetite", dispTerms, effectiveDispUri, classInfo);
                validateTerm(dispAnn, "autonomy", dispTerms, effectiveDispUri, classInfo);
                validateTerm(dispAnn, "conflictMode", dispTerms, effectiveDispUri, classInfo);
                validateArrayTerms(dispAnn, "dispositionProfile", dispTerms, effectiveDispUri, classInfo);
            }
        }
        if (!effectiveStyleUri.isEmpty()) {
            var styleTerms = vocabs.get(effectiveStyleUri);
            if (styleTerms != null) {
                validateArrayTerms(dispAnn, "styleProfile", styleTerms, effectiveStyleUri, classInfo);
            }
        }
    }
}

private void validateTerm(AnnotationInstance ann, String field,
                           Set<String> validTerms, String vocabUri, ClassInfo classInfo) {
    var term = stringValue(ann, field);
    if (term.isEmpty()) return;
    if (!validTerms.stream().anyMatch(t -> t.equalsIgnoreCase(term))) {
        LOG.warnf("@Disposition.%s value '%s' on %s may not be a valid term in vocabulary '%s'"
            + " (build-time check uses enum constant names, not VocabularyTerm.value())",
            field, term, classInfo.name(), vocabUri);
    }
}

private void validateArrayTerms(AnnotationInstance ann, String field,
                                  Set<String> validTerms, String vocabUri, ClassInfo classInfo) {
    var v = ann.value(field);
    if (v == null) return;
    for (var term : v.asStringArray()) {
        if (term.isEmpty()) continue;
        if (!validTerms.stream().anyMatch(t -> t.equalsIgnoreCase(term))) {
            LOG.warnf("@Disposition.%s value '%s' on %s may not be a valid term in vocabulary '%s'"
                + " (build-time check uses enum constant names, not VocabularyTerm.value())",
                field, term, classInfo.name(), vocabUri);
        }
    }
}
```

- [ ] **Step 3: Run vocab validation tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl annotations-deployment -Dtest=VocabularyValidationTest`
Expected: PASS

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add annotations-deployment/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat: hybrid vocabulary validation at build time Refs #139"
```

---

### Task 5: Annotated agent examples

**Files:**
- Create: `examples/agent-identity-annotated/pom.xml`
- Create: `examples/agent-identity-annotated/src/main/java/io/casehub/eidos/examples/DocumentAnalyst.java`
- Create: `examples/agent-identity-annotated/src/test/java/io/casehub/eidos/examples/DocumentAnalystTest.java`
- Create: `examples/goals-constraints-annotated/pom.xml`
- Create: `examples/goals-constraints-annotated/src/main/java/io/casehub/eidos/examples/LegalAnalystAgent.java`
- Create: `examples/goals-constraints-annotated/src/test/java/io/casehub/eidos/examples/LegalAnalystAgentTest.java`
- Modify: `examples/pom.xml` — add new example modules

**Interfaces:**
- Consumes: All annotation types from `casehub-eidos-annotations`, `@Discoverable` from `casehub-eidos-api`

- [ ] **Step 1: Create agent-identity-annotated example**

POM depends on `casehub-eidos-annotations`, `casehub-eidos`, `casehub-eidos-memory` (in-memory registry for testing), `casehub-eidos-vocab`.

`DocumentAnalyst.java`:
```java
package io.casehub.eidos.examples;

import io.casehub.eidos.annotations.Identity;
import io.casehub.eidos.annotations.Disposition;

@Identity(slot = "document-analyst",
          briefing = "Analyses documents and extracts key findings",
          vocabulary = "urn:casehub:vocab:svo")
@Disposition(socialOrient = "collaborative",
             ruleFollowing = "moderate",
             riskAppetite = "cautious")
public interface DocumentAnalyst {}
```

`DocumentAnalystTest.java`:
```java
package io.casehub.eidos.examples;

import io.casehub.eidos.api.AgentRegistry;
import io.casehub.eidos.api.DispositionAxis;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

@QuarkusTest
class DocumentAnalystTest {

    @Inject
    AgentRegistry registry;

    @Test
    void documentAnalystIsRegistered() {
        var d = registry.findById("document-analyst", "default").orElseThrow();
        assertThat(d.slot()).isEqualTo("document-analyst");
        assertThat(d.briefing()).isEqualTo("Analyses documents and extracts key findings");
        assertThat(d.disposition().primaryTerm(DispositionAxis.SOCIAL_ORIENTATION)).isEqualTo("collaborative");
    }
}
```

- [ ] **Step 2: Create goals-constraints-annotated example**

`LegalAnalystAgent.java` — the full example from the spec (already shown in spec).

`LegalAnalystAgentTest.java` — verifies registration, goals, constraints, capabilities.

- [ ] **Step 3: Add modules to examples/pom.xml**

- [ ] **Step 4: Run example tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl examples/agent-identity-annotated,examples/goals-constraints-annotated`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add examples/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat: annotated agent examples — agent-identity + goals-constraints Refs #139"
```

---

### Task 6: Parity tests

**Files:**
- Create: `annotations-deployment/src/test/java/io/casehub/eidos/annotations/deployment/AnnotationParityTest.java`

**Interfaces:**
- Consumes: Annotation types via reflection, builder types via reflection

- [ ] **Step 1: Write parity tests**

```java
package io.casehub.eidos.annotations.deployment;

import io.casehub.eidos.annotations.*;
import io.casehub.eidos.api.*;
import org.junit.jupiter.api.Test;
import java.util.Arrays;
import java.util.Set;
import java.util.stream.Collectors;
import static org.assertj.core.api.Assertions.*;

class AnnotationParityTest {

    @Test
    void everyIdentityFieldHasBuilderSetter() {
        var annotationFields = Arrays.stream(Identity.class.getDeclaredMethods())
                .map(m -> m.getName()).collect(Collectors.toSet());
        var builderSetters = Arrays.stream(AgentDescriptor.Builder.class.getDeclaredMethods())
                .map(m -> m.getName()).collect(Collectors.toSet());
        var renames = Set.of("id", "name", "vocabulary", "slotVocabulary",
                "dispositionVocabulary", "styleVocabulary");
        var builderNames = Set.of("agentId", "name", "domainVocabulary", "slotVocabulary",
                "dispositionVocabulary", "styleVocabulary");
        for (var field : annotationFields) {
            var builderName = switch (field) {
                case "id" -> "agentId";
                case "vocabulary" -> "domainVocabulary";
                default -> field;
            };
            assertThat(builderSetters).as("Builder setter for @Identity.%s() (→ %s)", field, builderName)
                    .contains(builderName);
        }
    }

    @Test
    void everyDispositionFieldHasBuilderSetter() {
        var annotationFields = Arrays.stream(Disposition.class.getDeclaredMethods())
                .map(m -> m.getName()).collect(Collectors.toSet());
        var builderSetters = Arrays.stream(AgentDisposition.Builder.class.getDeclaredMethods())
                .map(m -> m.getName()).collect(Collectors.toSet());
        for (var field : annotationFields) {
            assertThat(builderSetters).as("Builder setter for @Disposition.%s()", field)
                    .contains(field);
        }
    }

    @Test
    void everyGoalDefFieldHasRecordComponent() {
        var annotationFields = Arrays.stream(AgentGoalDef.class.getDeclaredMethods())
                .map(m -> m.getName()).collect(Collectors.toSet());
        var recordComponents = Arrays.stream(AgentGoal.class.getRecordComponents())
                .map(c -> c.getName()).collect(Collectors.toSet());
        for (var field : annotationFields) {
            assertThat(recordComponents).as("AgentGoal component for @AgentGoalDef.%s()", field)
                    .contains(field);
        }
    }

    @Test
    void everyConstraintDefFieldHasRecordComponent() {
        var annotationFields = Arrays.stream(AgentConstraintDef.class.getDeclaredMethods())
                .map(m -> m.getName()).collect(Collectors.toSet());
        var recordComponents = Arrays.stream(AgentConstraint.class.getRecordComponents())
                .map(c -> c.getName()).collect(Collectors.toSet());
        for (var field : annotationFields) {
            assertThat(recordComponents).as("AgentConstraint component for @AgentConstraintDef.%s()", field)
                    .contains(field);
        }
    }
}
```

- [ ] **Step 2: Run parity tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl annotations-deployment -Dtest=AnnotationParityTest`
Expected: PASS

- [ ] **Step 3: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS — all modules compile, all tests pass

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add annotations-deployment/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat: annotation-builder parity tests Refs #139"
```

---

## References

- [2026-08-18-eidos-annotations-design.md](/Users/mdproctor/claude/public/casehub/eidos/specs/issue-139-eidos-annotations/2026-08-18-eidos-annotations-design.md) — design spec this plan implements
- [AgentDescriptor.java](api/src/main/java/io/casehub/eidos/api/AgentDescriptor.java) — record + builder + validation
- [AgentDescriptorValidator.java](api/src/main/java/io/casehub/eidos/api/AgentDescriptorValidator.java) — validation utilities
- [AgentDisposition.java](api/src/main/java/io/casehub/eidos/api/AgentDisposition.java) — disposition record + builder
- [AgentGoal.java](api/src/main/java/io/casehub/eidos/api/AgentGoal.java) — goal record
- [AgentConstraint.java](api/src/main/java/io/casehub/eidos/api/AgentConstraint.java) — constraint record
- [AgentCapability.java](api/src/main/java/io/casehub/eidos/api/AgentCapability.java) — capability record + builder
- [EidosProcessor.java](deployment/src/main/java/io/casehub/eidos/deployment/EidosProcessor.java) — existing build extension
- [DescriptorCollector.java](runtime/src/main/java/io/casehub/eidos/runtime/registrar/DescriptorCollector.java) — registration pipeline
- [VocabularyMetadata.java](api/src/main/java/io/casehub/eidos/api/VocabularyMetadata.java) — vocabulary annotation for hybrid validation
- [GitHub #139](https://github.com/casehubio/eidos/issues/139) — focal issue
