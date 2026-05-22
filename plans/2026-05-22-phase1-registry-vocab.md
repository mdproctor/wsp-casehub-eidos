# Phase 1 — Agent Registry, Vocabulary, and Well-Known Vocabularies Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement the concrete backing for all Phase 1 SPIs in casehub-eidos-api: JPA and in-memory agent registries, CDI vocabulary registry, NoOp capability health, and SVO/Conscientiousness/CasehubSlot well-known vocabulary producers.

**Architecture:** Normalized JPA schema (agent_descriptor + agent_capability join table, indexed on capability name and tenancy+slot). Reactive JPA tier gated behind `casehub.eidos.reactive.enabled=true` build property per qhorus pattern. CDI vocabulary registry discovers `@Produces Vocabulary` beans at startup and merges with programmatic registrations.

**Tech Stack:** Java 21, Quarkus 3.32.2, Hibernate ORM (blocking), Hibernate Reactive Panache (reactive, build-gated), Jackson, H2 (blocking tests), PostgreSQL DevServices (reactive tests), JUnit 5, AssertJ, ArchUnit 1.4.1.

**Build command:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl <module>` — always use `mvn`, not `./mvnw`.

**Project root:** `/Users/mdproctor/claude/casehub/eidos`

---

## File Map

```
api/src/main/java/io/casehub/eidos/api/
  AgentDescriptor.java            MODIFY — add tenancyId field (last)
  AgentQuery.java                 CREATE — criteria record for find()
  AgentRegistry.java              MODIFY — replace 3 query methods with find(AgentQuery)
  ReactiveAgentRegistry.java      MODIFY — same, Uni<List<AgentDescriptor>> find(AgentQuery)

api/src/test/java/io/casehub/eidos/api/
  AgentQueryTest.java             CREATE
  AgentRegistrySpiTest.java       CREATE — contract compile test

deployment/src/main/java/io/casehub/eidos/deployment/
  EidosBuildTimeConfig.java       CREATE — @ConfigRoot BUILD_TIME for reactive gate
  EidosProcessor.java             MODIFY — inject EidosBuildTimeConfig

runtime/pom.xml                   MODIFY — add hibernate-reactive-panache, reactive-pg-client
runtime/src/main/resources/db/eidos/migration/
  V1__initial_schema.sql          CREATE

runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/
  AgentDescriptorEntity.java      CREATE
  AgentCapabilityEntity.java      CREATE
  AgentDescriptorMapper.java      CREATE
  AgentDescriptorReactivePanacheRepo.java  CREATE — @IfBuildProperty-gated

runtime/src/main/java/io/casehub/eidos/runtime/registry/
  JpaAgentRegistry.java           CREATE — @ApplicationScoped, EntityManager
  JpaReactiveAgentRegistry.java   CREATE — @IfBuildProperty, @WithTransaction

runtime/src/main/java/io/casehub/eidos/runtime/vocabulary/
  CdiVocabularyRegistry.java      CREATE — @DefaultBean, @Any Instance<Vocabulary>

runtime/src/main/java/io/casehub/eidos/runtime/health/
  NoOpCapabilityHealth.java       CREATE — @DefaultBean, always Ready

runtime/src/test/resources/
  application.properties          CREATE — H2 config for blocking tests

runtime/src/test/java/io/casehub/eidos/runtime/registry/
  JpaAgentRegistryTest.java       CREATE
  JpaReactiveAgentRegistryTest.java   CREATE — @TestProfile(ReactiveTestProfile)
  ReactiveTestProfile.java        CREATE
  BlockingReactiveParityTest.java CREATE — ArchUnit Uni<T> check

runtime/src/test/java/io/casehub/eidos/runtime/vocabulary/
  CdiVocabularyRegistryTest.java  CREATE

runtime/src/test/java/io/casehub/eidos/runtime/health/
  NoOpCapabilityHealthTest.java   CREATE

persistence-memory/pom.xml        MODIFY — add Jandex plugin execution
persistence-memory/src/main/java/io/casehub/eidos/memory/
  InMemoryAgentRegistry.java      CREATE — @Alternative @Priority(1)
  InMemoryReactiveAgentRegistry.java  CREATE — delegates to above

persistence-memory/src/test/resources/
  application.properties          CREATE — disable devservices
persistence-memory/src/test/java/io/casehub/eidos/memory/
  InMemoryAgentRegistryTest.java  CREATE
  InMemoryReactiveAgentRegistryTest.java  CREATE

vocab/pom.xml                     MODIFY — add Jandex plugin execution
vocab/src/main/java/io/casehub/eidos/vocab/
  SvoVocabularyProducer.java      CREATE
  ConscientiousnessVocabularyProducer.java  CREATE
  CasehubSlotVocabularyProducer.java  CREATE

vocab/src/test/java/io/casehub/eidos/vocab/
  SvoVocabularyTest.java          CREATE
  ConscientiousnessVocabularyTest.java  CREATE
  CasehubSlotVocabularyTest.java  CREATE
```

---

## Task 1: API Types — AgentDescriptor, AgentQuery, SPI Amendments

**Files:**
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentDescriptor.java`
- Create: `api/src/main/java/io/casehub/eidos/api/AgentQuery.java`
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentRegistry.java`
- Modify: `api/src/main/java/io/casehub/eidos/api/ReactiveAgentRegistry.java`

- [ ] **Step 1: Add tenancyId to AgentDescriptor**

```java
// api/src/main/java/io/casehub/eidos/api/AgentDescriptor.java
package io.casehub.eidos.api;

import java.util.List;

public record AgentDescriptor(
        String agentId,
        String name,
        String version,
        String provider,
        String modelFamily,
        String modelVersion,
        String weightsFingerprint,
        String domainVocabulary,
        String slotVocabulary,
        String dispositionVocabulary,
        String slot,
        List<AgentCapability> capabilities,
        AgentDisposition disposition,
        String jurisdiction,
        String dataHandlingPolicy,
        String tenancyId
) {}
```

- [ ] **Step 2: Create AgentQuery**

```java
// api/src/main/java/io/casehub/eidos/api/AgentQuery.java
package io.casehub.eidos.api;

import java.util.Objects;

public record AgentQuery(
        String slot,
        String capabilityName,
        String tenancyId
) {
    public AgentQuery {
        Objects.requireNonNull(tenancyId, "tenancyId must not be null");
    }

    public static AgentQuery bySlot(String slot, String tenancyId) {
        return new AgentQuery(slot, null, tenancyId);
    }

    public static AgentQuery byCapability(String capabilityName, String tenancyId) {
        return new AgentQuery(null, capabilityName, tenancyId);
    }

    public static AgentQuery bySlotAndCapability(String slot, String capabilityName, String tenancyId) {
        return new AgentQuery(slot, capabilityName, tenancyId);
    }

    public static AgentQuery all(String tenancyId) {
        return new AgentQuery(null, null, tenancyId);
    }
}
```

- [ ] **Step 3: Amend AgentRegistry**

```java
// api/src/main/java/io/casehub/eidos/api/AgentRegistry.java
package io.casehub.eidos.api;

import java.util.List;
import java.util.Optional;

public interface AgentRegistry {
    void register(AgentDescriptor descriptor);
    Optional<AgentDescriptor> findById(String agentId);
    List<AgentDescriptor> find(AgentQuery query);
}
```

- [ ] **Step 4: Amend ReactiveAgentRegistry**

```java
// api/src/main/java/io/casehub/eidos/api/ReactiveAgentRegistry.java
package io.casehub.eidos.api;

import io.smallrye.mutiny.Uni;
import java.util.List;
import java.util.Optional;

public interface ReactiveAgentRegistry {
    Uni<Void> register(AgentDescriptor descriptor);
    Uni<Optional<AgentDescriptor>> findById(String agentId);
    Uni<List<AgentDescriptor>> find(AgentQuery query);
}
```

- [ ] **Step 5: Verify api module compiles**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api
```

Expected: `BUILD SUCCESS`

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add api/src/main/java/io/casehub/eidos/api/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(api): add AgentQuery, tenancyId on descriptor, collapse registry query methods

AgentQuery carries slot/capabilityName/tenancyId — satisfies auth-retrofit-readiness
Rule 4 (filter hook) and no-conditional-tenancy-filtering protocol.
find(AgentQuery) replaces findBySlot/findByCapability/findBySlotAndCapability.

Refs casehubio/eidos#1"
```

---

## Task 2: API Tests

**Files:**
- Create: `api/src/test/java/io/casehub/eidos/api/AgentQueryTest.java`
- Create: `api/src/test/java/io/casehub/eidos/api/AgentRegistrySpiTest.java`

- [ ] **Step 1: Write AgentQueryTest**

```java
// api/src/test/java/io/casehub/eidos/api/AgentQueryTest.java
package io.casehub.eidos.api;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

class AgentQueryTest {

    @Test
    void null_tenancy_id_throws() {
        assertThatNullPointerException()
            .isThrownBy(() -> new AgentQuery("reviewer", null, null))
            .withMessageContaining("tenancyId");
    }

    @Test
    void bySlot_sets_correct_fields() {
        var q = AgentQuery.bySlot("reviewer", "default");
        assertThat(q.slot()).isEqualTo("reviewer");
        assertThat(q.capabilityName()).isNull();
        assertThat(q.tenancyId()).isEqualTo("default");
    }

    @Test
    void byCapability_sets_correct_fields() {
        var q = AgentQuery.byCapability("code-review", "default");
        assertThat(q.slot()).isNull();
        assertThat(q.capabilityName()).isEqualTo("code-review");
        assertThat(q.tenancyId()).isEqualTo("default");
    }

    @Test
    void bySlotAndCapability_sets_all_fields() {
        var q = AgentQuery.bySlotAndCapability("reviewer", "code-review", "default");
        assertThat(q.slot()).isEqualTo("reviewer");
        assertThat(q.capabilityName()).isEqualTo("code-review");
        assertThat(q.tenancyId()).isEqualTo("default");
    }

    @Test
    void all_sets_tenancy_only() {
        var q = AgentQuery.all("default");
        assertThat(q.slot()).isNull();
        assertThat(q.capabilityName()).isNull();
        assertThat(q.tenancyId()).isEqualTo("default");
    }
}
```

- [ ] **Step 2: Write SPI contract test**

```java
// api/src/test/java/io/casehub/eidos/api/AgentRegistrySpiTest.java
package io.casehub.eidos.api;

import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Optional;
import static org.assertj.core.api.Assertions.*;

class AgentRegistrySpiTest {

    // Verifies AgentRegistry is a valid interface that anonymous impls can satisfy.
    // The compiler error on any missing method is the RED; this compiling GREEN proves the contract.
    @Test
    void anonymous_implementation_satisfies_contract() {
        AgentRegistry registry = new AgentRegistry() {
            @Override public void register(AgentDescriptor d) {}
            @Override public Optional<AgentDescriptor> findById(String id) { return Optional.empty(); }
            @Override public List<AgentDescriptor> find(AgentQuery q) { return List.of(); }
        };
        assertThat(registry.findById("x")).isEmpty();
        assertThat(registry.find(AgentQuery.all("default"))).isEmpty();
    }
}
```

- [ ] **Step 3: Run api tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api
```

Expected: `BUILD SUCCESS`, all tests green.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add api/src/test/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "test(api): AgentQueryTest and SPI contract test

Refs casehubio/eidos#1"
```

---

## Task 3: Pom Updates

**Files:**
- Modify: `runtime/pom.xml`
- Modify: `persistence-memory/pom.xml`
- Modify: `vocab/pom.xml`

- [ ] **Step 1: Add Hibernate Reactive and reactive PG client to runtime/pom.xml**

In `runtime/pom.xml`, add after the `quarkus-hibernate-orm` dependency and before the JDBC drivers block:

```xml
    <!-- Reactive JPA — activated when casehub.eidos.reactive.enabled=true -->
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-hibernate-reactive-panache</artifactId>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-reactive-pg-client</artifactId>
      <optional>true</optional>
    </dependency>
```

Also add ArchUnit to the runtime test deps:

```xml
    <dependency>
      <groupId>com.tngtech.archunit</groupId>
      <artifactId>archunit-junit5</artifactId>
      <version>1.4.1</version>
      <scope>test</scope>
    </dependency>
```

- [ ] **Step 2: Add Jandex plugin execution to persistence-memory/pom.xml**

In `persistence-memory/pom.xml`, add a `<build>` section after `</dependencies>`:

```xml
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
```

- [ ] **Step 3: Add Jandex plugin execution to vocab/pom.xml**

Same `<build>` block as Step 2, added to `vocab/pom.xml`.

- [ ] **Step 4: Verify pom changes compile**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime,persistence-memory,vocab
```

Expected: `BUILD SUCCESS`

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add runtime/pom.xml persistence-memory/pom.xml vocab/pom.xml
git -C /Users/mdproctor/claude/casehub/eidos commit -m "build: add hibernate-reactive-panache, reactive-pg-client, archunit; jandex to memory+vocab

Refs casehubio/eidos#1"
```

---

## Task 4: EidosBuildTimeConfig

**Files:**
- Create: `deployment/src/main/java/io/casehub/eidos/deployment/EidosBuildTimeConfig.java`
- Modify: `deployment/src/main/java/io/casehub/eidos/deployment/EidosProcessor.java`

- [ ] **Step 1: Create EidosBuildTimeConfig**

```java
// deployment/src/main/java/io/casehub/eidos/deployment/EidosBuildTimeConfig.java
package io.casehub.eidos.deployment;

import io.quarkus.runtime.annotations.ConfigPhase;
import io.quarkus.runtime.annotations.ConfigRoot;
import io.smallrye.config.ConfigMapping;
import io.smallrye.config.WithDefault;

@ConfigMapping(prefix = "casehub.eidos")
@ConfigRoot(phase = ConfigPhase.BUILD_TIME)
public interface EidosBuildTimeConfig {

    ReactiveConfig reactive();

    interface ReactiveConfig {
        /**
         * Whether to activate the reactive service tier.
         *
         * Set to {@code true} in deployments providing a reactive datasource
         * (Hibernate Reactive + reactive PostgreSQL client). JDBC-only consumers
         * must leave this unset — default {@code false} excludes all reactive beans.
         */
        @WithDefault("false")
        boolean enabled();
    }
}
```

- [ ] **Step 2: Reference EidosBuildTimeConfig in EidosProcessor so it is formally consumed**

```java
// deployment/src/main/java/io/casehub/eidos/deployment/EidosProcessor.java
package io.casehub.eidos.deployment;

import io.quarkus.deployment.annotations.BuildStep;
import io.quarkus.deployment.builditem.FeatureBuildItem;

class EidosProcessor {

    private static final String FEATURE = "eidos";

    @BuildStep
    FeatureBuildItem feature(EidosBuildTimeConfig config) {
        return new FeatureBuildItem(FEATURE);
    }
}
```

- [ ] **Step 3: Compile deployment module**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl deployment
```

Expected: `BUILD SUCCESS`

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add deployment/src/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(deployment): EidosBuildTimeConfig — casehub.eidos.reactive.enabled build property

Default false. Set to true in reactive deployments. Follows qhorus pattern.

Refs casehubio/eidos#1"
```

---

## Task 5: V1 Flyway Migration + Runtime Test Config

**Files:**
- Create: `runtime/src/main/resources/db/eidos/migration/V1__initial_schema.sql`
- Create: `runtime/src/test/resources/application.properties`

- [ ] **Step 1: Create V1 migration**

```sql
-- runtime/src/main/resources/db/eidos/migration/V1__initial_schema.sql

CREATE TABLE agent_descriptor (
    agent_id               VARCHAR(255)  NOT NULL PRIMARY KEY,
    tenancy_id             VARCHAR(255)  NOT NULL,
    name                   VARCHAR(255),
    version                VARCHAR(255),
    provider               VARCHAR(255),
    model_family           VARCHAR(255),
    model_version          VARCHAR(255),
    weights_fingerprint    VARCHAR(255),
    domain_vocabulary      VARCHAR(255),
    slot_vocabulary        VARCHAR(255),
    disposition_vocabulary VARCHAR(255),
    slot                   VARCHAR(255),
    jurisdiction           VARCHAR(255),
    data_handling_policy   TEXT,
    disposition            TEXT
);

CREATE TABLE agent_capability (
    id                  BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    agent_id            VARCHAR(255)  NOT NULL
                            REFERENCES agent_descriptor(agent_id) ON DELETE CASCADE,
    name                VARCHAR(255)  NOT NULL,
    quality_hint        DOUBLE PRECISION,
    latency_hint_p50_ms BIGINT,
    cost_hint           VARCHAR(255),
    input_types         TEXT,
    output_types        TEXT,
    tags                TEXT,
    epistemic_domains   TEXT
);

CREATE INDEX idx_agent_descriptor_tenancy_slot ON agent_descriptor(tenancy_id, slot);
CREATE INDEX idx_agent_capability_name         ON agent_capability(name);
```

- [ ] **Step 2: Create runtime test application.properties**

```properties
# runtime/src/test/resources/application.properties
quarkus.datasource.db-kind=h2
quarkus.datasource.jdbc.url=jdbc:h2:mem:eidostest;MODE=PostgreSQL;DB_CLOSE_DELAY=-1
quarkus.datasource.devservices.enabled=false
quarkus.flyway.locations=classpath:db/eidos/migration
quarkus.flyway.migrate-at-start=true
quarkus.hibernate-orm.database.generation=none

# Reactive profile — activated by ReactiveTestProfile
%reactive.casehub.eidos.reactive.enabled=true
%reactive.quarkus.datasource.db-kind=postgresql
%reactive.quarkus.datasource.devservices.enabled=true
%reactive.quarkus.datasource.devservices.image-name=postgres:17-alpine
%reactive.quarkus.flyway.locations=classpath:db/eidos/migration
%reactive.quarkus.flyway.migrate-at-start=true
%reactive.quarkus.hibernate-orm.database.generation=none
```

- [ ] **Step 3: Verify the migration path is reachable**

```bash
ls /Users/mdproctor/claude/casehub/eidos/runtime/src/main/resources/db/eidos/migration/
```

Expected: `V1__initial_schema.sql`

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add runtime/src/main/resources/ runtime/src/test/resources/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(runtime): V1 Flyway migration — agent_descriptor + agent_capability schema

Scoped path db/eidos/migration — extension does not set quarkus.flyway.locations.
Composite index on (tenancy_id, slot); index on capability name.
Test properties: H2 for blocking, PostgreSQL DevServices for reactive profile.

Refs casehubio/eidos#1"
```

---

## Task 6: JPA Entities

**Files:**
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentDescriptorEntity.java`
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentCapabilityEntity.java`

- [ ] **Step 1: Create AgentDescriptorEntity**

```java
// runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentDescriptorEntity.java
package io.casehub.eidos.runtime.registry.jpa;

import jakarta.persistence.*;
import java.util.ArrayList;
import java.util.List;

@Entity
@Table(name = "agent_descriptor")
class AgentDescriptorEntity {

    @Id
    @Column(name = "agent_id")
    String agentId;

    @Column(name = "tenancy_id", nullable = false)
    String tenancyId;

    String name;
    String version;
    String provider;

    @Column(name = "model_family")
    String modelFamily;

    @Column(name = "model_version")
    String modelVersion;

    @Column(name = "weights_fingerprint")
    String weightsFingerprint;

    @Column(name = "domain_vocabulary")
    String domainVocabulary;

    @Column(name = "slot_vocabulary")
    String slotVocabulary;

    @Column(name = "disposition_vocabulary")
    String dispositionVocabulary;

    String slot;
    String jurisdiction;

    @Column(name = "data_handling_policy", columnDefinition = "TEXT")
    String dataHandlingPolicy;

    @Column(columnDefinition = "TEXT")
    String disposition;

    @OneToMany(mappedBy = "descriptor", cascade = CascadeType.ALL,
               fetch = FetchType.LAZY, orphanRemoval = true)
    List<AgentCapabilityEntity> capabilities = new ArrayList<>();
}
```

- [ ] **Step 2: Create AgentCapabilityEntity**

```java
// runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentCapabilityEntity.java
package io.casehub.eidos.runtime.registry.jpa;

import jakarta.persistence.*;

@Entity
@Table(name = "agent_capability")
class AgentCapabilityEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "agent_id", nullable = false)
    AgentDescriptorEntity descriptor;

    @Column(nullable = false)
    String name;

    @Column(name = "quality_hint")
    double qualityHint;

    @Column(name = "latency_hint_p50_ms")
    Long latencyHintP50Ms;

    @Column(name = "cost_hint")
    String costHint;

    @Column(name = "input_types", columnDefinition = "TEXT")
    String inputTypes;

    @Column(name = "output_types", columnDefinition = "TEXT")
    String outputTypes;

    @Column(columnDefinition = "TEXT")
    String tags;

    @Column(name = "epistemic_domains", columnDefinition = "TEXT")
    String epistemicDomains;
}
```

- [ ] **Step 3: Compile**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime
```

Expected: `BUILD SUCCESS`

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(runtime): JPA entities — AgentDescriptorEntity, AgentCapabilityEntity

FetchType.LAZY on @OneToMany — all queries use explicit JOIN FETCH.
TEXT for JSON columns (portable H2+PostgreSQL); CASCADE.ALL + orphanRemoval.

Refs casehubio/eidos#1"
```

---

## Task 7: AgentDescriptorMapper

**Files:**
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentDescriptorMapper.java`

- [ ] **Step 1: Create AgentDescriptorMapper**

```java
// runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentDescriptorMapper.java
package io.casehub.eidos.runtime.registry.jpa;

import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.eidos.api.*;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.List;
import java.util.Map;

@ApplicationScoped
class AgentDescriptorMapper {

    @Inject
    ObjectMapper mapper;

    AgentDescriptor toRecord(AgentDescriptorEntity e) {
        return new AgentDescriptor(
            e.agentId, e.name, e.version, e.provider,
            e.modelFamily, e.modelVersion, e.weightsFingerprint,
            e.domainVocabulary, e.slotVocabulary, e.dispositionVocabulary,
            e.slot,
            e.capabilities.stream().map(this::toCapability).toList(),
            readJson(e.disposition, AgentDisposition.class),
            e.jurisdiction, e.dataHandlingPolicy, e.tenancyId
        );
    }

    AgentDescriptorEntity toEntity(AgentDescriptor d) {
        var e = new AgentDescriptorEntity();
        e.agentId = d.agentId();
        e.tenancyId = d.tenancyId();
        e.name = d.name();
        e.version = d.version();
        e.provider = d.provider();
        e.modelFamily = d.modelFamily();
        e.modelVersion = d.modelVersion();
        e.weightsFingerprint = d.weightsFingerprint();
        e.domainVocabulary = d.domainVocabulary();
        e.slotVocabulary = d.slotVocabulary();
        e.dispositionVocabulary = d.dispositionVocabulary();
        e.slot = d.slot();
        e.jurisdiction = d.jurisdiction();
        e.dataHandlingPolicy = d.dataHandlingPolicy();
        e.disposition = writeJson(d.disposition());
        d.capabilities().stream()
            .map(c -> toCapabilityEntity(c, e))
            .forEach(e.capabilities::add);
        return e;
    }

    private AgentCapability toCapability(AgentCapabilityEntity c) {
        return new AgentCapability(
            c.name, c.qualityHint, c.latencyHintP50Ms, c.costHint,
            readJson(c.inputTypes, new TypeReference<List<String>>() {}),
            readJson(c.outputTypes, new TypeReference<List<String>>() {}),
            readJson(c.tags, new TypeReference<List<String>>() {}),
            readJson(c.epistemicDomains, new TypeReference<Map<String, Double>>() {})
        );
    }

    private AgentCapabilityEntity toCapabilityEntity(AgentCapability c, AgentDescriptorEntity parent) {
        var e = new AgentCapabilityEntity();
        e.descriptor = parent;
        e.name = c.name();
        e.qualityHint = c.qualityHint();
        e.latencyHintP50Ms = c.latencyHintP50Ms();
        e.costHint = c.costHint();
        e.inputTypes = writeJson(c.inputTypes());
        e.outputTypes = writeJson(c.outputTypes());
        e.tags = writeJson(c.tags());
        e.epistemicDomains = writeJson(c.epistemicDomains());
        return e;
    }

    private <T> T readJson(String json, Class<T> type) {
        if (json == null) return null;
        try { return mapper.readValue(json, type); }
        catch (Exception ex) { throw new RuntimeException("JSON deserialisation failed", ex); }
    }

    private <T> T readJson(String json, TypeReference<T> type) {
        if (json == null) return null;
        try { return mapper.readValue(json, type); }
        catch (Exception ex) { throw new RuntimeException("JSON deserialisation failed", ex); }
    }

    private String writeJson(Object obj) {
        if (obj == null) return null;
        try { return mapper.writeValueAsString(obj); }
        catch (Exception ex) { throw new RuntimeException("JSON serialisation failed", ex); }
    }
}
```

- [ ] **Step 2: Compile**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime
```

Expected: `BUILD SUCCESS`

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentDescriptorMapper.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(runtime): AgentDescriptorMapper — entity ↔ record conversion via Jackson

Jackson handles all JSON fields. Null-safe read/write. toEntity() builds
capability entities with parent reference set.

Refs casehubio/eidos#1"
```

---

## Task 8: JpaAgentRegistry — TDD

**Files:**
- Create: `runtime/src/test/java/io/casehub/eidos/runtime/registry/JpaAgentRegistryTest.java`
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/registry/JpaAgentRegistry.java`

- [ ] **Step 1: Write JpaAgentRegistryTest (failing)**

```java
// runtime/src/test/java/io/casehub/eidos/runtime/registry/JpaAgentRegistryTest.java
package io.casehub.eidos.runtime.registry;

import io.casehub.eidos.api.*;
import io.quarkus.test.TestTransaction;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.util.Arrays;
import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.*;

@QuarkusTest
class JpaAgentRegistryTest {

    @Inject
    AgentRegistry registry;

    static AgentDescriptor descriptor(String agentId, String slot, String tenancyId,
                                      String... capabilityNames) {
        var caps = Arrays.stream(capabilityNames)
            .map(n -> new AgentCapability(n, 0.9, null, null,
                List.of(), List.of(), List.of(), Map.of()))
            .toList();
        return new AgentDescriptor(
            agentId, "Agent " + agentId, "1.0", "anthropic",
            "claude", "claude-3-7", null,
            null, null, null,
            slot, caps,
            new AgentDisposition("collaborative", "principled", "measured", "semi-autonomous", false),
            null, null, tenancyId
        );
    }

    @Test
    @TestTransaction
    void register_and_find_by_id() {
        registry.register(descriptor("agent-1", "reviewer", "default", "code-review"));

        var found = registry.findById("agent-1");

        assertThat(found).isPresent();
        assertThat(found.get().agentId()).isEqualTo("agent-1");
        assertThat(found.get().slot()).isEqualTo("reviewer");
        assertThat(found.get().tenancyId()).isEqualTo("default");
        assertThat(found.get().capabilities()).hasSize(1);
        assertThat(found.get().capabilities().get(0).name()).isEqualTo("code-review");
    }

    @Test
    @TestTransaction
    void find_by_slot_returns_matching_agents_only() {
        registry.register(descriptor("agent-2a", "reviewer", "default", "code-review"));
        registry.register(descriptor("agent-2b", "planner", "default", "planning"));

        var reviewers = registry.find(AgentQuery.bySlot("reviewer", "default"));

        assertThat(reviewers).hasSize(1);
        assertThat(reviewers.get(0).agentId()).isEqualTo("agent-2a");
    }

    @Test
    @TestTransaction
    void find_by_capability_returns_agents_with_that_capability() {
        registry.register(descriptor("agent-3a", "reviewer", "default", "code-review", "test-writing"));
        registry.register(descriptor("agent-3b", "executor", "default", "test-writing"));

        var codeReviewers = registry.find(AgentQuery.byCapability("code-review", "default"));

        assertThat(codeReviewers).hasSize(1);
        assertThat(codeReviewers.get(0).agentId()).isEqualTo("agent-3a");
    }

    @Test
    @TestTransaction
    void find_by_slot_and_capability_applies_both_filters() {
        registry.register(descriptor("agent-4a", "reviewer", "default", "code-review"));
        registry.register(descriptor("agent-4b", "executor", "default", "code-review"));

        var result = registry.find(AgentQuery.bySlotAndCapability("reviewer", "code-review", "default"));

        assertThat(result).hasSize(1);
        assertThat(result.get(0).agentId()).isEqualTo("agent-4a");
    }

    @Test
    @TestTransaction
    void register_upserts_existing_agent() {
        registry.register(descriptor("agent-5", "reviewer", "default", "code-review"));
        registry.register(descriptor("agent-5", "planner", "default", "planning"));

        var found = registry.findById("agent-5");

        assertThat(found).isPresent();
        assertThat(found.get().slot()).isEqualTo("planner");
        assertThat(found.get().capabilities()).hasSize(1);
        assertThat(found.get().capabilities().get(0).name()).isEqualTo("planning");
    }

    @Test
    @TestTransaction
    void tenancy_isolation_excludes_other_tenant_agents() {
        registry.register(descriptor("agent-6", "reviewer", "tenant-a", "code-review"));

        var result = registry.find(AgentQuery.bySlot("reviewer", "tenant-b"));

        assertThat(result).isEmpty();
    }

    @Test
    @TestTransaction
    void find_all_returns_only_own_tenant() {
        registry.register(descriptor("agent-7a", "reviewer", "tenant-a", "code-review"));
        registry.register(descriptor("agent-7b", "planner", "tenant-b", "planning"));

        var tenantA = registry.find(AgentQuery.all("tenant-a"));

        assertThat(tenantA).hasSize(1);
        assertThat(tenantA.get(0).agentId()).isEqualTo("agent-7a");
    }

    @Test
    @TestTransaction
    void find_by_id_returns_empty_for_missing_agent() {
        assertThat(registry.findById("nonexistent")).isEmpty();
    }
}
```

- [ ] **Step 2: Run tests — expect compile failure (JpaAgentRegistry doesn't exist yet)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime 2>&1 | grep -E "ERROR|FAIL|BUILD"
```

Expected: compile error or CDI unsatisfied dependency.

- [ ] **Step 3: Create JpaAgentRegistry**

```java
// runtime/src/main/java/io/casehub/eidos/runtime/registry/JpaAgentRegistry.java
package io.casehub.eidos.runtime.registry;

import io.casehub.eidos.api.*;
import io.casehub.eidos.runtime.registry.jpa.AgentDescriptorEntity;
import io.casehub.eidos.runtime.registry.jpa.AgentDescriptorMapper;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.transaction.Transactional;
import java.util.List;
import java.util.Optional;

@ApplicationScoped
public class JpaAgentRegistry implements AgentRegistry {

    @Inject EntityManager em;
    @Inject AgentDescriptorMapper mapper;

    @Override
    @Transactional
    public void register(AgentDescriptor descriptor) {
        em.createQuery("DELETE FROM AgentDescriptorEntity e WHERE e.agentId = :id")
          .setParameter("id", descriptor.agentId())
          .executeUpdate();
        em.persist(mapper.toEntity(descriptor));
    }

    @Override
    public Optional<AgentDescriptor> findById(String agentId) {
        return em.createQuery(
                "SELECT DISTINCT a FROM AgentDescriptorEntity a LEFT JOIN FETCH a.capabilities"
                + " WHERE a.agentId = :id",
                AgentDescriptorEntity.class)
            .setParameter("id", agentId)
            .getResultStream()
            .findFirst()
            .map(mapper::toRecord);
    }

    @Override
    public List<AgentDescriptor> find(AgentQuery query) {
        String fetchJoin = query.capabilityName() != null
            ? "JOIN FETCH a.capabilities c"
            : "LEFT JOIN FETCH a.capabilities c";

        var jpql = new StringBuilder(
            "SELECT DISTINCT a FROM AgentDescriptorEntity a " + fetchJoin
            + " WHERE a.tenancyId = :tenancyId");
        if (query.slot() != null) jpql.append(" AND a.slot = :slot");
        if (query.capabilityName() != null) jpql.append(" AND c.name = :capabilityName");

        var q = em.createQuery(jpql.toString(), AgentDescriptorEntity.class)
                  .setParameter("tenancyId", query.tenancyId());
        if (query.slot() != null) q.setParameter("slot", query.slot());
        if (query.capabilityName() != null) q.setParameter("capabilityName", query.capabilityName());

        return q.getResultList().stream().map(mapper::toRecord).toList();
    }
}
```

- [ ] **Step 4: Run tests — expect GREEN**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=JpaAgentRegistryTest
```

Expected: `Tests run: 8, Failures: 0, Errors: 0, Skipped: 0`

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add runtime/src/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(runtime): JpaAgentRegistry — blocking JPA implementation with tenancy isolation

find(AgentQuery) builds JPQL dynamically. tenancyId always applied unconditionally.
JOIN FETCH vs LEFT JOIN FETCH based on whether capability filter is present.
Upsert: bulk delete + persist (ON DELETE CASCADE handles capability rows).

Refs casehubio/eidos#1"
```

---

## Task 9: JpaReactiveAgentRegistry — TDD

**Files:**
- Create: `runtime/src/test/java/io/casehub/eidos/runtime/registry/ReactiveTestProfile.java`
- Create: `runtime/src/test/java/io/casehub/eidos/runtime/registry/JpaReactiveAgentRegistryTest.java`
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentDescriptorReactivePanacheRepo.java`
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/registry/JpaReactiveAgentRegistry.java`

- [ ] **Step 1: Create ReactiveTestProfile**

```java
// runtime/src/test/java/io/casehub/eidos/runtime/registry/ReactiveTestProfile.java
package io.casehub.eidos.runtime.registry;

import io.quarkus.test.junit.QuarkusTestProfile;

public class ReactiveTestProfile implements QuarkusTestProfile {
    @Override
    public String getConfigProfile() {
        return "reactive";
    }
}
```

- [ ] **Step 2: Write JpaReactiveAgentRegistryTest (failing)**

```java
// runtime/src/test/java/io/casehub/eidos/runtime/registry/JpaReactiveAgentRegistryTest.java
package io.casehub.eidos.runtime.registry;

import io.casehub.eidos.api.*;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.TestProfile;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.util.Arrays;
import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.*;

@QuarkusTest
@TestProfile(ReactiveTestProfile.class)
class JpaReactiveAgentRegistryTest {

    @Inject
    ReactiveAgentRegistry registry;

    static AgentDescriptor descriptor(String agentId, String slot, String tenancyId,
                                      String... capabilityNames) {
        var caps = Arrays.stream(capabilityNames)
            .map(n -> new AgentCapability(n, 0.9, null, null,
                List.of(), List.of(), List.of(), Map.of()))
            .toList();
        return new AgentDescriptor(
            agentId, "Agent " + agentId, "1.0", "anthropic",
            "claude", "claude-3-7", null,
            null, null, null,
            slot, caps,
            new AgentDisposition("collaborative", "principled", "measured", "semi-autonomous", false),
            null, null, tenancyId
        );
    }

    @Test
    void reactive_register_and_find_by_id() {
        registry.register(descriptor("r-agent-1", "reviewer", "default", "code-review"))
            .await().indefinitely();

        var found = registry.findById("r-agent-1").await().indefinitely();

        assertThat(found).isPresent();
        assertThat(found.get().slot()).isEqualTo("reviewer");
        assertThat(found.get().tenancyId()).isEqualTo("default");
    }

    @Test
    void reactive_find_by_slot() {
        registry.register(descriptor("r-agent-2a", "reviewer", "default", "code-review"))
            .await().indefinitely();
        registry.register(descriptor("r-agent-2b", "planner", "default", "planning"))
            .await().indefinitely();

        var result = registry.find(AgentQuery.bySlot("reviewer", "default")).await().indefinitely();

        assertThat(result).hasSize(1);
        assertThat(result.get(0).agentId()).isEqualTo("r-agent-2a");
    }

    @Test
    void reactive_find_by_capability() {
        registry.register(descriptor("r-agent-3", "reviewer", "default", "code-review"))
            .await().indefinitely();

        var result = registry.find(AgentQuery.byCapability("code-review", "default"))
            .await().indefinitely();

        assertThat(result).hasSize(1);
    }

    @Test
    void reactive_tenancy_isolation() {
        registry.register(descriptor("r-agent-4", "reviewer", "tenant-a", "code-review"))
            .await().indefinitely();

        var result = registry.find(AgentQuery.bySlot("reviewer", "tenant-b"))
            .await().indefinitely();

        assertThat(result).isEmpty();
    }

    @Test
    void reactive_upsert() {
        registry.register(descriptor("r-agent-5", "reviewer", "default", "code-review"))
            .await().indefinitely();
        registry.register(descriptor("r-agent-5", "planner", "default", "planning"))
            .await().indefinitely();

        var found = registry.findById("r-agent-5").await().indefinitely();

        assertThat(found).isPresent();
        assertThat(found.get().slot()).isEqualTo("planner");
    }
}
```

- [ ] **Step 3: Create AgentDescriptorReactivePanacheRepo**

```java
// runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentDescriptorReactivePanacheRepo.java
package io.casehub.eidos.runtime.registry.jpa;

import io.quarkus.arc.properties.IfBuildProperty;
import io.quarkus.hibernate.reactive.panache.PanacheRepositoryBase;
import jakarta.enterprise.context.ApplicationScoped;

@IfBuildProperty(name = "casehub.eidos.reactive.enabled", stringValue = "true")
@ApplicationScoped
public class AgentDescriptorReactivePanacheRepo
        implements PanacheRepositoryBase<AgentDescriptorEntity, String> {
}
```

- [ ] **Step 4: Create JpaReactiveAgentRegistry**

```java
// runtime/src/main/java/io/casehub/eidos/runtime/registry/JpaReactiveAgentRegistry.java
package io.casehub.eidos.runtime.registry;

import io.casehub.eidos.api.*;
import io.casehub.eidos.runtime.registry.jpa.AgentDescriptorMapper;
import io.casehub.eidos.runtime.registry.jpa.AgentDescriptorReactivePanacheRepo;
import io.quarkus.arc.properties.IfBuildProperty;
import io.quarkus.hibernate.reactive.panache.common.WithTransaction;
import io.quarkus.panache.common.Parameters;
import io.smallrye.mutiny.Uni;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.List;
import java.util.Optional;

@IfBuildProperty(name = "casehub.eidos.reactive.enabled", stringValue = "true")
@ApplicationScoped
public class JpaReactiveAgentRegistry implements ReactiveAgentRegistry {

    @Inject AgentDescriptorReactivePanacheRepo repo;
    @Inject AgentDescriptorMapper mapper;

    @Override
    @WithTransaction
    public Uni<Void> register(AgentDescriptor descriptor) {
        return repo.delete("agentId", descriptor.agentId())
                   .chain(() -> repo.persist(mapper.toEntity(descriptor)))
                   .replaceWithVoid();
    }

    @Override
    public Uni<Optional<AgentDescriptor>> findById(String agentId) {
        return repo.find(
                "SELECT DISTINCT a FROM AgentDescriptorEntity a"
                + " LEFT JOIN FETCH a.capabilities WHERE a.agentId = ?1", agentId)
            .firstResult()
            .map(e -> Optional.ofNullable(e).map(mapper::toRecord));
    }

    @Override
    public Uni<List<AgentDescriptor>> find(AgentQuery query) {
        String fetchJoin = query.capabilityName() != null
            ? "JOIN FETCH a.capabilities c"
            : "LEFT JOIN FETCH a.capabilities c";

        var jpql = new StringBuilder(
            "SELECT DISTINCT a FROM AgentDescriptorEntity a " + fetchJoin
            + " WHERE a.tenancyId = :tenancyId");

        var params = Parameters.with("tenancyId", query.tenancyId());
        if (query.slot() != null) {
            jpql.append(" AND a.slot = :slot");
            params.and("slot", query.slot());
        }
        if (query.capabilityName() != null) {
            jpql.append(" AND c.name = :capabilityName");
            params.and("capabilityName", query.capabilityName());
        }

        return repo.list(jpql.toString(), params)
                   .map(list -> list.stream().map(mapper::toRecord).toList());
    }
}
```

- [ ] **Step 5: Run reactive tests (requires Docker)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=JpaReactiveAgentRegistryTest
```

Expected: `Tests run: 5, Failures: 0, Errors: 0, Skipped: 0`
Note: DevServices starts a PostgreSQL container automatically. First run is slow (~30s).

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add runtime/src/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(runtime): JpaReactiveAgentRegistry — Hibernate Reactive, build-gated

@IfBuildProperty(casehub.eidos.reactive.enabled=true). @WithTransaction on writes.
AgentDescriptorReactivePanacheRepo for JPQL queries. Parameters builder for
named params in dynamic queries. Same tenancy-always-applied guarantee as blocking.

Refs casehubio/eidos#1"
```

---

## Task 10: NoOpCapabilityHealth — TDD

**Files:**
- Create: `runtime/src/test/java/io/casehub/eidos/runtime/health/NoOpCapabilityHealthTest.java`
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/health/NoOpCapabilityHealth.java`

- [ ] **Step 1: Write test**

```java
// runtime/src/test/java/io/casehub/eidos/runtime/health/NoOpCapabilityHealthTest.java
package io.casehub.eidos.runtime.health;

import io.casehub.eidos.api.CapabilityHealth;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.*;

@QuarkusTest
class NoOpCapabilityHealthTest {

    @Inject
    CapabilityHealth health;

    @Test
    void probe_always_returns_ready() {
        var status = health.probe("any-agent", "any-capability",
            CapabilityHealth.ProbeContext.of("any-domain"));

        assertThat(status).isInstanceOf(CapabilityHealth.CapabilityStatus.Ready.class);
    }
}
```

- [ ] **Step 2: Create NoOpCapabilityHealth**

```java
// runtime/src/main/java/io/casehub/eidos/runtime/health/NoOpCapabilityHealth.java
package io.casehub.eidos.runtime.health;

import io.casehub.eidos.api.CapabilityHealth;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;

@DefaultBean
@ApplicationScoped
public class NoOpCapabilityHealth implements CapabilityHealth {

    @Override
    public CapabilityStatus probe(String agentId, String capabilityTag, ProbeContext context) {
        return new CapabilityStatus.Ready();
    }
}
```

- [ ] **Step 3: Run test**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=NoOpCapabilityHealthTest
```

Expected: `Tests run: 1, Failures: 0, Errors: 0`

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add runtime/src/main/java/io/casehub/eidos/runtime/health/ runtime/src/test/java/io/casehub/eidos/runtime/health/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(runtime): NoOpCapabilityHealth — @DefaultBean, always Ready

Phase 2 replaces with real probe logic against declared capabilities.

Refs casehubio/eidos#1"
```

---

## Task 11: CdiVocabularyRegistry — TDD

**Files:**
- Create: `runtime/src/test/java/io/casehub/eidos/runtime/vocabulary/CdiVocabularyRegistryTest.java`
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/vocabulary/CdiVocabularyRegistry.java`

- [ ] **Step 1: Write CdiVocabularyRegistryTest**

```java
// runtime/src/test/java/io/casehub/eidos/runtime/vocabulary/CdiVocabularyRegistryTest.java
package io.casehub.eidos.runtime.vocabulary;

import io.casehub.eidos.api.*;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.*;

@QuarkusTest
class CdiVocabularyRegistryTest {

    @Inject
    VocabularyRegistry registry;

    static Vocabulary testVocab(String uri) {
        return new Vocabulary(uri, "Test Vocab", "1.0", Map.of(
            "alpha", new VocabularyTerm("alpha", "Alpha", "First", List.of("a", "one"),
                Map.of("urn:other", "primary")),
            "beta",  new VocabularyTerm("beta",  "Beta",  "Second", List.of("b"),
                Map.of("urn:other", "secondary"))
        ));
    }

    @Test
    void programmatic_register_and_find() {
        var vocab = testVocab("urn:test:prog");
        registry.register(vocab);

        var found = registry.find("urn:test:prog");

        assertThat(found).isPresent();
        assertThat(found.get().name()).isEqualTo("Test Vocab");
    }

    @Test
    void find_returns_empty_for_unknown_uri() {
        assertThat(registry.find("urn:does-not-exist")).isEmpty();
    }

    @Test
    void resolve_by_exact_value() {
        registry.register(testVocab("urn:test:resolve"));

        var term = registry.resolve("urn:test:resolve", "alpha");

        assertThat(term).isPresent();
        assertThat(term.get().value()).isEqualTo("alpha");
    }

    @Test
    void resolve_by_alias() {
        registry.register(testVocab("urn:test:alias"));

        var term = registry.resolve("urn:test:alias", "one");

        assertThat(term).isPresent();
        assertThat(term.get().value()).isEqualTo("alpha");
    }

    @Test
    void resolve_returns_empty_for_unknown_value() {
        registry.register(testVocab("urn:test:miss"));

        assertThat(registry.resolve("urn:test:miss", "gamma")).isEmpty();
    }

    @Test
    void equivalent_values_returns_cross_vocab_match() {
        registry.register(testVocab("urn:test:equiv"));

        var equiv = registry.equivalentValues("urn:test:equiv", "alpha", "urn:other");

        assertThat(equiv).containsExactly("primary");
    }

    @Test
    void equivalent_values_via_alias_match() {
        registry.register(testVocab("urn:test:alias-equiv"));

        var equiv = registry.equivalentValues("urn:test:alias-equiv", "one", "urn:other");

        assertThat(equiv).containsExactly("primary");
    }

    @Test
    void equivalent_values_empty_for_no_match() {
        registry.register(testVocab("urn:test:no-match"));

        assertThat(registry.equivalentValues("urn:test:no-match", "alpha", "urn:nonexistent"))
            .isEmpty();
    }

    @Test
    void programmatic_register_overrides_existing() {
        registry.register(testVocab("urn:test:override"));
        registry.register(new Vocabulary("urn:test:override", "Updated", "2.0", Map.of()));

        assertThat(registry.find("urn:test:override").get().name()).isEqualTo("Updated");
    }
}
```

- [ ] **Step 2: Create CdiVocabularyRegistry**

```java
// runtime/src/main/java/io/casehub/eidos/runtime/vocabulary/CdiVocabularyRegistry.java
package io.casehub.eidos.runtime.vocabulary;

import io.casehub.eidos.api.*;
import io.quarkus.arc.DefaultBean;
import jakarta.annotation.PostConstruct;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Any;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import java.util.Objects;
import java.util.Optional;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;
import java.util.stream.Collectors;

@DefaultBean
@ApplicationScoped
public class CdiVocabularyRegistry implements VocabularyRegistry {

    @Inject @Any Instance<Vocabulary> cdiBeans;

    private final ConcurrentHashMap<String, Vocabulary> registry = new ConcurrentHashMap<>();

    @PostConstruct
    void init() {
        for (Vocabulary v : cdiBeans) {
            registry.put(v.uri(), v);
        }
    }

    @Override
    public void register(Vocabulary vocabulary) {
        registry.put(vocabulary.uri(), vocabulary);
    }

    @Override
    public Optional<Vocabulary> find(String uri) {
        return Optional.ofNullable(registry.get(uri));
    }

    @Override
    public Optional<VocabularyTerm> resolve(String vocabularyUri, String value) {
        return find(vocabularyUri)
            .flatMap(vocab -> vocab.terms().values().stream()
                .filter(t -> t.value().equals(value) || t.aliases().contains(value))
                .findFirst());
    }

    @Override
    public Set<String> equivalentValues(String vocabularyUri, String value, String targetVocabularyUri) {
        return find(vocabularyUri)
            .map(vocab -> vocab.terms().values().stream()
                .filter(t -> t.value().equals(value) || t.aliases().contains(value))
                .map(t -> t.exactMatches().get(targetVocabularyUri))
                .filter(Objects::nonNull)
                .collect(Collectors.toSet()))
            .orElse(Set.of());
    }
}
```

- [ ] **Step 3: Run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=CdiVocabularyRegistryTest
```

Expected: `Tests run: 9, Failures: 0, Errors: 0`

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add runtime/src/main/java/io/casehub/eidos/runtime/vocabulary/ runtime/src/test/java/io/casehub/eidos/runtime/vocabulary/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(runtime): CdiVocabularyRegistry — @DefaultBean, CDI discovery + programmatic map

@PostConstruct loads all CDI Vocabulary beans. ConcurrentHashMap for thread safety.
equivalentValues() collects across all matching terms (value or alias).

Refs casehubio/eidos#1"
```

---

## Task 12: InMemoryAgentRegistry — TDD

**Files:**
- Create: `persistence-memory/src/test/resources/application.properties`
- Create: `persistence-memory/src/test/java/io/casehub/eidos/memory/InMemoryAgentRegistryTest.java`
- Create: `persistence-memory/src/main/java/io/casehub/eidos/memory/InMemoryAgentRegistry.java`

- [ ] **Step 1: Create persistence-memory test application.properties**

```properties
# persistence-memory/src/test/resources/application.properties
quarkus.datasource.devservices.enabled=false
```

- [ ] **Step 2: Write InMemoryAgentRegistryTest**

```java
// persistence-memory/src/test/java/io/casehub/eidos/memory/InMemoryAgentRegistryTest.java
package io.casehub.eidos.memory;

import io.casehub.eidos.api.*;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.*;

@QuarkusTest
class InMemoryAgentRegistryTest {

    @Inject AgentRegistry registry;

    static AgentDescriptor descriptor(String agentId, String slot, String tenancyId,
                                      String... caps) {
        var capabilities = java.util.Arrays.stream(caps)
            .map(n -> new AgentCapability(n, 0.9, null, null,
                List.of(), List.of(), List.of(), Map.of()))
            .toList();
        return new AgentDescriptor(
            agentId, "Agent", "1.0", "anthropic", "claude", "claude-3-7",
            null, null, null, null, slot, capabilities,
            new AgentDisposition("collaborative", "principled", "measured", "semi-autonomous", false),
            null, null, tenancyId
        );
    }

    @BeforeEach
    void clearStore() {
        // InMemoryAgentRegistry is @ApplicationScoped — state persists between tests.
        // Register and then overwrite with known state by using unique IDs per test.
    }

    @Test
    void register_and_find_by_id() {
        registry.register(descriptor("m-1", "reviewer", "default", "code-review"));
        var found = registry.findById("m-1");
        assertThat(found).isPresent();
        assertThat(found.get().slot()).isEqualTo("reviewer");
        assertThat(found.get().tenancyId()).isEqualTo("default");
    }

    @Test
    void find_by_slot() {
        registry.register(descriptor("m-2a", "reviewer", "default", "code-review"));
        registry.register(descriptor("m-2b", "planner", "default", "planning"));
        var result = registry.find(AgentQuery.bySlot("reviewer", "default"));
        assertThat(result.stream().map(AgentDescriptor::agentId).toList())
            .contains("m-2a").doesNotContain("m-2b");
    }

    @Test
    void find_by_capability() {
        registry.register(descriptor("m-3a", "reviewer", "default", "code-review"));
        registry.register(descriptor("m-3b", "executor", "default", "testing"));
        var result = registry.find(AgentQuery.byCapability("code-review", "default"));
        assertThat(result.stream().map(AgentDescriptor::agentId).toList())
            .contains("m-3a").doesNotContain("m-3b");
    }

    @Test
    void find_by_slot_and_capability() {
        registry.register(descriptor("m-4a", "reviewer", "default", "code-review"));
        registry.register(descriptor("m-4b", "executor", "default", "code-review"));
        var result = registry.find(AgentQuery.bySlotAndCapability("reviewer", "code-review", "default"));
        assertThat(result.stream().map(AgentDescriptor::agentId).toList())
            .contains("m-4a").doesNotContain("m-4b");
    }

    @Test
    void tenancy_isolation() {
        registry.register(descriptor("m-5", "reviewer", "tenant-a", "code-review"));
        assertThat(registry.find(AgentQuery.bySlot("reviewer", "tenant-b"))).isEmpty();
    }

    @Test
    void upsert_replaces_existing() {
        registry.register(descriptor("m-6", "reviewer", "default", "code-review"));
        registry.register(descriptor("m-6", "planner", "default", "planning"));
        assertThat(registry.findById("m-6").get().slot()).isEqualTo("planner");
    }
}
```

- [ ] **Step 3: Create InMemoryAgentRegistry**

```java
// persistence-memory/src/main/java/io/casehub/eidos/memory/InMemoryAgentRegistry.java
package io.casehub.eidos.memory;

import io.casehub.eidos.api.*;
import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import java.util.List;
import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;
import java.util.stream.Collectors;

@Alternative
@Priority(1)
@ApplicationScoped
public class InMemoryAgentRegistry implements AgentRegistry {

    private final ConcurrentHashMap<String, AgentDescriptor> store = new ConcurrentHashMap<>();

    @Override
    public void register(AgentDescriptor descriptor) {
        store.put(descriptor.agentId(), descriptor);
    }

    @Override
    public Optional<AgentDescriptor> findById(String agentId) {
        return Optional.ofNullable(store.get(agentId));
    }

    @Override
    public List<AgentDescriptor> find(AgentQuery query) {
        return store.values().stream()
            .filter(d -> d.tenancyId().equals(query.tenancyId()))
            .filter(d -> query.slot() == null || d.slot().equals(query.slot()))
            .filter(d -> query.capabilityName() == null
                || d.capabilities().stream().anyMatch(c -> c.name().equals(query.capabilityName())))
            .collect(Collectors.toList());
    }
}
```

- [ ] **Step 4: Run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory -Dtest=InMemoryAgentRegistryTest
```

Expected: `Tests run: 6, Failures: 0, Errors: 0`

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add persistence-memory/src/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(persistence-memory): InMemoryAgentRegistry — @Alternative @Priority(1), ConcurrentHashMap

Zero datasource. find() applies tenancyId unconditionally. Upsert via put().

Refs casehubio/eidos#1"
```

---

## Task 13: InMemoryReactiveAgentRegistry — TDD

**Files:**
- Create: `persistence-memory/src/test/java/io/casehub/eidos/memory/InMemoryReactiveAgentRegistryTest.java`
- Create: `persistence-memory/src/main/java/io/casehub/eidos/memory/InMemoryReactiveAgentRegistry.java`

- [ ] **Step 1: Write InMemoryReactiveAgentRegistryTest**

```java
// persistence-memory/src/test/java/io/casehub/eidos/memory/InMemoryReactiveAgentRegistryTest.java
package io.casehub.eidos.memory;

import io.casehub.eidos.api.*;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.*;

@QuarkusTest
class InMemoryReactiveAgentRegistryTest {

    @Inject ReactiveAgentRegistry registry;

    static AgentDescriptor descriptor(String agentId, String slot, String tenancyId) {
        return new AgentDescriptor(
            agentId, "Agent", "1.0", "anthropic", "claude", "claude-3-7",
            null, null, null, null, slot,
            List.of(new AgentCapability("cap", 0.9, null, null,
                List.of(), List.of(), List.of(), Map.of())),
            new AgentDisposition("collaborative", "principled", "measured", "semi-autonomous", false),
            null, null, tenancyId
        );
    }

    @Test
    void register_and_find_by_id() {
        registry.register(descriptor("rm-1", "reviewer", "default")).await().indefinitely();

        var found = registry.findById("rm-1").await().indefinitely();

        assertThat(found).isPresent();
        assertThat(found.get().slot()).isEqualTo("reviewer");
    }

    @Test
    void find_by_slot() {
        registry.register(descriptor("rm-2a", "reviewer", "default")).await().indefinitely();
        registry.register(descriptor("rm-2b", "planner", "default")).await().indefinitely();

        var result = registry.find(AgentQuery.bySlot("reviewer", "default")).await().indefinitely();

        assertThat(result.stream().map(AgentDescriptor::agentId).toList())
            .contains("rm-2a").doesNotContain("rm-2b");
    }

    @Test
    void tenancy_isolation() {
        registry.register(descriptor("rm-3", "reviewer", "tenant-a")).await().indefinitely();

        var result = registry.find(AgentQuery.bySlot("reviewer", "tenant-b"))
            .await().indefinitely();

        assertThat(result).isEmpty();
    }
}
```

- [ ] **Step 2: Create InMemoryReactiveAgentRegistry**

```java
// persistence-memory/src/main/java/io/casehub/eidos/memory/InMemoryReactiveAgentRegistry.java
package io.casehub.eidos.memory;

import io.casehub.eidos.api.*;
import io.smallrye.mutiny.Uni;
import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import jakarta.inject.Inject;
import java.util.List;
import java.util.Optional;

@Alternative
@Priority(1)
@ApplicationScoped
public class InMemoryReactiveAgentRegistry implements ReactiveAgentRegistry {

    @Inject InMemoryAgentRegistry delegate;

    @Override
    public Uni<Void> register(AgentDescriptor descriptor) {
        delegate.register(descriptor);
        return Uni.createFrom().voidItem();
    }

    @Override
    public Uni<Optional<AgentDescriptor>> findById(String agentId) {
        return Uni.createFrom().item(delegate.findById(agentId));
    }

    @Override
    public Uni<List<AgentDescriptor>> find(AgentQuery query) {
        return Uni.createFrom().item(delegate.find(query));
    }
}
```

- [ ] **Step 3: Run all persistence-memory tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory
```

Expected: `Tests run: 9, Failures: 0, Errors: 0`

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add persistence-memory/src/main/java/io/casehub/eidos/memory/InMemoryReactiveAgentRegistry.java persistence-memory/src/test/java/io/casehub/eidos/memory/InMemoryReactiveAgentRegistryTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(persistence-memory): InMemoryReactiveAgentRegistry — delegates to blocking store

Uni.createFrom().item() is safe for in-memory ops (no I/O latency).
Both registries are @Alternative @Priority(1) — classpath-activated.

Refs casehubio/eidos#1"
```

---

## Task 14: Vocab Producers — TDD

**Files:**
- Create: `vocab/src/test/java/io/casehub/eidos/vocab/SvoVocabularyTest.java`
- Create: `vocab/src/test/java/io/casehub/eidos/vocab/ConscientiousnessVocabularyTest.java`
- Create: `vocab/src/test/java/io/casehub/eidos/vocab/CasehubSlotVocabularyTest.java`
- Create: `vocab/src/main/java/io/casehub/eidos/vocab/SvoVocabularyProducer.java`
- Create: `vocab/src/main/java/io/casehub/eidos/vocab/ConscientiousnessVocabularyProducer.java`
- Create: `vocab/src/main/java/io/casehub/eidos/vocab/CasehubSlotVocabularyProducer.java`

- [ ] **Step 1: Write SvoVocabularyTest**

```java
// vocab/src/test/java/io/casehub/eidos/vocab/SvoVocabularyTest.java
package io.casehub.eidos.vocab;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

class SvoVocabularyTest {

    final SvoVocabularyProducer producer = new SvoVocabularyProducer();

    @Test
    void uri_is_correct() {
        assertThat(producer.svoVocabulary().uri()).isEqualTo("urn:casehub:vocab:svo");
    }

    @Test
    void has_performer_evaluator_coordinator_terms() {
        var terms = producer.svoVocabulary().terms();
        assertThat(terms).containsKeys("performer", "evaluator", "coordinator");
    }

    @Test
    void performer_has_expected_aliases() {
        var term = producer.svoVocabulary().terms().get("performer");
        assertThat(term.aliases()).contains("actor", "executor");
    }

    @Test
    void evaluator_maps_to_casehub_slot_reviewer() {
        var term = producer.svoVocabulary().terms().get("evaluator");
        assertThat(term.exactMatches().get(CasehubSlotVocabularyProducer.URI)).isEqualTo("reviewer");
    }

    @Test
    void coordinator_maps_to_casehub_slot_planner() {
        var term = producer.svoVocabulary().terms().get("coordinator");
        assertThat(term.exactMatches().get(CasehubSlotVocabularyProducer.URI)).isEqualTo("planner");
    }

    @Test
    void performer_maps_to_casehub_slot_executor() {
        var term = producer.svoVocabulary().terms().get("performer");
        assertThat(term.exactMatches().get(CasehubSlotVocabularyProducer.URI)).isEqualTo("executor");
    }
}
```

- [ ] **Step 2: Write ConscientiousnessVocabularyTest**

```java
// vocab/src/test/java/io/casehub/eidos/vocab/ConscientiousnessVocabularyTest.java
package io.casehub.eidos.vocab;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

class ConscientiousnessVocabularyTest {

    final ConscientiousnessVocabularyProducer producer = new ConscientiousnessVocabularyProducer();

    @Test
    void uri_is_correct() {
        assertThat(producer.conscientiousnessVocabulary().uri())
            .isEqualTo("urn:casehub:vocab:conscientiousness");
    }

    @Test
    void covers_all_four_disposition_axes() {
        var terms = producer.conscientiousnessVocabulary().terms();
        // ruleFollowing
        assertThat(terms).containsKeys("strict", "principled", "flexible");
        // riskAppetite
        assertThat(terms).containsKeys("conservative", "measured", "bold");
        // socialOrient
        assertThat(terms).containsKeys("collaborative", "independent", "facilitative");
        // autonomy
        assertThat(terms).containsKeys("directed", "semi-autonomous", "autonomous");
    }

    @Test
    void strict_has_rule_bound_alias() {
        assertThat(producer.conscientiousnessVocabulary().terms().get("strict").aliases())
            .contains("rule-bound");
    }
}
```

- [ ] **Step 3: Write CasehubSlotVocabularyTest**

```java
// vocab/src/test/java/io/casehub/eidos/vocab/CasehubSlotVocabularyTest.java
package io.casehub.eidos.vocab;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

class CasehubSlotVocabularyTest {

    final CasehubSlotVocabularyProducer producer = new CasehubSlotVocabularyProducer();

    @Test
    void uri_is_correct() {
        assertThat(producer.casehubSlotVocabulary().uri())
            .isEqualTo("urn:casehub:vocab:casehub-slot");
    }

    @Test
    void has_four_slots() {
        assertThat(producer.casehubSlotVocabulary().terms())
            .containsKeys("planner", "reviewer", "executor", "supervisor");
    }

    @Test
    void reviewer_maps_to_svo_evaluator() {
        var term = producer.casehubSlotVocabulary().terms().get("reviewer");
        assertThat(term.exactMatches().get(SvoVocabularyProducer.URI)).isEqualTo("evaluator");
    }

    @Test
    void planner_maps_to_svo_coordinator() {
        var term = producer.casehubSlotVocabulary().terms().get("planner");
        assertThat(term.exactMatches().get(SvoVocabularyProducer.URI)).isEqualTo("coordinator");
    }

    @Test
    void executor_maps_to_svo_performer() {
        var term = producer.casehubSlotVocabulary().terms().get("executor");
        assertThat(term.exactMatches().get(SvoVocabularyProducer.URI)).isEqualTo("performer");
    }

    @Test
    void supervisor_has_no_svo_exact_match() {
        var term = producer.casehubSlotVocabulary().terms().get("supervisor");
        assertThat(term.exactMatches()).doesNotContainKey(SvoVocabularyProducer.URI);
    }

    @Test
    void cross_reference_consistency_svo_to_slot_and_back() {
        // If SVO.evaluator → casehub-slot.reviewer, then casehub-slot.reviewer → SVO.evaluator
        var svo = new SvoVocabularyProducer().svoVocabulary();
        var slot = producer.casehubSlotVocabulary();

        String svoEvaluatorMapsToSlot = svo.terms().get("evaluator").exactMatches().get(CasehubSlotVocabularyProducer.URI);
        String slotReviewerMapsToSvo  = slot.terms().get(svoEvaluatorMapsToSlot).exactMatches().get(SvoVocabularyProducer.URI);

        assertThat(slotReviewerMapsToSvo).isEqualTo("evaluator");
    }
}
```

- [ ] **Step 4: Run tests — expect compile failure (producers don't exist yet)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -pl vocab 2>&1 | grep -E "ERROR|cannot find"
```

Expected: compile errors on missing producer classes.

- [ ] **Step 5: Create SvoVocabularyProducer**

```java
// vocab/src/main/java/io/casehub/eidos/vocab/SvoVocabularyProducer.java
package io.casehub.eidos.vocab;

import io.casehub.eidos.api.Vocabulary;
import io.casehub.eidos.api.VocabularyTerm;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;
import java.util.List;
import java.util.Map;

@ApplicationScoped
public class SvoVocabularyProducer {

    public static final String URI = "urn:casehub:vocab:svo";

    @Produces
    @ApplicationScoped
    public Vocabulary svoVocabulary() {
        return new Vocabulary(URI, "SVO Roles", "1.0", Map.of(
            "performer", new VocabularyTerm("performer", "Performer",
                "Executes the assigned work",
                List.of("actor", "executor"),
                Map.of(CasehubSlotVocabularyProducer.URI, "executor")),
            "evaluator", new VocabularyTerm("evaluator", "Evaluator",
                "Assesses quality of work",
                List.of("reviewer", "judge"),
                Map.of(CasehubSlotVocabularyProducer.URI, "reviewer")),
            "coordinator", new VocabularyTerm("coordinator", "Coordinator",
                "Orchestrates other agents",
                List.of("planner", "orchestrator"),
                Map.of(CasehubSlotVocabularyProducer.URI, "planner"))
        ));
    }
}
```

- [ ] **Step 6: Create ConscientiousnessVocabularyProducer**

```java
// vocab/src/main/java/io/casehub/eidos/vocab/ConscientiousnessVocabularyProducer.java
package io.casehub.eidos.vocab;

import io.casehub.eidos.api.Vocabulary;
import io.casehub.eidos.api.VocabularyTerm;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;
import java.util.List;
import java.util.Map;

@ApplicationScoped
public class ConscientiousnessVocabularyProducer {

    public static final String URI = "urn:casehub:vocab:conscientiousness";

    @Produces
    @ApplicationScoped
    public Vocabulary conscientiousnessVocabulary() {
        return new Vocabulary(URI, "Conscientiousness Disposition Axes", "1.0", Map.of(
            "strict",          new VocabularyTerm("strict", "Strict Rule Following",
                "Follows rules rigidly", List.of("rule-bound", "compliant"), Map.of()),
            "principled",      new VocabularyTerm("principled", "Principled",
                "Follows intent of rules", List.of("values-based"), Map.of()),
            "flexible",        new VocabularyTerm("flexible", "Flexible",
                "Adapts rules to context", List.of("adaptive", "pragmatic"), Map.of()),
            "conservative",    new VocabularyTerm("conservative", "Conservative Risk",
                "Avoids uncertainty", List.of("risk-averse", "cautious"), Map.of()),
            "measured",        new VocabularyTerm("measured", "Measured Risk",
                "Balances risk and reward", List.of("balanced"), Map.of()),
            "bold",            new VocabularyTerm("bold", "Bold Risk",
                "Accepts high uncertainty for reward", List.of("risk-tolerant", "adventurous"), Map.of()),
            "collaborative",   new VocabularyTerm("collaborative", "Collaborative",
                "Works with others by default", List.of("team-oriented", "cooperative"), Map.of()),
            "independent",     new VocabularyTerm("independent", "Independent",
                "Works alone by preference", List.of("autonomous-social", "self-directed"), Map.of()),
            "facilitative",    new VocabularyTerm("facilitative", "Facilitative",
                "Enables others to work", List.of("supportive", "enabling"), Map.of()),
            "directed",        new VocabularyTerm("directed", "Directed Autonomy",
                "Follows explicit instructions", List.of("instruction-following"), Map.of()),
            "semi-autonomous", new VocabularyTerm("semi-autonomous", "Semi-Autonomous",
                "Acts within defined boundaries", List.of("bounded-autonomy"), Map.of()),
            "autonomous",      new VocabularyTerm("autonomous", "Autonomous",
                "Acts on own judgment", List.of("self-governing", "agentic"), Map.of())
        ));
    }
}
```

- [ ] **Step 7: Create CasehubSlotVocabularyProducer**

```java
// vocab/src/main/java/io/casehub/eidos/vocab/CasehubSlotVocabularyProducer.java
package io.casehub.eidos.vocab;

import io.casehub.eidos.api.Vocabulary;
import io.casehub.eidos.api.VocabularyTerm;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;
import java.util.List;
import java.util.Map;

@ApplicationScoped
public class CasehubSlotVocabularyProducer {

    public static final String URI = "urn:casehub:vocab:casehub-slot";

    @Produces
    @ApplicationScoped
    public Vocabulary casehubSlotVocabulary() {
        return new Vocabulary(URI, "CaseHub Slot Roles", "1.0", Map.of(
            "planner",    new VocabularyTerm("planner", "Planner",
                "Plans and coordinates case execution",
                List.of("orchestrator"),
                Map.of(SvoVocabularyProducer.URI, "coordinator")),
            "reviewer",   new VocabularyTerm("reviewer", "Reviewer",
                "Evaluates outputs for quality",
                List.of("evaluator", "judge"),
                Map.of(SvoVocabularyProducer.URI, "evaluator")),
            "executor",   new VocabularyTerm("executor", "Executor",
                "Executes assigned tasks",
                List.of("performer"),
                Map.of(SvoVocabularyProducer.URI, "performer")),
            "supervisor", new VocabularyTerm("supervisor", "Supervisor",
                "Oversees and governs agent behaviour",
                List.of("overseer"),
                Map.of())
        ));
    }
}
```

- [ ] **Step 8: Run vocab tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl vocab
```

Expected: `Tests run: 13, Failures: 0, Errors: 0`

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add vocab/src/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(vocab): SVO, Conscientiousness, CasehubSlot vocabulary producers

Cross-references: SVO ↔ CasehubSlot bidirectional exactMatches.
Conscientiousness covers all four AgentDisposition axes.
Tests verify term presence, aliases, and cross-vocab consistency.

Refs casehubio/eidos#1"
```

---

## Task 15: BlockingReactiveParityTest (ArchUnit)

**Files:**
- Create: `runtime/src/test/java/io/casehub/eidos/runtime/registry/BlockingReactiveParityTest.java`

- [ ] **Step 1: Create BlockingReactiveParityTest**

```java
// runtime/src/test/java/io/casehub/eidos/runtime/registry/BlockingReactiveParityTest.java
package io.casehub.eidos.runtime.registry;

import com.tngtech.archunit.core.domain.JavaClasses;
import com.tngtech.archunit.core.importer.ClassFileImporter;
import com.tngtech.archunit.lang.ArchRule;
import io.casehub.eidos.api.ReactiveAgentRegistry;
import io.smallrye.mutiny.Uni;
import org.junit.jupiter.api.Test;

import static com.tngtech.archunit.lang.syntax.ArchRuleDefinition.methods;
import static org.assertj.core.api.Assertions.assertThat;

class BlockingReactiveParityTest {

    private static final JavaClasses API_CLASSES = new ClassFileImporter()
        .importPackages("io.casehub.eidos.api");

    @Test
    void reactive_registry_methods_return_uni() {
        ArchRule rule = methods()
            .that().areDeclaredIn(ReactiveAgentRegistry.class)
            .should().haveRawReturnType(assignableTo(Uni.class));

        rule.check(API_CLASSES);
    }

    @Test
    void reactive_registry_has_at_least_one_method() {
        long count = API_CLASSES.get(ReactiveAgentRegistry.class)
            .getMethods().size();
        assertThat(count).isGreaterThanOrEqualTo(1);
    }

    private static com.tngtech.archunit.base.DescribedPredicate<com.tngtech.archunit.core.domain.JavaClass> assignableTo(Class<?> type) {
        return com.tngtech.archunit.base.DescribedPredicate.describe(
            "assignable to " + type.getSimpleName(),
            c -> c.isAssignableTo(type)
        );
    }
}
```

- [ ] **Step 2: Run ArchUnit test**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=BlockingReactiveParityTest
```

Expected: `Tests run: 2, Failures: 0, Errors: 0`

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add runtime/src/test/java/io/casehub/eidos/runtime/registry/BlockingReactiveParityTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "test(runtime): BlockingReactiveParityTest — verify ReactiveAgentRegistry methods return Uni<T>

Refs casehubio/eidos#1"
```

---

## Task 16: Full Build Verification

- [ ] **Step 1: Build all modules**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install
```

Expected: `BUILD SUCCESS` — all modules compile, all tests green.

If any test fails, fix before proceeding.

- [ ] **Step 2: Verify Jandex indexes are generated**

```bash
find /Users/mdproctor/claude/casehub/eidos/persistence-memory/target /Users/mdproctor/claude/casehub/eidos/vocab/target -name "jandex.idx"
```

Expected: two `jandex.idx` files found (one per module).

- [ ] **Step 3: Update PLATFORM.md capability ownership table**

In `/Users/mdproctor/claude/casehub/parent/docs/PLATFORM.md`, add to the Capability Ownership table:

```markdown
| Agent descriptor (structured 4-layer identity) | `casehub-eidos` | `AgentDescriptor` record — identity, slot, capabilities, disposition; tenancyId always required |
| Agent registry (store + discover by slot/capability) | `casehub-eidos` | `AgentRegistry` (blocking) + `ReactiveAgentRegistry` (reactive, build-gated); `InMemoryAgentRegistry` for ephemeral installs |
| Vocabulary registry (term resolution + cross-vocab equivalence) | `casehub-eidos` | `VocabularyRegistry` SPI + `CdiVocabularyRegistry` @DefaultBean; discovers `@Produces Vocabulary` CDI beans |
| Well-known vocabularies (SVO, Conscientiousness, CasehubSlot) | `casehub-eidos-vocab` | Optional module — add as dependency to activate; Jandex-indexed for CDI bean discovery |
| Agent capability health (declared vs operable) | `casehub-eidos` | `CapabilityHealth` SPI; `NoOpCapabilityHealth` @DefaultBean (Phase 2: real probing) |
```

- [ ] **Step 4: Commit PLATFORM.md**

```bash
git -C /Users/mdproctor/claude/casehub/parent add docs/PLATFORM.md
git -C /Users/mdproctor/claude/casehub/parent commit -m "docs: add casehub-eidos capability ownership entries

AgentRegistry, VocabularyRegistry, CapabilityHealth, well-known vocabularies.

Refs casehubio/eidos#1"
git -C /Users/mdproctor/claude/casehub/parent push
```

- [ ] **Step 5: Final commit (tag Phase 1 complete)**

```bash
git -C /Users/mdproctor/claude/casehub/eidos tag phase1-complete
git -C /Users/mdproctor/claude/casehub/eidos push --tags
```

---

## Self-Review Notes

- All 8 spec sections are covered by tasks 1–16.
- `AgentQuery.tenancyId` null check tested in Task 2.
- Tenancy isolation tested in both JPA (Task 8) and in-memory (Task 12) registries.
- Cross-vocabulary bidirectional consistency tested in `CasehubSlotVocabularyTest.cross_reference_consistency_svo_to_slot_and_back()` (Task 14).
- Reactive test (Task 9) requires Docker for PostgreSQL DevServices.
- `BlockingReactiveParityTest` uses ArchUnit to assert `Uni<T>` return types — full method-count parity enforcement is a Phase 2 improvement (tracked in casehubio/eidos#1).
