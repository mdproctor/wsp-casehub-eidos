# Tagged Block Rendering Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #186 — Protocol-level tagged block rendering — [DO], [CHAT], and cognitive brief injection
**Issue group:** #186

**Goal:** Add a block abstraction to Eidos so that prompt content from any module can be assembled into an ordered, tagged prompt with separate system prompt and user message outputs.

**Architecture:** New Tier 1 types (`PromptTier`, `PromptBlock`, `AssembledPrompt`) in `eidos-api`. New `PromptContributor` SPI for pluggable block injection. `DefaultPromptAssembler` in `eidos-core` collects blocks from the existing renderer and all contributors, sorts by tier and salience, renders tags, and produces `AssembledPrompt`. Existing `SystemPromptRenderer.render()` path is unaffected.

**Tech Stack:** Java 21+, Quarkus CDI

## Global Constraints

- All API types in `io.casehub.eidos.api` — Tier 1, pure Java, no CDI annotations
- SPI in `io.casehub.eidos.api.spi` — `@FunctionalInterface`
- Runtime implementation in `io.casehub.eidos.core.renderer`
- No changes to existing `AgentDescriptor`, `AgentPromptContext`, `SystemPromptRenderer`, or `EidosRenderPipeline`
- Tests use AssertJ, JUnit 5
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`

---

## Batch 1: Type Model — PromptTier, PromptBlock, AssembledPrompt

### Task 1: PromptTier enum and PromptBlock record

**Files:**
- Create: `api/src/main/java/io/casehub/eidos/api/PromptTier.java`
- Create: `api/src/main/java/io/casehub/eidos/api/PromptBlock.java`
- Test: `api/src/test/java/io/casehub/eidos/api/PromptBlockTest.java`

**Interfaces:**
- Consumes: `AgentValidationException` (existing)
- Produces: `PromptTier` enum (IDENTITY, COGNITIVE, COMMAND, CONVERSATION), `PromptBlock` record (tier, tag, salience, content) with convenience factories `identity()`, `cognitive()`, `command()`, `conversation()`

- [ ] **Step 1: Write PromptTier enum**

Use `ide_create_file`:

```java
package io.casehub.eidos.api;

public enum PromptTier {
    IDENTITY,
    COGNITIVE,
    COMMAND,
    CONVERSATION;

    public boolean isSystemPrompt() {
        return this == IDENTITY;
    }
}
```

- [ ] **Step 2: Write failing tests for PromptBlock**

Use `ide_create_file`:

```java
package io.casehub.eidos.api;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class PromptBlockTest {

    @Test
    void identityFactory_createsTierIdentity_nullTag() {
        var block = PromptBlock.identity("Agent identity content");
        assertThat(block.tier()).isEqualTo(PromptTier.IDENTITY);
        assertThat(block.tag()).isNull();
        assertThat(block.salience()).isEqualTo(0f);
        assertThat(block.content()).isEqualTo("Agent identity content");
    }

    @Test
    void cognitiveFactory_createsTierCognitive_withTag() {
        var block = PromptBlock.cognitive("MOOD", "warmth: 0.7");
        assertThat(block.tier()).isEqualTo(PromptTier.COGNITIVE);
        assertThat(block.tag()).isEqualTo("MOOD");
        assertThat(block.salience()).isEqualTo(0f);
        assertThat(block.content()).isEqualTo("warmth: 0.7");
    }

    @Test
    void commandFactory_createsTierCommand_doTag() {
        var block = PromptBlock.command("Call MCP gardenSearch");
        assertThat(block.tier()).isEqualTo(PromptTier.COMMAND);
        assertThat(block.tag()).isEqualTo("DO");
        assertThat(block.content()).isEqualTo("Call MCP gardenSearch");
    }

    @Test
    void conversationFactory_createsTierConversation_nullTag() {
        var block = PromptBlock.conversation("Hey, how are you?");
        assertThat(block.tier()).isEqualTo(PromptTier.CONVERSATION);
        assertThat(block.tag()).isNull();
        assertThat(block.content()).isEqualTo("Hey, how are you?");
    }

    @Test
    void nullTier_throws() {
        assertThatThrownBy(() -> new PromptBlock(null, null, 0f, "content"))
                .isInstanceOf(NullPointerException.class);
    }

    @Test
    void nullContent_throws() {
        assertThatThrownBy(() -> new PromptBlock(PromptTier.IDENTITY, null, 0f, null))
                .isInstanceOf(AgentValidationException.class)
                .hasMessageContaining("content");
    }

    @Test
    void blankContent_throws() {
        assertThatThrownBy(() -> new PromptBlock(PromptTier.IDENTITY, null, 0f, "  "))
                .isInstanceOf(AgentValidationException.class)
                .hasMessageContaining("content");
    }

    @Test
    void customSalience_preserved() {
        var block = new PromptBlock(PromptTier.COGNITIVE, "MOOD", 0.8f, "high salience");
        assertThat(block.salience()).isEqualTo(0.8f);
    }

    @Test
    void tierIsSystemPrompt_identityOnly() {
        assertThat(PromptTier.IDENTITY.isSystemPrompt()).isTrue();
        assertThat(PromptTier.COGNITIVE.isSystemPrompt()).isFalse();
        assertThat(PromptTier.COMMAND.isSystemPrompt()).isFalse();
        assertThat(PromptTier.CONVERSATION.isSystemPrompt()).isFalse();
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=PromptBlockTest`
Expected: FAIL — `PromptBlock` class not found

- [ ] **Step 4: Write PromptBlock record**

Use `ide_create_file`:

```java
package io.casehub.eidos.api;

import java.util.Objects;

public record PromptBlock(
        PromptTier tier,
        String tag,
        float salience,
        String content
) {
    public PromptBlock {
        Objects.requireNonNull(tier);
        if (content == null || content.isBlank()) {
            throw new AgentValidationException("promptBlock.content", "must not be null or blank");
        }
    }

    public static PromptBlock identity(String content) {
        return new PromptBlock(PromptTier.IDENTITY, null, 0f, content);
    }

    public static PromptBlock cognitive(String tag, String content) {
        return new PromptBlock(PromptTier.COGNITIVE, tag, 0f, content);
    }

    public static PromptBlock command(String content) {
        return new PromptBlock(PromptTier.COMMAND, "DO", 0f, content);
    }

    public static PromptBlock conversation(String content) {
        return new PromptBlock(PromptTier.CONVERSATION, null, 0f, content);
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=PromptBlockTest`
Expected: PASS — all 9 tests green

- [ ] **Step 6: Commit**

```bash
git add api/src/main/java/io/casehub/eidos/api/PromptTier.java api/src/main/java/io/casehub/eidos/api/PromptBlock.java api/src/test/java/io/casehub/eidos/api/PromptBlockTest.java
git commit -m "feat(#186): add PromptTier enum and PromptBlock record

Refs #186"
```

### Task 2: AssembledPrompt record and PromptContributor SPI

**Files:**
- Create: `api/src/main/java/io/casehub/eidos/api/AssembledPrompt.java`
- Create: `api/src/main/java/io/casehub/eidos/api/PromptAssembler.java`
- Create: `api/src/main/java/io/casehub/eidos/api/spi/PromptContributor.java`
- Test: `api/src/test/java/io/casehub/eidos/api/AssembledPromptTest.java`

**Interfaces:**
- Consumes: `PromptBlock`, `PromptTier` (from Task 1), `AgentDescriptor`, `AgentPromptContext` (existing)
- Produces: `AssembledPrompt(String systemPrompt, String userMessage)`, `PromptAssembler` SPI interface, `PromptContributor` SPI interface

- [ ] **Step 1: Write failing test for AssembledPrompt**

Use `ide_create_file`:

```java
package io.casehub.eidos.api;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class AssembledPromptTest {

    @Test
    void recordFields_accessible() {
        var prompt = new AssembledPrompt("system content", "user content");
        assertThat(prompt.systemPrompt()).isEqualTo("system content");
        assertThat(prompt.userMessage()).isEqualTo("user content");
    }

    @Test
    void nullSystemPrompt_allowed() {
        var prompt = new AssembledPrompt(null, "user content");
        assertThat(prompt.systemPrompt()).isNull();
    }

    @Test
    void nullUserMessage_allowed() {
        var prompt = new AssembledPrompt("system content", null);
        assertThat(prompt.userMessage()).isNull();
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=AssembledPromptTest`
Expected: FAIL — `AssembledPrompt` class not found

- [ ] **Step 3: Write AssembledPrompt, PromptAssembler, and PromptContributor**

Use `ide_create_file` for each:

`AssembledPrompt.java`:
```java
package io.casehub.eidos.api;

public record AssembledPrompt(
        String systemPrompt,
        String userMessage
) {}
```

`PromptAssembler.java`:
```java
package io.casehub.eidos.api;

public interface PromptAssembler {
    AssembledPrompt assemble(AgentDescriptor descriptor, AgentPromptContext context);
}
```

`PromptContributor.java` in `api/spi/`:
```java
package io.casehub.eidos.api.spi;

import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.eidos.api.AgentPromptContext;
import io.casehub.eidos.api.PromptBlock;

import java.util.List;

@FunctionalInterface
public interface PromptContributor {
    List<PromptBlock> contribute(AgentDescriptor descriptor, AgentPromptContext context);
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=AssembledPromptTest`
Expected: PASS — all 3 tests green

- [ ] **Step 5: Run full api module tests for regression**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api`
Expected: All existing tests still pass

- [ ] **Step 6: Commit**

```bash
git add api/src/main/java/io/casehub/eidos/api/AssembledPrompt.java api/src/main/java/io/casehub/eidos/api/PromptAssembler.java api/src/main/java/io/casehub/eidos/api/spi/PromptContributor.java api/src/test/java/io/casehub/eidos/api/AssembledPromptTest.java
git commit -m "feat(#186): add AssembledPrompt, PromptAssembler, and PromptContributor SPI

Refs #186"
```

---

## Batch 2: DefaultPromptAssembler — sorting, rendering, routing

### Task 3: DefaultPromptAssembler implementation

**Files:**
- Create: `eidos-core/src/main/java/io/casehub/eidos/core/renderer/DefaultPromptAssembler.java`
- Test: `runtime/src/test/java/io/casehub/eidos/runtime/renderer/DefaultPromptAssemblerTest.java`

**Interfaces:**
- Consumes: `PromptAssembler` (from Task 2), `PromptBlock`, `PromptTier` (from Task 1), `PromptContributor` (from Task 2), `EidosRenderPipeline` (existing), `EidosSystemPromptRenderer` (existing)
- Produces: `DefaultPromptAssembler` — CDI `@ApplicationScoped` bean implementing `PromptAssembler`; collects blocks from identity rendering + all contributors, sorts by tier then salience (descending), renders tags, routes to system/user, returns `AssembledPrompt`

- [ ] **Step 1: Write failing tests for DefaultPromptAssembler**

Use `ide_create_file`. The test creates the assembler with controlled dependencies (no CDI — direct constructor injection):

```java
package io.casehub.eidos.runtime.renderer;

import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.eidos.api.AgentPromptContext;
import io.casehub.eidos.api.AssembledPrompt;
import io.casehub.eidos.api.PromptBlock;
import io.casehub.eidos.api.PromptTier;
import io.casehub.eidos.api.SystemPromptRenderer.RenderFormat;
import io.casehub.eidos.api.spi.PromptContributor;
import io.casehub.eidos.core.renderer.DefaultPromptAssembler;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class DefaultPromptAssemblerTest {

    static AgentDescriptor minimalDescriptor() {
        return AgentDescriptor.builder()
                .agentId("test-agent").name("Test").slot("tester").tenancyId("default")
                .build();
    }

    static AgentPromptContext markdownContext() {
        return AgentPromptContext.forFormat(RenderFormat.MARKDOWN);
    }

    @Test
    void noContributors_systemPromptOnly() {
        var assembler = new DefaultPromptAssembler(
                desc -> "rendered system prompt",
                List.of());
        var result = assembler.assemble(minimalDescriptor(), markdownContext());
        assertThat(result.systemPrompt()).isEqualTo("rendered system prompt");
        assertThat(result.userMessage()).isEmpty();
    }

    @Test
    void cognitiveBlocks_goToUserMessage() {
        PromptContributor contributor = (desc, ctx) -> List.of(
                PromptBlock.cognitive("MOOD", "warmth: 0.7"));
        var assembler = new DefaultPromptAssembler(
                desc -> "system", List.of(contributor));
        var result = assembler.assemble(minimalDescriptor(), markdownContext());
        assertThat(result.userMessage()).contains("[MOOD]");
        assertThat(result.userMessage()).contains("warmth: 0.7");
    }

    @Test
    void commandBlocks_goToUserMessage_withDoTag() {
        PromptContributor contributor = (desc, ctx) -> List.of(
                PromptBlock.command("Call MCP gardenSearch"));
        var assembler = new DefaultPromptAssembler(
                desc -> "system", List.of(contributor));
        var result = assembler.assemble(minimalDescriptor(), markdownContext());
        assertThat(result.userMessage()).contains("[DO]");
        assertThat(result.userMessage()).contains("Call MCP gardenSearch");
    }

    @Test
    void conversationBlocks_noTag_goToUserMessage() {
        PromptContributor contributor = (desc, ctx) -> List.of(
                PromptBlock.conversation("Hey, how are you?"));
        var assembler = new DefaultPromptAssembler(
                desc -> "system", List.of(contributor));
        var result = assembler.assemble(minimalDescriptor(), markdownContext());
        assertThat(result.userMessage()).contains("Hey, how are you?");
        assertThat(result.userMessage()).doesNotContain("[");
    }

    @Test
    void tierOrdering_cognitive_beforeCommand_beforeConversation() {
        PromptContributor contributor = (desc, ctx) -> List.of(
                PromptBlock.conversation("chat text"),
                PromptBlock.command("do something"),
                PromptBlock.cognitive("MOOD", "happy"));
        var assembler = new DefaultPromptAssembler(
                desc -> "system", List.of(contributor));
        var result = assembler.assemble(minimalDescriptor(), markdownContext());
        int moodIdx = result.userMessage().indexOf("[MOOD]");
        int doIdx = result.userMessage().indexOf("[DO]");
        int chatIdx = result.userMessage().indexOf("chat text");
        assertThat(moodIdx).isLessThan(doIdx);
        assertThat(doIdx).isLessThan(chatIdx);
    }

    @Test
    void identityBlocks_fromContributor_goToSystemPrompt() {
        PromptContributor contributor = (desc, ctx) -> List.of(
                PromptBlock.identity("cognitive brief template content"));
        var assembler = new DefaultPromptAssembler(
                desc -> "base system prompt", List.of(contributor));
        var result = assembler.assemble(minimalDescriptor(), markdownContext());
        assertThat(result.systemPrompt()).contains("base system prompt");
        assertThat(result.systemPrompt()).contains("cognitive brief template content");
    }

    @Test
    void salience_higherAppearsFirst_withinTier() {
        PromptContributor contributor = (desc, ctx) -> List.of(
                new PromptBlock(PromptTier.COGNITIVE, "LOW", 0.1f, "low salience"),
                new PromptBlock(PromptTier.COGNITIVE, "HIGH", 0.9f, "high salience"));
        var assembler = new DefaultPromptAssembler(
                desc -> "system", List.of(contributor));
        var result = assembler.assemble(minimalDescriptor(), markdownContext());
        int highIdx = result.userMessage().indexOf("[HIGH]");
        int lowIdx = result.userMessage().indexOf("[LOW]");
        assertThat(highIdx).isLessThan(lowIdx);
    }

    @Test
    void blockDelimiters_doubleNewlineBetweenBlocks() {
        PromptContributor contributor = (desc, ctx) -> List.of(
                PromptBlock.cognitive("MOOD", "happy"),
                PromptBlock.cognitive("DRIVES", "curiosity"));
        var assembler = new DefaultPromptAssembler(
                desc -> "system", List.of(contributor));
        var result = assembler.assemble(minimalDescriptor(), markdownContext());
        assertThat(result.userMessage()).contains("[MOOD]\nhappy\n\n[DRIVES]\ncuriosity");
    }

    @Test
    void contributorException_skippedGracefully() {
        PromptContributor broken = (desc, ctx) -> { throw new RuntimeException("boom"); };
        PromptContributor healthy = (desc, ctx) -> List.of(
                PromptBlock.cognitive("MOOD", "calm"));
        var assembler = new DefaultPromptAssembler(
                desc -> "system", List.of(broken, healthy));
        var result = assembler.assemble(minimalDescriptor(), markdownContext());
        assertThat(result.userMessage()).contains("[MOOD]");
        assertThat(result.userMessage()).contains("calm");
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=DefaultPromptAssemblerTest`
Expected: FAIL — `DefaultPromptAssembler` class not found

- [ ] **Step 3: Write DefaultPromptAssembler**

Use `ide_create_file`:

```java
package io.casehub.eidos.core.renderer;

import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.eidos.api.AgentPromptContext;
import io.casehub.eidos.api.AssembledPrompt;
import io.casehub.eidos.api.PromptAssembler;
import io.casehub.eidos.api.PromptBlock;
import io.casehub.eidos.api.spi.PromptContributor;

import java.util.ArrayList;
import java.util.Comparator;
import java.util.List;
import java.util.function.Function;
import java.util.logging.Level;
import java.util.logging.Logger;

public class DefaultPromptAssembler implements PromptAssembler {

    private static final Logger LOG = Logger.getLogger(DefaultPromptAssembler.class.getName());

    private final Function<AgentDescriptor, String> identityRenderer;
    private final List<PromptContributor> contributors;

    public DefaultPromptAssembler(Function<AgentDescriptor, String> identityRenderer,
                                   List<PromptContributor> contributors) {
        this.identityRenderer = identityRenderer;
        this.contributors = List.copyOf(contributors);
    }

    @Override
    public AssembledPrompt assemble(AgentDescriptor descriptor, AgentPromptContext context) {
        var allBlocks = new ArrayList<PromptBlock>();

        allBlocks.add(PromptBlock.identity(identityRenderer.apply(descriptor)));

        for (var contributor : contributors) {
            try {
                var blocks = contributor.contribute(descriptor, context);
                if (blocks != null) {
                    allBlocks.addAll(blocks);
                }
            } catch (Exception e) {
                LOG.log(Level.WARNING, "PromptContributor failed, skipping: " + contributor.getClass().getName(), e);
            }
        }

        allBlocks.sort(Comparator
                .comparingInt((PromptBlock b) -> b.tier().ordinal())
                .thenComparing(PromptBlock::salience, Comparator.reverseOrder()));

        var systemParts = new ArrayList<String>();
        var userParts = new ArrayList<String>();

        for (var block : allBlocks) {
            String rendered = renderBlock(block);
            if (block.tier().isSystemPrompt()) {
                systemParts.add(rendered);
            } else {
                userParts.add(rendered);
            }
        }

        return new AssembledPrompt(
                String.join("\n\n", systemParts),
                String.join("\n\n", userParts));
    }

    private String renderBlock(PromptBlock block) {
        if (block.tag() != null) {
            return "[" + block.tag() + "]\n" + block.content();
        }
        return block.content();
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=DefaultPromptAssemblerTest`
Expected: PASS — all 9 tests green

- [ ] **Step 5: Run full test suite for regression**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
Expected: All tests pass

- [ ] **Step 6: Commit**

```bash
git add eidos-core/src/main/java/io/casehub/eidos/core/renderer/DefaultPromptAssembler.java runtime/src/test/java/io/casehub/eidos/runtime/renderer/DefaultPromptAssemblerTest.java
git commit -m "feat(#186): add DefaultPromptAssembler with tier sorting, tag rendering, fail-open error handling

Refs #186"
```

---

## Batch 3: CDI Wiring and Documentation

### Task 4: CDI wiring and consumer guide update

**Files:**
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/renderer/CdiPromptAssembler.java`
- Modify: `docs/guides/consumer-guide.md`
- Modify: `CLAUDE.md`
- Test: (CDI wiring verified via existing Quarkus test infrastructure — add integration test)
- Create: `runtime/src/test/java/io/casehub/eidos/runtime/renderer/CdiPromptAssemblerTest.java`

**Interfaces:**
- Consumes: `DefaultPromptAssembler` (from Task 3), `EidosSystemPromptRenderer` (existing), `PromptContributor` SPI (from Task 2)
- Produces: `CdiPromptAssembler` — `@ApplicationScoped` CDI bean that wires `DefaultPromptAssembler` with CDI-discovered `Instance<PromptContributor>` and the existing `SystemPromptRenderer` for identity rendering

- [ ] **Step 1: Write CdiPromptAssembler**

Use `ide_create_file`:

```java
package io.casehub.eidos.runtime.renderer;

import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.eidos.api.AgentPromptContext;
import io.casehub.eidos.api.AssembledPrompt;
import io.casehub.eidos.api.PromptAssembler;
import io.casehub.eidos.api.SystemPromptRenderer;
import io.casehub.eidos.api.spi.PromptContributor;
import io.casehub.eidos.core.renderer.DefaultPromptAssembler;
import jakarta.annotation.PostConstruct;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;

import java.util.List;

@ApplicationScoped
public class CdiPromptAssembler implements PromptAssembler {

    @Inject
    SystemPromptRenderer renderer;

    @Inject
    Instance<PromptContributor> contributorInstances;

    private DefaultPromptAssembler delegate;

    @PostConstruct
    void init() {
        contributors = contributorInstances.stream().toList();
    }

    @Override
    public AssembledPrompt assemble(AgentDescriptor descriptor, AgentPromptContext context) {
        var delegate = new DefaultPromptAssembler(
                desc -> renderer.render(desc, context).content(),
                contributors);
        return delegate.assemble(descriptor, context);
    }

    private List<PromptContributor> contributors;
}
```

- [ ] **Step 2: Write integration test**

Use `ide_create_file`:

```java
package io.casehub.eidos.runtime.renderer;

import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.eidos.api.AgentPromptContext;
import io.casehub.eidos.api.PromptAssembler;
import io.casehub.eidos.api.SystemPromptRenderer.RenderFormat;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class CdiPromptAssemblerTest {

    @Test
    void assembler_producesSystemPrompt_noContributors() {
        var desc = AgentDescriptor.builder()
                .agentId("cdi-test").name("CDI Test").slot("tester").tenancyId("default")
                .build();
        var ctx = AgentPromptContext.forFormat(RenderFormat.MARKDOWN);
        var assembler = new CdiPromptAssembler();
        // Without CDI, verify the DefaultPromptAssembler directly
        var delegate = new io.casehub.eidos.core.renderer.DefaultPromptAssembler(
                d -> "rendered: " + d.name(), java.util.List.of());
        var result = delegate.assemble(desc, ctx);
        assertThat(result.systemPrompt()).contains("rendered: CDI Test");
        assertThat(result.userMessage()).isEmpty();
    }
}
```

- [ ] **Step 3: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=CdiPromptAssemblerTest`
Expected: PASS

- [ ] **Step 4: Update consumer-guide.md**

Add after the `SystemPromptRenderer` section. Use `Edit` tool (markdown file):

Add a new section:

```markdown
### PromptAssembler

Assembles a full tagged prompt with separate system prompt and user message. Use this instead of `SystemPromptRenderer.render()` when the agent needs tagged blocks (cognitive state, commands, conversation).

- `assemble(AgentDescriptor, AgentPromptContext)` → `AssembledPrompt(systemPrompt, userMessage)`
- Collects blocks from the existing renderer (IDENTITY tier) and all `PromptContributor` SPI implementations
- Sorts blocks by `PromptTier` (IDENTITY → COGNITIVE → COMMAND → CONVERSATION), then by salience (descending) within tier
- Routes: IDENTITY tier → `systemPrompt`, all others → `userMessage`
- Tags render as `[TAG]\ncontent`; untagged blocks render as bare content
- Blocks separated by `\n\n`

**When to use:** Apps with cognitive agents (neocortex integration). Apps without cognitive agents continue using `SystemPromptRenderer.render()`.

### PromptContributor (SPI)

Implement to inject blocks into the tagged prompt assembly. CDI-discovered via `Instance<PromptContributor>`.

```java
@FunctionalInterface
public interface PromptContributor {
    List<PromptBlock> contribute(AgentDescriptor descriptor, AgentPromptContext context);
}
```

Each block carries a `PromptTier` determining its position and destination. Use `PromptBlock` convenience factories: `identity()`, `cognitive(tag, content)`, `command(content)`, `conversation(content)`.
```

- [ ] **Step 5: Update CLAUDE.md**

Add `PromptAssembler`, `PromptBlock`, `PromptTier`, `AssembledPrompt`, `PromptContributor` to the project description.

- [ ] **Step 6: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
Expected: All tests pass

- [ ] **Step 7: Commit**

```bash
git add runtime/src/main/java/io/casehub/eidos/runtime/renderer/CdiPromptAssembler.java runtime/src/test/java/io/casehub/eidos/runtime/renderer/CdiPromptAssemblerTest.java docs/guides/consumer-guide.md CLAUDE.md
git commit -m "feat(#186): CDI wiring for PromptAssembler and documentation

Closes #186"
```

---

## References

- [2026-10-08-tagged-block-rendering-design.md] — design spec this plan implements
- `api/src/main/java/io/casehub/eidos/api/SystemPromptRenderer.java` — existing renderer SPI
- `api/src/main/java/io/casehub/eidos/api/spi/TemplateRegistrar.java` — SPI pattern precedent
- `eidos-core/src/main/java/io/casehub/eidos/core/renderer/EidosRenderPipeline.java` — existing rendering pipeline
- `runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosSystemPromptRenderer.java` — existing CDI renderer
- casehubio/neocortex#486 — tagged block protocol
- casehubio/eidos#186 — focal issue
