# AgentOutcome.observedAt + ReactiveAgentGraphQuery Parity — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `observedAt` to `AgentOutcome` so ledger backfill can supply accurate timestamps, and add missing `historyByCapability` + `attestationsFor` methods to `ReactiveAgentGraphQuery` and its bridge.

**Architecture:** Two independent API changes in the eidos graph module. Task 1 is a breaking record field addition (5-arg constructor replaces 4-arg; required before nullable). Task 2 adds two methods to an interface and its `@DefaultBean` bridge, with a new package-private constructor on the bridge for testability. Each task ends in a commit that closes one GitHub issue.

**Tech Stack:** Java 21 records, Quarkus 3.32.2, JPA/Hibernate, SmallRye Mutiny, AssertJ, Mockito (version-managed by Quarkus BOM). Test runner: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`.

---

## File Map

| File | Change |
|------|--------|
| `api/src/main/java/io/casehub/eidos/api/AgentOutcome.java` | Replace 4-field record; add compact constructor |
| `graph/src/main/java/io/casehub/eidos/graph/entity/AgentOutcomeEntity.java` | Fix `from()` to use `o.observedAt()`; fix `toRecord()` to pass `observedAt` |
| `graph/src/test/java/io/casehub/eidos/graph/JpaAgentGraphStoreTest.java` | Update 1 call site; add round-trip assertion |
| `graph/src/test/java/io/casehub/eidos/graph/JpaAgentGraphQueryTest.java` | Update 5 call sites (1 in helper + 4 direct) |
| `api/src/main/java/io/casehub/eidos/api/ReactiveAgentGraphQuery.java` | Add 2 methods |
| `runtime/src/main/java/io/casehub/eidos/runtime/graph/BlockingToReactiveGraphBridge.java` | Add no-arg + test constructors; add 2 method implementations |
| `runtime/pom.xml` | Add `mockito-core` + `mockito-junit-jupiter` test deps |
| `runtime/src/test/java/io/casehub/eidos/runtime/graph/BlockingToReactiveGraphBridgeTest.java` | New: 2 delegation tests |

---

## Task 1: Add `observedAt` to `AgentOutcome` (Closes #36)

### Files
- **Modify:** `api/src/main/java/io/casehub/eidos/api/AgentOutcome.java`
- **Modify:** `graph/src/main/java/io/casehub/eidos/graph/entity/AgentOutcomeEntity.java`
- **Modify:** `graph/src/test/java/io/casehub/eidos/graph/JpaAgentGraphStoreTest.java`
- **Modify:** `graph/src/test/java/io/casehub/eidos/graph/JpaAgentGraphQueryTest.java`

---

- [ ] **Step 1.1: Add the round-trip assertion to JpaAgentGraphStoreTest**

  This test will not compile until Task 1 is complete — that is the red state.

  Replace the existing `recordOutcome_links_to_task` test in
  `graph/src/test/java/io/casehub/eidos/graph/JpaAgentGraphStoreTest.java`
  with this (add the `ChronoUnit` import alongside `Instant`):

  ```java
  import java.time.temporal.ChronoUnit;
  ```

  Replace the test body:

  ```java
  @Test
  void recordOutcome_links_to_task() {
      Instant before = Instant.now().truncatedTo(ChronoUnit.MICROS);
      store.recordTask(task("t2", "agent-a", "code-review", "java"));
      store.recordOutcome(new AgentTaskId("t2", "agent-a", "t1"),
                          new AgentOutcome("t2", TaskResult.SUCCEEDED, 0.9, before, null));

      var history = query.agentHistory("agent-a", "t1");

      assertThat(history.outcomes()).hasSize(1);
      assertThat(history.outcomes().get(0).result()).isEqualTo(TaskResult.SUCCEEDED);
      assertThat(history.outcomes().get(0).confidence()).isEqualTo(0.9);
      assertThat(history.outcomes().get(0).observedAt()).isEqualTo(before);
  }
  ```

  Context: `@TestTransaction` on the class rolls back after each test, so `.get(0)` sees only this test's data.

- [ ] **Step 1.2: Run build to observe compile failures (red)**

  ```bash
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -pl api,graph 2>&1 | grep "error:" | head -20
  ```

  Expected: compile errors on `AgentOutcome(...)` constructors (wrong arg count), and on `outcome.observedAt()` (method not found). This confirms the test is driving the design.

- [ ] **Step 1.3: Replace `AgentOutcome` record**

  Overwrite `api/src/main/java/io/casehub/eidos/api/AgentOutcome.java` completely:

  ```java
  package io.casehub.eidos.api;

  import java.time.Instant;
  import java.util.Objects;

  public record AgentOutcome(
      String taskId,
      TaskResult result,
      double confidence,           // 0.0–1.0, enforced in compact constructor
      Instant observedAt,          // when the outcome occurred, not when it was persisted
      DegradationReason degradationReason  // nullable — null if no degradation
  ) {
      public AgentOutcome {
          Objects.requireNonNull(taskId, "taskId must not be null");
          Objects.requireNonNull(result, "result must not be null");
          if (confidence < 0.0 || confidence > 1.0)
              throw new AgentValidationException("confidence", "must be between 0.0 and 1.0");
          Objects.requireNonNull(observedAt, "observedAt must not be null");
      }
  }
  ```

  Field order: all required fields before the nullable `degradationReason`.

- [ ] **Step 1.4: Fix `AgentOutcomeEntity`**

  In `graph/src/main/java/io/casehub/eidos/graph/entity/AgentOutcomeEntity.java`, make two changes:

  **In `from()`** — replace `Instant.now()` with the caller-supplied value:
  ```java
  e.observedAt = o.observedAt();   // was: Instant.now()
  ```

  **In `toRecord()`** — add `observedAt` as the 4th constructor arg:
  ```java
  public AgentOutcome toRecord() {
      return new AgentOutcome(
          taskId,
          TaskResult.valueOf(result),
          confidence,
          observedAt,
          degradationReason != null ? DegradationReason.valueOf(degradationReason) : null
      );
  }
  ```

  Full updated `from()` for reference:
  ```java
  public static AgentOutcomeEntity from(final AgentOutcome o, final AgentTaskEntity task) {
      var e = new AgentOutcomeEntity();
      e.task              = task;
      e.taskId            = o.taskId();
      e.result            = o.result().name();
      e.confidence        = o.confidence();
      e.degradationReason = o.degradationReason() != null ? o.degradationReason().name() : null;
      e.observedAt        = o.observedAt();
      return e;
  }
  ```

- [ ] **Step 1.5: Update `JpaAgentGraphStoreTest` call site**

  The remaining old construction in `JpaAgentGraphStoreTest` is the old 4-arg form.
  Find this line (the `recordOutcome_links_to_task` test was already updated in Step 1.1).
  No other `AgentOutcome` constructions remain in this file after Step 1.1 — verify with:

  ```bash
  grep -n "new AgentOutcome" graph/src/test/java/io/casehub/eidos/graph/JpaAgentGraphStoreTest.java
  ```

  If any remain, update them to the 5-arg form with `Instant.now()` as the 4th arg.

- [ ] **Step 1.6: Update `JpaAgentGraphQueryTest` — all 5 call sites**

  All 5 constructions are in two places:

  **`addOutcomes` helper** (1 construction, used by multiple tests) — add `Instant.now()` as 4th arg:
  ```java
  store.recordOutcome(new AgentTaskId(tid, agentId, "t1"),
                      new AgentOutcome(tid, result, confidence, Instant.now(), null));
  ```

  **`historyByCapability_outcomes_scoped_to_that_capability_only`** (4 direct constructions):
  ```java
  store.recordOutcome(new AgentTaskId(tid, "agent-a", "t1"),
                      new AgentOutcome(tid, TaskResult.SUCCEEDED, 0.9, Instant.now(), null));
  // and:
  store.recordOutcome(new AgentTaskId(tid, "agent-a", "t1"),
                      new AgentOutcome(tid, TaskResult.FAILED, 0.1, Instant.now(), null));
  ```

  **`tenancy_isolation_in_ranking`** (1 direct construction in the other-tenant loop):
  ```java
  store.recordOutcome(new AgentTaskId(tid, "agent-a", "other-tenant"),
      new AgentOutcome(tid, TaskResult.SUCCEEDED, 0.9, Instant.now(), null));
  ```

  Verify no old constructions remain:
  ```bash
  grep -n "new AgentOutcome" graph/src/test/java/io/casehub/eidos/graph/JpaAgentGraphQueryTest.java
  ```
  Every line must show 5 arguments.

- [ ] **Step 1.7: Run all tests and confirm green**

  ```bash
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api,graph
  ```

  Expected: BUILD SUCCESS. All tests pass including the new `observedAt` round-trip assertion.

- [ ] **Step 1.8: Commit**

  ```bash
  git -C /Users/mdproctor/claude/casehub/eidos add \
    api/src/main/java/io/casehub/eidos/api/AgentOutcome.java \
    graph/src/main/java/io/casehub/eidos/graph/entity/AgentOutcomeEntity.java \
    graph/src/test/java/io/casehub/eidos/graph/JpaAgentGraphStoreTest.java \
    graph/src/test/java/io/casehub/eidos/graph/JpaAgentGraphQueryTest.java
  git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(graph): add observedAt to AgentOutcome — business time not persistence time

  observedAt is when the agent produced the result (supplied by caller),
  not Instant.now() at persist time. Entity.from() now uses o.observedAt();
  toRecord() includes it. Compact constructor validates non-null.
  confidence range 0.0–1.0 also enforced at construction.

  Closes #36

  Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
  ```

---

## Task 2: ReactiveAgentGraphQuery Parity (Closes #37)

### Files
- **Modify:** `api/src/main/java/io/casehub/eidos/api/ReactiveAgentGraphQuery.java`
- **Modify:** `runtime/src/main/java/io/casehub/eidos/runtime/graph/BlockingToReactiveGraphBridge.java`
- **Modify:** `runtime/pom.xml`
- **Create:** `runtime/src/test/java/io/casehub/eidos/runtime/graph/BlockingToReactiveGraphBridgeTest.java`

---

- [ ] **Step 2.1: Add Mockito test dependencies to `runtime/pom.xml`**

  In `runtime/pom.xml`, add inside the `<dependencies>` block alongside the existing test deps:

  ```xml
  <dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-core</artifactId>
    <scope>test</scope>
  </dependency>
  <dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-junit-jupiter</artifactId>
    <scope>test</scope>
  </dependency>
  ```

  No version needed — both are managed by the Quarkus BOM imported in the parent pom.

- [ ] **Step 2.2: Write the failing bridge test (red)**

  Create `runtime/src/test/java/io/casehub/eidos/runtime/graph/BlockingToReactiveGraphBridgeTest.java`:

  ```java
  package io.casehub.eidos.runtime.graph;

  import io.casehub.eidos.api.*;
  import org.junit.jupiter.api.Test;
  import org.junit.jupiter.api.extension.ExtendWith;
  import org.mockito.Mock;
  import org.mockito.junit.jupiter.MockitoExtension;
  import java.time.Instant;
  import java.util.List;
  import static org.assertj.core.api.Assertions.assertThat;
  import static org.mockito.Mockito.*;

  @ExtendWith(MockitoExtension.class)
  class BlockingToReactiveGraphBridgeTest {

      @Mock AgentGraphQuery mockBlocking;

      @Test
      void historyByCapability_delegates_to_blocking() {
          var expected = new AgentTaskHistory("agent-a", "t1",
              List.of(), List.of(), List.of(),
              GraphDataSufficiency.forCount(0, null, null, List.of()));
          when(mockBlocking.historyByCapability("agent-a", "code-review", "t1"))
              .thenReturn(expected);
          var bridge = new BlockingToReactiveGraphBridge(mockBlocking);

          var result = bridge.historyByCapability("agent-a", "code-review", "t1")
                             .await().indefinitely();

          assertThat(result).isSameAs(expected);
          verify(mockBlocking).historyByCapability("agent-a", "code-review", "t1");
      }

      @Test
      void attestationsFor_delegates_to_blocking() {
          var expected = List.of(
              new AttestationRef(null, "agent-a", "t1", "hash-xyz", "ML", Instant.now()));
          when(mockBlocking.attestationsFor("agent-a", "t1")).thenReturn(expected);
          var bridge = new BlockingToReactiveGraphBridge(mockBlocking);

          var result = bridge.attestationsFor("agent-a", "t1")
                             .await().indefinitely();

          assertThat(result).isSameAs(expected);
          verify(mockBlocking).attestationsFor("agent-a", "t1");
      }
  }
  ```

  This will fail to compile: `BlockingToReactiveGraphBridge` has no constructor accepting `AgentGraphQuery`, and the two methods don't exist yet on `ReactiveAgentGraphQuery`.

- [ ] **Step 2.3: Run build to observe compile failures (red)**

  ```bash
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -pl api,runtime 2>&1 | grep "error:" | head -20
  ```

  Expected: errors on `BlockingToReactiveGraphBridge(mockBlocking)` (no such constructor), and on `bridge.historyByCapability` and `bridge.attestationsFor` (method not found on `ReactiveAgentGraphQuery`).

- [ ] **Step 2.4: Add missing methods to `ReactiveAgentGraphQuery`**

  Overwrite `api/src/main/java/io/casehub/eidos/api/ReactiveAgentGraphQuery.java`:

  ```java
  package io.casehub.eidos.api;

  import io.smallrye.mutiny.Uni;
  import java.util.List;

  public interface ReactiveAgentGraphQuery {
      Uni<AgentTaskHistory> agentHistory(String agentId, String tenancyId);
      Uni<List<String>> topAgentsByOutcome(String capabilityTag, String taskDomain,
                                            String tenancyId, int limit);
      Uni<AgentTaskHistory> historyByCapability(String agentId, String capabilityTag,
                                                 String tenancyId);
      Uni<List<AttestationRef>> attestationsFor(String agentId, String tenancyId);
  }
  ```

  `Uni<List<AttestationRef>>` is deliberate: matches the blocking return type for bridge simplicity. The bridge wraps synchronous JPA; `Multi` would not change the behaviour here.

- [ ] **Step 2.5: Update `BlockingToReactiveGraphBridge`**

  Overwrite `runtime/src/main/java/io/casehub/eidos/runtime/graph/BlockingToReactiveGraphBridge.java`:

  ```java
  package io.casehub.eidos.runtime.graph;

  import io.casehub.eidos.api.*;
  import io.quarkus.arc.DefaultBean;
  import io.smallrye.mutiny.Uni;
  import io.smallrye.mutiny.infrastructure.Infrastructure;
  import jakarta.enterprise.context.ApplicationScoped;
  import jakarta.inject.Inject;
  import java.util.List;

  @DefaultBean
  @ApplicationScoped
  public class BlockingToReactiveGraphBridge implements ReactiveAgentGraphQuery {

      @Inject AgentGraphQuery blocking;

      public BlockingToReactiveGraphBridge() {}           // CDI path — required; adding the test
                                                          // constructor removes the implicit no-arg

      BlockingToReactiveGraphBridge(AgentGraphQuery blocking) { // test path
          this.blocking = blocking;
      }

      @Override
      public Uni<AgentTaskHistory> agentHistory(final String agentId, final String tenancyId) {
          return Uni.createFrom()
                    .item(() -> blocking.agentHistory(agentId, tenancyId))
                    .runSubscriptionOn(Infrastructure.getDefaultWorkerPool());
      }

      @Override
      public Uni<List<String>> topAgentsByOutcome(final String capabilityTag,
                                                   final String taskDomain,
                                                   final String tenancyId, final int limit) {
          return Uni.createFrom()
                    .item(() -> blocking.topAgentsByOutcome(capabilityTag, taskDomain, tenancyId, limit))
                    .runSubscriptionOn(Infrastructure.getDefaultWorkerPool());
      }

      @Override
      public Uni<AgentTaskHistory> historyByCapability(final String agentId,
                                                        final String capabilityTag,
                                                        final String tenancyId) {
          return Uni.createFrom()
                    .item(() -> blocking.historyByCapability(agentId, capabilityTag, tenancyId))
                    .runSubscriptionOn(Infrastructure.getDefaultWorkerPool());
      }

      @Override
      public Uni<List<AttestationRef>> attestationsFor(final String agentId,
                                                        final String tenancyId) {
          return Uni.createFrom()
                    .item(() -> blocking.attestationsFor(agentId, tenancyId))
                    .runSubscriptionOn(Infrastructure.getDefaultWorkerPool());
      }
  }
  ```

  **Mutiny pattern explanation:** `Uni.createFrom().item(Supplier<T>)` is the lazy form — the supplier is not called until subscription. `runSubscriptionOn(Infrastructure.getDefaultWorkerPool())` moves the subscription (including the blocking JPA call inside the supplier) onto a worker thread. `emitOn` would only move downstream processing, not the blocking call itself.

- [ ] **Step 2.6: Run all tests and confirm green**

  ```bash
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api,runtime,graph
  ```

  Expected: BUILD SUCCESS. Both new bridge delegation tests pass. All prior tests still pass.

- [ ] **Step 2.7: Run full build to confirm no module breakage**

  ```bash
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test
  ```

  Expected: BUILD SUCCESS across all modules.

- [ ] **Step 2.8: Commit**

  ```bash
  git -C /Users/mdproctor/claude/casehub/eidos add \
    api/src/main/java/io/casehub/eidos/api/ReactiveAgentGraphQuery.java \
    runtime/src/main/java/io/casehub/eidos/runtime/graph/BlockingToReactiveGraphBridge.java \
    runtime/src/test/java/io/casehub/eidos/runtime/graph/BlockingToReactiveGraphBridgeTest.java \
    runtime/pom.xml
  git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(graph): ReactiveAgentGraphQuery parity — historyByCapability + attestationsFor

  Adds historyByCapability(agentId, capabilityTag, tenancyId) and
  attestationsFor(agentId, tenancyId) to ReactiveAgentGraphQuery SPI and
  BlockingToReactiveGraphBridge. Adds package-private test constructor to
  bridge (explicit no-arg constructor required alongside it for CDI).

  Closes #37

  Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
  ```
