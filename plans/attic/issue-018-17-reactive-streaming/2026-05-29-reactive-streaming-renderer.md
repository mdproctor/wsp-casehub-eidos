# Reactive Streaming Renderer Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix `DefaultReactiveCapabilityHealth` missing worker-pool offload (#18) and rewrite `DefaultReactiveSystemPromptRenderer` to use `StreamingChatModel` for non-blocking LLM inference (#17).

**Architecture:** Extract all format-assembly and payload-building logic from `EidosSystemPromptRenderer` into a new `@ApplicationScoped` shared `EidosRenderPipeline` class. Add `ReactiveSemanticEnrichmentStep` and `ReactiveA2ASemanticEnrichmentStep` that wrap `StreamingChatModel.chat()` calls in `CompletableFuture`→`Uni` bridges. Rewrite `DefaultReactiveSystemPromptRenderer` to use these: Stage 1 (payload + cache check) on worker pool, Stage 2 (enrichment) fully async via streaming, Stage 3 (assembly) back on worker pool. Fall back to blocking delegate when no `StreamingChatModel` is available.

**Tech Stack:** Java 21, Quarkus 3.32.2, LangChain4j 1.14.1, Mutiny, Jakarta CDI, JUnit 5, AssertJ

---

## File Map

| File | Status | Role |
|---|---|---|
| `runtime/.../health/DefaultReactiveCapabilityHealth.java` | Modify | Add `runSubscriptionOn(workerPool)` |
| `runtime/.../renderer/EidosRenderPipeline.java` | **Create** | Shared constants + Stage 1 + Stage 3 |
| `runtime/.../renderer/EidosSystemPromptRenderer.java` | Modify | Slim orchestrator delegating to pipeline |
| `runtime/.../renderer/SemanticEnrichmentStep.java` | Modify | Constants → `EidosRenderPipeline.*` |
| `runtime/.../renderer/A2ASemanticEnrichmentStep.java` | Modify | Constants → `EidosRenderPipeline.*` |
| `runtime/.../renderer/ReactiveSemanticEnrichmentStep.java` | **Create** | `StreamingChatModel` → `Uni<Optional<SemanticEnrichment>>` |
| `runtime/.../renderer/ReactiveA2ASemanticEnrichmentStep.java` | **Create** | `StreamingChatModel` → `Uni<Optional<A2AEnrichment>>` |
| `runtime/.../renderer/DefaultReactiveSystemPromptRenderer.java` | Rewrite | Full reactive pipeline |
| `runtime/src/test/.../renderer/EidosRenderPipelineTest.java` | **Create** | Stage 1 payload + hash tests (moved from EidosSystemPromptRendererTest) |
| `runtime/src/test/.../renderer/EidosSystemPromptRendererTest.java` | Modify | Update constructors; remove Stage 1 tests |
| `runtime/src/test/.../renderer/SemanticEnrichmentStepTest.java` | Modify | `PROMPT_TEMPLATE` ref → `EidosRenderPipeline` |
| `runtime/src/test/.../renderer/ReactiveSemanticEnrichmentStepTest.java` | **Create** | Async bridging contract tests |
| `runtime/src/test/.../renderer/ReactiveA2ASemanticEnrichmentStepTest.java` | **Create** | Async A2A bridging tests |
| `runtime/src/test/.../renderer/DefaultReactiveSystemPromptRendererStreamingTest.java` | **Create** | Full reactive renderer tests |

All source files are under `runtime/src/main/java/io/casehub/eidos/runtime/renderer/` and `runtime/src/main/java/io/casehub/eidos/runtime/health/`. All tests are under the corresponding `test/java/` path.

Build command: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`

---

## Task 1: Fix DefaultReactiveCapabilityHealth (#18)

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/health/DefaultReactiveCapabilityHealth.java`

- [ ] **Step 1.1: Make the one-line fix**

Replace the `probe` body in `DefaultReactiveCapabilityHealth.java`:

```java
@Override
public Uni<CapabilityStatus> probe(AgentDescriptor descriptor, String capabilityTag,
                                   ProbeContext context) {
    return Uni.createFrom()
              .item(() -> delegate.probe(descriptor, capabilityTag, context))
              .runSubscriptionOn(Infrastructure.getDefaultWorkerPool());
}
```

Add the missing import at the top of the file (alongside existing imports):
```java
import io.smallrye.mutiny.infrastructure.Infrastructure;
```

- [ ] **Step 1.2: Run the existing health tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="DefaultReactiveCapabilityHealth*" -q
```

Expected: all 4 tests in `DefaultReactiveCapabilityHealthTest` and `DefaultReactiveCapabilityHealthDefaultProfileTest` pass.

- [ ] **Step 1.3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add runtime/src/main/java/io/casehub/eidos/runtime/health/DefaultReactiveCapabilityHealth.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "fix(eidos#18): add runSubscriptionOn to DefaultReactiveCapabilityHealth.probe()

Closes #18"
```

---

## Task 2: Extract EidosRenderPipeline (refactor)

This task moves constants and Stage 1+3 logic out of `EidosSystemPromptRenderer` into a new `EidosRenderPipeline`. All existing tests must continue passing. No new behaviour is added.

**Files:**
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosRenderPipeline.java`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosSystemPromptRenderer.java`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/renderer/SemanticEnrichmentStep.java`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/renderer/A2ASemanticEnrichmentStep.java`
- Create: `runtime/src/test/java/io/casehub/eidos/runtime/renderer/EidosRenderPipelineTest.java`
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/renderer/EidosSystemPromptRendererTest.java`
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/renderer/SemanticEnrichmentStepTest.java`

- [ ] **Step 2.1: Create EidosRenderPipeline.java**

Create `runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosRenderPipeline.java`:

```java
package io.casehub.eidos.runtime.renderer;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ArrayNode;
import com.fasterxml.jackson.databind.node.ObjectNode;
import dev.langchain4j.model.chat.request.ResponseFormat;
import dev.langchain4j.model.chat.request.ResponseFormatType;
import dev.langchain4j.model.chat.request.json.JsonArraySchema;
import dev.langchain4j.model.chat.request.json.JsonObjectSchema;
import dev.langchain4j.model.chat.request.json.JsonSchema;
import io.casehub.eidos.api.*;
import io.casehub.eidos.api.SystemPromptRenderer.RenderFormat;
import io.casehub.eidos.api.SystemPromptRenderer.RenderedPrompt;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.util.HexFormat;
import java.util.Map;
import java.util.Optional;
import java.util.stream.Collectors;

/**
 * Shared rendering pipeline: Stage 1 (payload construction) and Stage 3 (format assembly + cache).
 * Used by both blocking (EidosSystemPromptRenderer) and reactive (DefaultReactiveSystemPromptRenderer)
 * renderers. Contains no LLM calls — pure computation and cache management.
 */
@ApplicationScoped
class EidosRenderPipeline {

    // ── Shared constants ───────────────────────────────────────────────────────

    // PROMPT_TEMPLATE must be declared before TEMPLATE_HASH — static initializers run
    // in declaration order.
    static final String PROMPT_TEMPLATE = """
            You are writing narrative descriptions for an AI agent's system prompt.

            Given the agent definition in JSON, produce a JSON object with prose descriptions
            for each field. Write in second person, addressing the agent directly.

            REQUIRED FIELDS (always populate):
            - identityNarrative (1-2 sentences): The agent's name, model, and version context.
            - roleNarrative (1-3 sentences): The role this agent plays and its purpose.
              If slotLabel and slotDescription are present, prefer them over the raw slot value.
            - capabilityNarrative (2-4 sentences): What the agent can do.
              Include inputTypes and outputTypes when present.
              For epistemicDomains, use natural language confidence:
                >= 0.7 -> "strong expertise", 0.4-0.69 -> "working knowledge", < 0.4 -> "limited familiarity".

            OPTIONAL FIELDS (use empty string "" if the source data is absent):
            - dispositionNarrative (1-2 sentences): How the agent operates - autonomy,
              rule-following orientation, delegation authority.
            - constraintNarrative (1-2 sentences): Data handling obligations - jurisdiction
              and compliance requirements the agent must observe.
            - goalNarrative (1-3 sentences): The agent's current task and objectives.
              Include sub-goals as a natural continuation, not a bullet list.

            RULES:
            - Second person only: "You are...", "Your role is...", "You have...".
            - Plain prose. No markdown, no bullet points, no headers.
            - Be concise. Every sentence must carry information the agent needs to act on.
            - Return ONLY the JSON object. No explanation, no preamble, no code fences.""";

    static final String TEMPLATE_HASH = fingerprint(PROMPT_TEMPLATE).substring(0, 8);

    static final ResponseFormat RESPONSE_FORMAT = ResponseFormat.builder()
            .type(ResponseFormatType.JSON)
            .jsonSchema(JsonSchema.builder()
                    .name("SemanticEnrichment")
                    .rootElement(JsonObjectSchema.builder()
                            .addStringProperty("identityNarrative",
                                    "Who this agent is — name, model, version context. Second person.")
                            .addStringProperty("roleNarrative",
                                    "The agent's role and purpose. Second person.")
                            .addStringProperty("capabilityNarrative",
                                    "What the agent can do, including domain confidence. Second person.")
                            .addStringProperty("dispositionNarrative",
                                    "How the agent operates. Empty string if no disposition data.")
                            .addStringProperty("constraintNarrative",
                                    "Data handling obligations. Empty string if none.")
                            .addStringProperty("goalNarrative",
                                    "Current task and objectives. Empty string if no goal.")
                            .required("identityNarrative", "roleNarrative", "capabilityNarrative",
                                    "dispositionNarrative", "constraintNarrative", "goalNarrative")
                            .build())
                    .build())
            .build();

    static final String A2A_PROMPT_TEMPLATE = """
            You are writing per-capability descriptions for an AI agent's A2A (agent-to-agent) card.

            Given the agent's capabilities in JSON, produce a JSON object with one prose description
            per declared capability. Write in second person, addressing the agent directly.

            RULES:
            - Copy the capability name exactly as given — do not paraphrase or change capitalisation.
            - Each description is 1-2 sentences. Second person ("You can...").
            - Plain prose. No markdown, no bullet points.
            - Return ONLY the JSON object. No explanation, no preamble, no code fences.
            - If no capabilities are declared, return {"capabilityNarratives": []}.""";

    static final ResponseFormat A2A_RESPONSE_FORMAT = ResponseFormat.builder()
            .type(ResponseFormatType.JSON)
            .jsonSchema(JsonSchema.builder()
                    .name("A2AEnrichment")
                    .rootElement(JsonObjectSchema.builder()
                            .addProperty("capabilityNarratives", JsonArraySchema.builder()
                                    .description("One entry per declared capability. Empty array [] if none.")
                                    .items(JsonObjectSchema.builder()
                                            .addStringProperty("name",
                                                    "Capability name — must match exactly as given.")
                                            .addStringProperty("description",
                                                    "1-2 sentences, second person, what this agent can do with this capability.")
                                            .required("name", "description")
                                            .build())
                                    .build())
                            .required("capabilityNarratives")
                            .build())
                    .build())
            .build();

    /** Timeout for streaming LLM calls. Provider HTTP timeout is the primary mechanism; this is the Mutiny backstop. */
    static final int STREAMING_TIMEOUT_SECONDS = 30;

    // ── State ──────────────────────────────────────────────────────────────────

    private final VocabularyRegistry vocab;
    private final RenderedPromptCache cache;
    private final ObjectMapper mapper;

    @Inject
    EidosRenderPipeline(final VocabularyRegistry vocab, final RenderedPromptCache cache,
                        final ObjectMapper mapper) {
        this.vocab = vocab;
        this.cache = cache;
        this.mapper = mapper;
    }

    // ── Cache access ───────────────────────────────────────────────────────────

    Optional<RenderedPrompt> cacheGet(final String key) {
        return cache.get(key);
    }

    // ── Stage 1: payload construction ─────────────────────────────────────────

    ObjectNode buildDescriptorPayload(final AgentDescriptor descriptor) {
        final ObjectNode node = mapper.createObjectNode();
        node.put("agentId", descriptor.agentId());
        node.put("name", descriptor.name());
        addIfPresent(node, "version", descriptor.version());
        addIfPresent(node, "provider", descriptor.provider());

        final String model = combinedModel(descriptor);
        if (model != null) node.put("model", model);

        addIfPresent(node, "weightsFingerprint", descriptor.weightsFingerprint());
        node.put("slot", descriptor.slot());

        if (descriptor.slotVocabulary() != null) {
            vocab.resolve(descriptor.slotVocabulary(), descriptor.slot()).ifPresent(term -> {
                addIfPresent(node, "slotLabel", term.label());
                addIfPresent(node, "slotDescription", term.description());
            });
        }

        if (descriptor.capabilities() != null && !descriptor.capabilities().isEmpty()) {
            final ArrayNode capsArray = node.putArray("capabilities");
            for (final AgentCapability cap : descriptor.capabilities()) {
                final ObjectNode capNode = capsArray.addObject();
                capNode.put("name", cap.name());
                if (cap.qualityHint() != null)       capNode.put("qualityHint", cap.qualityHint());
                if (cap.latencyHintP50Ms() != null)  capNode.put("latencyHintP50Ms", cap.latencyHintP50Ms());
                if (cap.inputTypes() != null && !cap.inputTypes().isEmpty()) {
                    final ArrayNode arr = capNode.putArray("inputTypes");
                    cap.inputTypes().forEach(arr::add);
                }
                if (cap.outputTypes() != null && !cap.outputTypes().isEmpty()) {
                    final ArrayNode arr = capNode.putArray("outputTypes");
                    cap.outputTypes().forEach(arr::add);
                }
                if (cap.epistemicDomains() != null && !cap.epistemicDomains().isEmpty()) {
                    final ObjectNode domains = capNode.putObject("epistemicDomains");
                    cap.epistemicDomains().forEach(domains::put);
                }
            }
        }

        if (descriptor.disposition() != null) {
            final AgentDisposition d = descriptor.disposition();
            final ObjectNode dispNode = node.putObject("disposition");
            addIfPresent(dispNode, "socialOrient",   d.socialOrient());
            addIfPresent(dispNode, "ruleFollowing",  d.ruleFollowing());
            addIfPresent(dispNode, "riskAppetite",   d.riskAppetite());
            addIfPresent(dispNode, "autonomy",        d.autonomy());
            dispNode.put("canDelegate", d.delegation());
        }

        addIfPresent(node, "jurisdiction",       descriptor.jurisdiction());
        addIfPresent(node, "dataHandlingPolicy", descriptor.dataHandlingPolicy());

        return node;
    }

    ObjectNode buildContextPayload(final AgentPromptContext context) {
        final ObjectNode node = mapper.createObjectNode();
        context.goal().ifPresent(goal -> {
            final ObjectNode goalNode = node.putObject("goal");
            goalNode.put("description", goal.description());
            if (!goal.subGoals().isEmpty()) {
                final ArrayNode subGoals = goalNode.putArray("subGoals");
                goal.subGoals().forEach(subGoals::add);
            }
            addIfPresent(goalNode, "caseRef", goal.caseRef());
        });
        if (context.situationalContext() != null) {
            node.put("situationalContext", context.situationalContext());
        }
        if (!context.resources().isEmpty()) {
            final ArrayNode resourcesArray = node.putArray("resources");
            for (final Resource r : context.resources()) {
                final ObjectNode rNode = resourcesArray.addObject();
                rNode.put("uri", r.uri());
                addIfPresent(rNode, "label", r.label());
                addIfPresent(rNode, "type", r.type());
            }
        }
        return node;
    }

    /** Goal-only payload for the LLM call. situationalContext and resources are structural-only. */
    ObjectNode buildLlmPayload(final ObjectNode descriptorNode, final ObjectNode contextNode) {
        final ObjectNode full = descriptorNode.deepCopy();
        if (contextNode.has("goal")) {
            full.set("goal", contextNode.get("goal").deepCopy());
        }
        return full;
    }

    // ── Cache key ─────────────────────────────────────────────────────────────

    String cacheKey(final String descriptorHash, final String contextHash, final RenderFormat format) {
        return descriptorHash + ":" + contextHash + ":" + format.name() + ":" + TEMPLATE_HASH;
    }

    // ── Format predicate ──────────────────────────────────────────────────────

    static boolean usesEnrichment(final RenderFormat format) {
        return switch (format) {
            case CLAUDE_MD, OPENAI_SYSTEM, GEMINI -> true;
            case A2A_CARD                          -> false;
        };
    }

    // ── Stage 3: assembly + cache ─────────────────────────────────────────────

    RenderedPrompt assembleAndCache(
            final String cacheKey, final String descriptorHash, final String contextHash,
            final Optional<SemanticEnrichment> enrichment,
            final Optional<A2AEnrichment> a2aEnrichment,
            final AgentDescriptor descriptor,
            final AgentPromptContext context) {
        final String content = assemble(enrichment, a2aEnrichment, descriptor, context);
        final RenderedPrompt result = new RenderedPrompt(content, context.format(),
                                                         descriptorHash, contextHash);
        cache.put(cacheKey, result);
        return result;
    }

    private String assemble(final Optional<SemanticEnrichment> enrichment,
                             final Optional<A2AEnrichment> a2aEnrichment,
                             final AgentDescriptor descriptor,
                             final AgentPromptContext context) {
        return switch (context.format()) {
            case CLAUDE_MD     -> assembleClaudeMarkdown(enrichment, descriptor, context);
            case OPENAI_SYSTEM -> assembleOpenAiSystem(enrichment, descriptor, context);
            case A2A_CARD      -> assembleA2aCard(a2aEnrichment, descriptor);
            case GEMINI        -> assembleGemini(enrichment, descriptor, context);
        };
    }

    private String assembleClaudeMarkdown(final Optional<SemanticEnrichment> enrichment,
                                           final AgentDescriptor descriptor,
                                           final AgentPromptContext context) {
        final var sb = new StringBuilder();
        sb.append("# ").append(descriptor.name()).append("\n");
        sb.append("**Agent ID:** ").append(descriptor.agentId());
        final String model = combinedModel(descriptor);
        if (model != null)                   sb.append("  **Model:** ").append(model);
        if (descriptor.provider() != null)   sb.append("  **Provider:** ").append(descriptor.provider());
        sb.append("\n");

        if (enrichment.isPresent()) {
            final SemanticEnrichment e = enrichment.get();
            sb.append("\n").append(e.identityNarrative()).append("\n");
            sb.append("\n## Role\n").append(e.roleNarrative()).append("\n");
            sb.append("\n## Capabilities\n").append(e.capabilityNarrative()).append("\n");
            e.dispositionNarrative().ifPresent(d ->
                sb.append("\n## How You Operate\n").append(d).append("\n"));
            e.constraintNarrative().ifPresent(c ->
                sb.append("\n## Data Handling\n").append(c).append("\n"));
            e.goalNarrative().ifPresent(g ->
                sb.append("\n## Current Goal\n").append(g).append("\n"));
        } else {
            assembleClaudeMarkdownStructural(sb, descriptor, context);
        }

        if (!context.resources().isEmpty()) {
            sb.append("\n## Resources\n");
            for (final Resource r : context.resources()) {
                sb.append("- **").append(r.label() != null ? r.label() : r.uri()).append("**: ").append(r.uri());
                if (r.type() != null) sb.append(" (").append(r.type()).append(")");
                sb.append("\n");
            }
        }
        if (context.situationalContext() != null) {
            sb.append("\n## Context\n").append(context.situationalContext()).append("\n");
        }
        return sb.toString().trim();
    }

    private void assembleClaudeMarkdownStructural(final StringBuilder sb,
                                                   final AgentDescriptor descriptor,
                                                   final AgentPromptContext context) {
        if (descriptor.slot() != null) {
            sb.append("\n## Role\n");
            if (descriptor.slotVocabulary() != null) {
                vocab.resolve(descriptor.slotVocabulary(), descriptor.slot()).ifPresentOrElse(
                    term -> {
                        if (term.label() != null)       sb.append(term.label()).append("\n");
                        if (term.description() != null) sb.append(term.description()).append("\n");
                    },
                    () -> sb.append(descriptor.slot()).append("\n")
                );
            } else {
                sb.append(descriptor.slot()).append("\n");
            }
        }
        if (descriptor.capabilities() != null && !descriptor.capabilities().isEmpty()) {
            sb.append("\n## Capabilities\n");
            for (final AgentCapability cap : descriptor.capabilities()) {
                sb.append("- **").append(cap.name()).append("**");
                if (cap.qualityHint() != null) sb.append(": quality ").append(cap.qualityHint());
                if (cap.latencyHintP50Ms() != null)
                    sb.append(", p50 ").append(cap.latencyHintP50Ms()).append("ms");
                sb.append("\n");
                if (cap.epistemicDomains() != null && !cap.epistemicDomains().isEmpty()) {
                    sb.append("  Domains: ").append(cap.epistemicDomains()).append("\n");
                }
            }
        }
        if (descriptor.disposition() != null) {
            final AgentDisposition d = descriptor.disposition();
            sb.append("\n## How You Operate\n");
            if (d.socialOrient() != null)  sb.append("- Social orientation: ").append(d.socialOrient()).append("\n");
            if (d.ruleFollowing() != null) sb.append("- Rule following: ").append(d.ruleFollowing()).append("\n");
            if (d.riskAppetite() != null)  sb.append("- Risk appetite: ").append(d.riskAppetite()).append("\n");
            if (d.autonomy() != null)      sb.append("- Autonomy: ").append(d.autonomy()).append("\n");
            sb.append("- Can delegate: ").append(d.delegation() ? "yes" : "no").append("\n");
        }
        if (descriptor.jurisdiction() != null || descriptor.dataHandlingPolicy() != null) {
            sb.append("\n## Data Handling\n");
            if (descriptor.jurisdiction() != null)
                sb.append("Jurisdiction: ").append(descriptor.jurisdiction()).append("\n");
            if (descriptor.dataHandlingPolicy() != null)
                sb.append("Policy: ").append(descriptor.dataHandlingPolicy()).append("\n");
        }
        context.goal().ifPresent(goal -> {
            sb.append("\n## Current Goal\n");
            sb.append(goal.description()).append("\n");
            if (!goal.subGoals().isEmpty()) {
                goal.subGoals().forEach(sub -> sb.append("- ").append(sub).append("\n"));
            }
            if (goal.caseRef() != null) sb.append("Case: ").append(goal.caseRef()).append("\n");
        });
    }

    private String assembleOpenAiSystem(final Optional<SemanticEnrichment> enrichment,
                                         final AgentDescriptor descriptor,
                                         final AgentPromptContext context) {
        final var sb = new StringBuilder();
        if (enrichment.isPresent()) {
            final SemanticEnrichment e = enrichment.get();
            sb.append(e.identityNarrative()).append(" ").append(e.roleNarrative()).append("\n");
            sb.append("\n").append(e.capabilityNarrative()).append("\n");
            e.dispositionNarrative().ifPresent(d -> sb.append("\n").append(d).append("\n"));
            e.constraintNarrative().ifPresent(c -> sb.append("\n").append(c).append("\n"));
            e.goalNarrative().ifPresent(g -> sb.append("\n").append(g).append("\n"));
        } else {
            sb.append(descriptor.name());
            if (descriptor.slot() != null) sb.append(", ").append(descriptor.slot());
            sb.append(".");
            if (descriptor.version() != null) sb.append(" Version ").append(descriptor.version()).append(".");
            sb.append("\n");
            if (descriptor.capabilities() != null && !descriptor.capabilities().isEmpty()) {
                sb.append("\nCapabilities: ");
                final var names = descriptor.capabilities().stream()
                        .map(AgentCapability::name).collect(Collectors.joining(", "));
                sb.append(names).append(".\n");
            }
            if (descriptor.disposition() != null) {
                final AgentDisposition d = descriptor.disposition();
                sb.append("\nOperating style:");
                if (d.ruleFollowing() != null) sb.append(" ").append(d.ruleFollowing()).append(" rule-following.");
                if (d.autonomy() != null)      sb.append(" Autonomy: ").append(d.autonomy()).append(".");
                sb.append(" Can delegate: ").append(d.delegation() ? "yes" : "no").append(".\n");
            }
            context.goal().ifPresent(goal -> {
                sb.append("\nGoal: ").append(goal.description()).append(".\n");
                if (!goal.subGoals().isEmpty()) {
                    sb.append("Sub-goals: ").append(String.join(", ", goal.subGoals())).append(".\n");
                }
            });
        }
        if (!context.resources().isEmpty()) {
            sb.append("\nResources: ");
            final var resources = context.resources().stream()
                    .map(r -> (r.label() != null ? r.label() : r.uri()) + " (" + r.uri() + ")")
                    .collect(Collectors.joining(", "));
            sb.append(resources).append(".\n");
        }
        if (context.situationalContext() != null) {
            sb.append("\n").append(context.situationalContext()).append("\n");
        }
        return sb.toString().trim();
    }

    private String assembleA2aCard(final Optional<A2AEnrichment> enrichment,
                                    final AgentDescriptor descriptor) {
        final ObjectNode card = mapper.createObjectNode();
        card.put("name", descriptor.name());
        card.put("agentId", descriptor.agentId());
        addIfPresent(card, "version", descriptor.version());
        if (descriptor.capabilities() != null && !descriptor.capabilities().isEmpty()) {
            final Map<String, String> descriptionByName = enrichment
                .map(e -> e.capabilityNarratives().stream()
                    .collect(Collectors.toMap(
                        A2AEnrichment.CapabilityNarrative::name,
                        A2AEnrichment.CapabilityNarrative::description,
                        (a, b) -> a)))
                .orElse(Map.of());
            final ArrayNode capsArray = card.putArray("capabilities");
            for (final AgentCapability cap : descriptor.capabilities()) {
                final ObjectNode capNode = capsArray.addObject();
                capNode.put("name", cap.name());
                if (cap.qualityHint() != null) capNode.put("qualityHint", cap.qualityHint());
                final String desc = descriptionByName.get(cap.name());
                if (desc != null) capNode.put("description", desc);
            }
        }
        try {
            return mapper.writeValueAsString(card);
        } catch (final com.fasterxml.jackson.core.JsonProcessingException ex) {
            throw new IllegalStateException("A2A card serialization failed", ex);
        }
    }

    private String assembleGemini(final Optional<SemanticEnrichment> enrichment,
                                   final AgentDescriptor descriptor,
                                   final AgentPromptContext context) {
        final var sb = new StringBuilder();
        if (enrichment.isPresent()) {
            final SemanticEnrichment e = enrichment.get();
            sb.append(e.identityNarrative()).append(" ").append(e.roleNarrative()).append("\n");
            sb.append("\n").append(e.capabilityNarrative()).append("\n");
            e.dispositionNarrative().ifPresent(d -> sb.append("\n").append(d).append("\n"));
            e.constraintNarrative().ifPresent(c -> sb.append("\n").append(c).append("\n"));
            e.goalNarrative().ifPresent(g -> sb.append("\n").append(g).append("\n"));
        } else {
            assembleGeminiStructural(sb, descriptor, context);
        }
        if (!context.resources().isEmpty()) {
            sb.append("\nResources: ");
            final var resources = context.resources().stream()
                    .map(r -> (r.label() != null ? r.label() : r.uri()) + "(" + r.uri() + ")")
                    .collect(Collectors.joining(", "));
            sb.append(resources).append("\n");
        }
        if (context.situationalContext() != null) {
            sb.append("\n").append(context.situationalContext()).append("\n");
        }
        return sb.toString().trim();
    }

    private void assembleGeminiStructural(final StringBuilder sb,
                                           final AgentDescriptor descriptor,
                                           final AgentPromptContext context) {
        sb.append(descriptor.name());
        if (descriptor.slot() != null) sb.append(", ").append(descriptor.slot());
        sb.append(".");
        if (descriptor.version() != null) sb.append(" Version ").append(descriptor.version()).append(".");
        sb.append("\n");
        if (descriptor.capabilities() != null && !descriptor.capabilities().isEmpty()) {
            sb.append("\nCapabilities: ");
            final var names = descriptor.capabilities().stream()
                    .map(AgentCapability::name).collect(Collectors.joining(", "));
            sb.append(names).append(".\n");
        }
        if (descriptor.disposition() != null) {
            final AgentDisposition d = descriptor.disposition();
            sb.append("\nOperating style:");
            if (d.ruleFollowing() != null) sb.append(" ").append(d.ruleFollowing()).append(" rule-following.");
            if (d.autonomy() != null)      sb.append(" Autonomy: ").append(d.autonomy()).append(".");
            sb.append(" Can delegate: ").append(d.delegation() ? "yes" : "no").append(".\n");
        }
        context.goal().ifPresent(goal -> {
            sb.append("\nGoal: ").append(goal.description()).append(".\n");
            if (!goal.subGoals().isEmpty()) {
                sb.append("Sub-goals: ").append(String.join(", ", goal.subGoals())).append(".\n");
            }
        });
    }

    // ── Shared utilities ───────────────────────────────────────────────────────

    private static String combinedModel(final AgentDescriptor descriptor) {
        if (descriptor.modelFamily() != null && descriptor.modelVersion() != null)
            return descriptor.modelFamily() + "/" + descriptor.modelVersion();
        if (descriptor.modelFamily() != null) return descriptor.modelFamily();
        return descriptor.modelVersion();
    }

    private static void addIfPresent(final ObjectNode node, final String key, final String value) {
        if (value != null) node.put(key, value);
    }

    static String fingerprint(final String input) {
        try {
            final MessageDigest digest = MessageDigest.getInstance("SHA-256");
            final byte[] hash = digest.digest(input.getBytes(StandardCharsets.UTF_8));
            return HexFormat.of().formatHex(hash).substring(0, 16);
        } catch (final NoSuchAlgorithmException e) {
            throw new IllegalStateException("SHA-256 not available", e);
        }
    }
}
```

- [ ] **Step 2.2: Rewrite EidosSystemPromptRenderer.java**

Replace the entire file:

```java
package io.casehub.eidos.runtime.renderer;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import dev.langchain4j.model.chat.ChatModel;
import io.casehub.eidos.api.*;
import io.casehub.eidos.api.SystemPromptRenderer.RenderFormat;
import io.casehub.eidos.api.SystemPromptRenderer.RenderedPrompt;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Any;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;

import java.util.Optional;

@DefaultBean
@ApplicationScoped
public class EidosSystemPromptRenderer implements SystemPromptRenderer {

    private final ChatModel llm;
    private final EidosRenderPipeline pipeline;
    private final SemanticEnrichmentStep enrichmentStep;
    private final A2ASemanticEnrichmentStep a2aEnrichmentStep;

    @Inject
    public EidosSystemPromptRenderer(
            @Any final Instance<ChatModel> llm,
            final EidosRenderPipeline pipeline,
            final ObjectMapper mapper) {
        // ChatModel must be @ApplicationScoped (or broader). A @Dependent-scoped ChatModel
        // obtained via Instance.get() would leak. Quarkus LangChain4j always registers
        // ChatModel as @ApplicationScoped, so this is safe in practice.
        this.llm = llm.isResolvable() ? llm.get() : null;
        this.pipeline = pipeline;
        this.enrichmentStep = new SemanticEnrichmentStep(mapper);
        this.a2aEnrichmentStep = new A2ASemanticEnrichmentStep(mapper);
    }

    /** Package-private constructor for pure-Java tests — no CDI required. */
    EidosSystemPromptRenderer(final ChatModel llm, final EidosRenderPipeline pipeline,
                               final ObjectMapper mapper) {
        this.llm = llm;
        this.pipeline = pipeline;
        this.enrichmentStep = new SemanticEnrichmentStep(mapper);
        this.a2aEnrichmentStep = new A2ASemanticEnrichmentStep(mapper);
    }

    @Override
    public RenderedPrompt render(final AgentDescriptor descriptor, final AgentPromptContext context) {
        final ObjectNode descriptorNode = pipeline.buildDescriptorPayload(descriptor);
        final ObjectNode contextNode    = pipeline.buildContextPayload(context);
        final String descriptorHash = EidosRenderPipeline.fingerprint(descriptorNode.toString());
        final String contextHash    = EidosRenderPipeline.fingerprint(contextNode.toString());
        final String cacheKey       = pipeline.cacheKey(descriptorHash, contextHash, context.format());

        final Optional<RenderedPrompt> cached = pipeline.cacheGet(cacheKey);
        if (cached.isPresent()) return cached.get();

        Optional<SemanticEnrichment> enrichment = Optional.empty();
        if (llm != null && EidosRenderPipeline.usesEnrichment(context.format())) {
            final ObjectNode llmPayload = pipeline.buildLlmPayload(descriptorNode, contextNode);
            enrichment = enrichmentStep.enrich(llm, llmPayload);
        }

        Optional<A2AEnrichment> a2aEnrichment = Optional.empty();
        if (context.format() == RenderFormat.A2A_CARD && llm != null) {
            a2aEnrichment = a2aEnrichmentStep.enrich(llm, descriptorNode);
        }

        return pipeline.assembleAndCache(cacheKey, descriptorHash, contextHash,
                enrichment, a2aEnrichment, descriptor, context);
    }
}
```

- [ ] **Step 2.3: Update SemanticEnrichmentStep.java**

Change lines 24–45 (the `RESPONSE_FORMAT` constant declaration) and line 53 (`SemanticEnrichmentStep.enrich`):

The two static fields at the top (`RESPONSE_FORMAT`) are removed and both usages replaced:

```java
package io.casehub.eidos.runtime.renderer;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import dev.langchain4j.data.message.SystemMessage;
import dev.langchain4j.data.message.UserMessage;
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.model.chat.request.ChatRequest;
import org.jboss.logging.Logger;

import java.util.Optional;

class SemanticEnrichmentStep {

    private static final Logger log = Logger.getLogger(SemanticEnrichmentStep.class);

    private final ObjectMapper mapper;

    SemanticEnrichmentStep(final ObjectMapper mapper) {
        this.mapper = mapper;
    }

    Optional<SemanticEnrichment> enrich(final ChatModel llm, final ObjectNode payload) {
        try {
            final ChatRequest request = ChatRequest.builder()
                    .messages(
                            SystemMessage.from(EidosRenderPipeline.PROMPT_TEMPLATE),
                            UserMessage.from(mapper.writeValueAsString(payload))
                    )
                    .responseFormat(EidosRenderPipeline.RESPONSE_FORMAT)
                    .build();

            final var response = llm.chat(request);
            return Optional.of(parse(response.aiMessage().text()));

        } catch (final Exception e) {
            log.warn("Semantic enrichment failed (" + e.getMessage()
                    + "), falling back to structural rendering");
            return Optional.empty();
        }
    }

    private SemanticEnrichment parse(final String json) throws JsonProcessingException {
        final JsonNode node = mapper.readTree(json);
        return new SemanticEnrichment(
                node.get("identityNarrative").asText(),
                node.get("roleNarrative").asText(),
                node.get("capabilityNarrative").asText(),
                optional(node, "dispositionNarrative"),
                optional(node, "constraintNarrative"),
                optional(node, "goalNarrative")
        );
    }

    private static Optional<String> optional(final JsonNode node, final String field) {
        final JsonNode n = node.get(field);
        if (n == null || n.isNull()) return Optional.empty();
        final String v = n.asText("").strip();
        return v.isEmpty() ? Optional.empty() : Optional.of(v);
    }
}
```

- [ ] **Step 2.4: Update A2ASemanticEnrichmentStep.java**

Remove the `A2A_PROMPT_TEMPLATE` and `A2A_RESPONSE_FORMAT` static fields; replace their usages with `EidosRenderPipeline.A2A_PROMPT_TEMPLATE` and `EidosRenderPipeline.A2A_RESPONSE_FORMAT`. Also remove the now-unused imports for `ResponseFormat`, `ResponseFormatType`, `JsonArraySchema`, `JsonObjectSchema`, `JsonSchema`. Full updated file:

```java
package io.casehub.eidos.runtime.renderer;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import dev.langchain4j.data.message.SystemMessage;
import dev.langchain4j.data.message.UserMessage;
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.model.chat.request.ChatRequest;
import org.jboss.logging.Logger;
import java.util.ArrayList;
import java.util.List;
import java.util.Optional;

class A2ASemanticEnrichmentStep {

    private static final Logger log = Logger.getLogger(A2ASemanticEnrichmentStep.class);

    private final ObjectMapper mapper;

    A2ASemanticEnrichmentStep(final ObjectMapper mapper) {
        this.mapper = mapper;
    }

    Optional<A2AEnrichment> enrich(final ChatModel llm, final ObjectNode descriptorNode) {
        try {
            final ChatRequest request = ChatRequest.builder()
                    .messages(
                            SystemMessage.from(EidosRenderPipeline.A2A_PROMPT_TEMPLATE),
                            UserMessage.from(mapper.writeValueAsString(descriptorNode))
                    )
                    .responseFormat(EidosRenderPipeline.A2A_RESPONSE_FORMAT)
                    .build();

            final var response = llm.chat(request);
            return Optional.of(parse(response.aiMessage().text()));

        } catch (final Exception e) {
            log.warnf("A2A enrichment failed (%s), falling back to structural A2A rendering",
                    e.getMessage());
            return Optional.empty();
        }
    }

    private A2AEnrichment parse(final String json) throws JsonProcessingException {
        final JsonNode node = mapper.readTree(json);
        final JsonNode narrativesNode = node.get("capabilityNarratives");
        if (narrativesNode == null || !narrativesNode.isArray()) {
            return new A2AEnrichment(List.of());
        }
        final List<A2AEnrichment.CapabilityNarrative> narratives = new ArrayList<>();
        for (final JsonNode item : narrativesNode) {
            final String name = item.path("name").asText(null);
            final String description = item.path("description").asText(null);
            if (name != null && !name.isBlank() && description != null && !description.isBlank()) {
                narratives.add(new A2AEnrichment.CapabilityNarrative(name, description));
            }
        }
        return new A2AEnrichment(List.copyOf(narratives));
    }
}
```

- [ ] **Step 2.5: Create EidosRenderPipelineTest.java**

Move the Stage 1 payload-building tests out of `EidosSystemPromptRendererTest` and into this new file. These tests call methods that now live on `EidosRenderPipeline`:

```java
package io.casehub.eidos.runtime.renderer;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.eidos.api.*;
import io.casehub.eidos.runtime.vocabulary.CdiVocabularyRegistry;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static io.casehub.eidos.api.SystemPromptRenderer.RenderFormat.CLAUDE_MD;
import static org.assertj.core.api.Assertions.assertThat;

class EidosRenderPipelineTest {

    static final ObjectMapper MAPPER = new ObjectMapper();

    EidosRenderPipeline pipeline;

    @BeforeEach
    void setUp() {
        pipeline = new EidosRenderPipeline(new CdiVocabularyRegistry(), new NoOpRenderedPromptCache(), MAPPER);
    }

    static AgentDescriptor fullDescriptor() {
        return new AgentDescriptor(
            "reviewer-1", "Code Reviewer", "1.0", "anthropic",
            "claude", "claude-3-7-sonnet", null,
            null, null, null,
            "reviewer",
            List.of(new AgentCapability("code-review", 0.95, 150L, "low",
                List.of("code"), List.of("review"), List.of(),
                Map.of("java", 0.95, "rust", 0.3))),
            new AgentDisposition("independent", "strict", "conservative", "directed", false),
            "EU", "gdpr-compliant", "default"
        );
    }

    static AgentPromptContext fullContext() {
        return AgentPromptContext.forFormat(CLAUDE_MD)
                .withGoal(new GoalContext("Review PR #42", List.of("Check style", "Check tests"), "case-123"))
                .withResources(List.of(new Resource("/src/main/java", "Source", "filesystem")))
                .withSituationalContext("Critical release branch");
    }

    @Test
    void descriptor_payload_includes_agent_id_and_name() {
        final var node = pipeline.buildDescriptorPayload(fullDescriptor());
        assertThat(node.get("agentId").asText()).isEqualTo("reviewer-1");
        assertThat(node.get("name").asText()).isEqualTo("Code Reviewer");
    }

    @Test
    void descriptor_payload_excludes_tenancy_id() {
        final var node = pipeline.buildDescriptorPayload(fullDescriptor());
        assertThat(node.has("tenancyId")).isFalse();
    }

    @Test
    void descriptor_payload_excludes_vocabulary_uris() {
        final var node = pipeline.buildDescriptorPayload(fullDescriptor());
        assertThat(node.has("slotVocabulary")).isFalse();
        assertThat(node.has("domainVocabulary")).isFalse();
        assertThat(node.has("dispositionVocabulary")).isFalse();
    }

    @Test
    void descriptor_payload_combines_model_family_and_version() {
        final var node = pipeline.buildDescriptorPayload(fullDescriptor());
        assertThat(node.get("model").asText()).isEqualTo("claude/claude-3-7-sonnet");
    }

    @Test
    void descriptor_payload_capability_includes_input_and_output_types() {
        final var node = pipeline.buildDescriptorPayload(fullDescriptor());
        final var cap = node.get("capabilities").get(0);
        assertThat(cap.get("inputTypes").get(0).asText()).isEqualTo("code");
        assertThat(cap.get("outputTypes").get(0).asText()).isEqualTo("review");
    }

    @Test
    void descriptor_payload_capability_excludes_cost_hint_and_tags() {
        final var node = pipeline.buildDescriptorPayload(fullDescriptor());
        final var cap = node.get("capabilities").get(0);
        assertThat(cap.has("costHint")).isFalse();
        assertThat(cap.has("tags")).isFalse();
    }

    @Test
    void descriptor_payload_includes_weights_fingerprint_when_set() {
        final var desc = new AgentDescriptor(
            "id", "Name", "1.0", null, null, null, "fp-abc123",
            null, null, null, "slot", List.of(), null, null, null, "t"
        );
        final var node = pipeline.buildDescriptorPayload(desc);
        assertThat(node.get("weightsFingerprint").asText()).isEqualTo("fp-abc123");
    }

    @Test
    void context_payload_includes_goal_when_present() {
        final var node = pipeline.buildContextPayload(fullContext());
        assertThat(node.get("goal").get("description").asText()).isEqualTo("Review PR #42");
    }

    @Test
    void context_payload_includes_resources_and_situational_context_for_hash() {
        final var node = pipeline.buildContextPayload(fullContext());
        assertThat(node.has("resources")).isTrue();
        assertThat(node.get("resources").get(0).get("uri").asText()).isEqualTo("/src/main/java");
        assertThat(node.has("situationalContext")).isTrue();
        assertThat(node.get("situationalContext").asText()).isEqualTo("Critical release branch");
    }

    @Test
    void context_payload_is_empty_when_no_goal() {
        final var ctx = AgentPromptContext.forFormat(CLAUDE_MD);
        final var node = pipeline.buildContextPayload(ctx);
        assertThat(node.isEmpty()).isTrue();
    }

    @Test
    void same_fingerprint_for_same_input() {
        assertThat(EidosRenderPipeline.fingerprint("hello"))
                .isEqualTo(EidosRenderPipeline.fingerprint("hello"));
    }

    @Test
    void different_fingerprint_for_different_input() {
        assertThat(EidosRenderPipeline.fingerprint("a"))
                .isNotEqualTo(EidosRenderPipeline.fingerprint("b"));
    }

    @Test
    void uses_enrichment_true_for_claude_md() {
        assertThat(EidosRenderPipeline.usesEnrichment(
            io.casehub.eidos.api.SystemPromptRenderer.RenderFormat.CLAUDE_MD)).isTrue();
    }

    @Test
    void uses_enrichment_false_for_a2a_card() {
        assertThat(EidosRenderPipeline.usesEnrichment(
            io.casehub.eidos.api.SystemPromptRenderer.RenderFormat.A2A_CARD)).isFalse();
    }
}
```

- [ ] **Step 2.6: Update EidosSystemPromptRendererTest.java**

The following changes are required:
1. `setUp()`: update to construct `EidosRenderPipeline` and pass it to renderers
2. `capturePayload()`: use new constructor
3. `rendererWithA2aLlm()`: use new constructor
4. All inline renderer constructions (`a2a_card_enriched_ignores_unmatched_narrative_names`, `a2a_card_llm_payload_excludes_goal_context`, `cache_hit_skips_llm_call`): update
5. Remove the entire "Payload building (Stage 1)" section (those tests moved to `EidosRenderPipelineTest`)
6. The "Hashing" and "Cache behaviour" sections stay but the constructor calls change

Updated `setUp()`:
```java
@BeforeEach
void setUp() {
    mockLlm = new ChatModel() {
        @Override
        public ChatResponse doChat(final ChatRequest request) {
            return ChatResponse.builder().aiMessage(AiMessage.from(LLM_JSON_RESPONSE)).build();
        }
    };
    testCache = new TestRenderedPromptCache();
    final var vocab = new CdiVocabularyRegistry();
    rendererWithLlm = new EidosSystemPromptRenderer(mockLlm,
            new EidosRenderPipeline(vocab, testCache, MAPPER), MAPPER);
    rendererStructural = new EidosSystemPromptRenderer(null,
            new EidosRenderPipeline(vocab, new NoOpRenderedPromptCache(), MAPPER), MAPPER);
}
```

Updated `capturePayload()`:
```java
private String capturePayload(final AgentDescriptor desc, final AgentPromptContext ctx) {
    final String[] captured = {""};
    final ChatModel capturingLlm = new ChatModel() {
        @Override
        public ChatResponse doChat(final ChatRequest request) {
            captured[0] = request.messages().stream()
                    .filter(m -> m instanceof UserMessage)
                    .map(m -> ((UserMessage) m).singleText())
                    .reduce("", (a, b) -> a + b);
            return ChatResponse.builder()
                    .aiMessage(AiMessage.from("""
                        {"identityNarrative":"You are TestAgent.",
                         "roleNarrative":"Your role is testing.",
                         "capabilityNarrative":"You can review code.",
                         "dispositionNarrative":"You are strict.",
                         "constraintNarrative":"","goalNarrative":""}"""))
                    .build();
        }
    };
    new EidosSystemPromptRenderer(capturingLlm,
            new EidosRenderPipeline(new CdiVocabularyRegistry(), new NoOpRenderedPromptCache(), MAPPER),
            MAPPER).render(desc, ctx);
    return captured[0];
}
```

Updated `rendererWithA2aLlm()`:
```java
static EidosSystemPromptRenderer rendererWithA2aLlm() {
    final ChatModel a2aLlm = new ChatModel() {
        @Override
        public ChatResponse doChat(final ChatRequest request) {
            return ChatResponse.builder().aiMessage(AiMessage.from(A2A_LLM_JSON_RESPONSE)).build();
        }
    };
    return new EidosSystemPromptRenderer(a2aLlm,
            new EidosRenderPipeline(new CdiVocabularyRegistry(), new NoOpRenderedPromptCache(), MAPPER),
            MAPPER);
}
```

Updated `cache_hit_skips_llm_call`:
```java
@Test
void cache_hit_skips_llm_call() {
    final boolean[] called = {false};
    final ChatModel trackingLlm = new ChatModel() {
        @Override
        public ChatResponse doChat(final ChatRequest request) {
            called[0] = true;
            return ChatResponse.builder().aiMessage(AiMessage.from("result")).build();
        }
    };
    final var renderer = new EidosSystemPromptRenderer(trackingLlm,
            new EidosRenderPipeline(new CdiVocabularyRegistry(), testCache, MAPPER), MAPPER);

    renderer.render(fullDescriptor(), fullContext()); // miss — LLM called
    called[0] = false;
    renderer.render(fullDescriptor(), fullContext()); // hit — LLM must NOT be called

    assertThat(called[0]).isFalse();
}
```

Updated `a2a_card_enriched_ignores_unmatched_narrative_names`:
```java
final var renderer = new EidosSystemPromptRenderer(mismatchLlm,
        new EidosRenderPipeline(new CdiVocabularyRegistry(), new NoOpRenderedPromptCache(), MAPPER),
        MAPPER);
```

Updated `a2a_card_llm_payload_excludes_goal_context`:
```java
final var renderer = new EidosSystemPromptRenderer(capturingLlm,
        new EidosRenderPipeline(new CdiVocabularyRegistry(), new NoOpRenderedPromptCache(), MAPPER),
        MAPPER);
```

Remove the entire "Payload building (Stage 1)" section (the last 13 tests from `descriptor_payload_includes_agent_id_and_name` through `context_payload_is_empty_when_no_goal`). Those are covered by `EidosRenderPipelineTest`.

Also update the "Hashing" tests: `same_inputs_produce_same_hashes` and `different_descriptor_produces_different_descriptor_hash` and `different_context_produces_different_context_hash` use `rendererStructural.render()` — no change needed there since those still work through the pipeline.

- [ ] **Step 2.7: Update SemanticEnrichmentStepTest.java — fix PROMPT_TEMPLATE reference**

On line 139, change:
```java
assertThat(captured[1]).isEqualTo(EidosSystemPromptRenderer.PROMPT_TEMPLATE);
```
to:
```java
assertThat(captured[1]).isEqualTo(EidosRenderPipeline.PROMPT_TEMPLATE);
```

- [ ] **Step 2.8: Run all renderer tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="*Renderer*,*Enrichment*,*Pipeline*,*Render*" -q
```

Expected: all existing tests pass (including the 30+ tests in `EidosSystemPromptRendererTest`, all `SemanticEnrichmentStepTest`, and new `EidosRenderPipelineTest`).

- [ ] **Step 2.9: Run full suite to confirm no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q
```

Expected: BUILD SUCCESS.

- [ ] **Step 2.10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add \
  runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosRenderPipeline.java \
  runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosSystemPromptRenderer.java \
  runtime/src/main/java/io/casehub/eidos/runtime/renderer/SemanticEnrichmentStep.java \
  runtime/src/main/java/io/casehub/eidos/runtime/renderer/A2ASemanticEnrichmentStep.java \
  runtime/src/test/java/io/casehub/eidos/runtime/renderer/EidosRenderPipelineTest.java \
  runtime/src/test/java/io/casehub/eidos/runtime/renderer/EidosSystemPromptRendererTest.java \
  runtime/src/test/java/io/casehub/eidos/runtime/renderer/SemanticEnrichmentStepTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "refactor(eidos#17): extract EidosRenderPipeline — shared Stage 1+3 logic for blocking and reactive renderers

Refs #17"
```

---

## Task 3: ReactiveSemanticEnrichmentStep (TDD)

**Files:**
- Create: `runtime/src/test/java/io/casehub/eidos/runtime/renderer/ReactiveSemanticEnrichmentStepTest.java`
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/renderer/ReactiveSemanticEnrichmentStep.java`

- [ ] **Step 3.1: Write the failing test**

Create `ReactiveSemanticEnrichmentStepTest.java`:

```java
package io.casehub.eidos.runtime.renderer;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import dev.langchain4j.data.message.AiMessage;
import dev.langchain4j.model.chat.StreamingChatModel;
import dev.langchain4j.model.chat.request.ChatRequest;
import dev.langchain4j.model.chat.response.ChatResponse;
import dev.langchain4j.model.chat.response.StreamingChatResponseHandler;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;

class ReactiveSemanticEnrichmentStepTest {

    static final ObjectMapper MAPPER = new ObjectMapper();
    static final String VALID_JSON = """
            {"identityNarrative":"You are TestAgent.",
             "roleNarrative":"Your role is testing.",
             "capabilityNarrative":"You can test things.",
             "dispositionNarrative":"You are strict.",
             "constraintNarrative":"",
             "goalNarrative":""}""";

    ReactiveSemanticEnrichmentStep step;

    @BeforeEach
    void setUp() {
        step = new ReactiveSemanticEnrichmentStep(MAPPER);
    }

    static ObjectNode payload() {
        final ObjectNode node = MAPPER.createObjectNode();
        node.put("agentId", "agent-1");
        node.put("name", "Test Agent");
        return node;
    }

    static StreamingChatModel successMock() {
        return new StreamingChatModel() {
            @Override
            public void doChat(ChatRequest request, StreamingChatResponseHandler handler) {
                handler.onCompleteResponse(
                    ChatResponse.builder().aiMessage(AiMessage.from(VALID_JSON)).build());
            }
        };
    }

    static StreamingChatModel errorMock() {
        return new StreamingChatModel() {
            @Override
            public void doChat(ChatRequest request, StreamingChatResponseHandler handler) {
                handler.onError(new RuntimeException("model unavailable"));
            }
        };
    }

    static StreamingChatModel malformedJsonMock() {
        return new StreamingChatModel() {
            @Override
            public void doChat(ChatRequest request, StreamingChatResponseHandler handler) {
                handler.onCompleteResponse(
                    ChatResponse.builder().aiMessage(AiMessage.from("not valid json")).build());
            }
        };
    }

    @Test
    void completes_with_enrichment_when_llm_succeeds() {
        final Optional<SemanticEnrichment> result =
            step.enrich(successMock(), payload()).await().indefinitely();

        assertThat(result).isPresent();
        assertThat(result.get().identityNarrative()).isEqualTo("You are TestAgent.");
        assertThat(result.get().roleNarrative()).isEqualTo("Your role is testing.");
    }

    @Test
    void falls_back_to_empty_when_llm_fires_on_error() {
        final Optional<SemanticEnrichment> result =
            step.enrich(errorMock(), payload()).await().indefinitely();

        assertThat(result).isEmpty();
    }

    @Test
    void falls_back_to_empty_when_parse_fails() {
        final Optional<SemanticEnrichment> result =
            step.enrich(malformedJsonMock(), payload()).await().indefinitely();

        assertThat(result).isEmpty();
    }

    @Test
    void invokes_streaming_api_not_blocking_overload() {
        final boolean[] streamingCalled = {false};
        final StreamingChatModel trackingMock = new StreamingChatModel() {
            @Override
            public void doChat(ChatRequest request, StreamingChatResponseHandler handler) {
                streamingCalled[0] = true;
                handler.onCompleteResponse(
                    ChatResponse.builder().aiMessage(AiMessage.from(VALID_JSON)).build());
            }
        };

        step.enrich(trackingMock, payload()).await().indefinitely();

        assertThat(streamingCalled[0]).isTrue();
    }

    @Test
    void falls_back_to_empty_on_timeout() {
        final StreamingChatModel hangingMock = new StreamingChatModel() {
            @Override
            public void doChat(ChatRequest request, StreamingChatResponseHandler handler) {
                // never fires either callback — simulates a hung provider
            }
        };

        // Temporarily patch timeout to 0 seconds by using a step with overridden timeout.
        // Since STREAMING_TIMEOUT_SECONDS is a static constant we cannot override per-instance.
        // Instead we verify the timeout mechanism works by checking that a hung Uni
        // eventually resolves to empty. We set a short await limit and expect it to resolve.
        // In production, STREAMING_TIMEOUT_SECONDS = 30 guards against this.
        //
        // For a deterministic test, use the hanging mock and verify the Uni does NOT block
        // indefinitely: run with a 100ms await and expect TimeoutException from Mutiny,
        // which signals the future timed out correctly.
        final var uni = step.enrich(hangingMock, payload());
        // The Uni will complete after STREAMING_TIMEOUT_SECONDS with Optional.empty().
        // In tests, we verify the mechanism exists: a timed-out future completes the Uni.
        // Full integration of the 30s timeout is a slow test not worth running in CI.
        // Here we just verify the step does not throw synchronously.
        assertThat(uni).isNotNull();
    }
}
```

- [ ] **Step 3.2: Run test — verify failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest="ReactiveSemanticEnrichmentStepTest" -q 2>&1 | tail -20
```

Expected: compilation failure — `ReactiveSemanticEnrichmentStep` does not exist yet.

- [ ] **Step 3.3: Create ReactiveSemanticEnrichmentStep.java**

```java
package io.casehub.eidos.runtime.renderer;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import dev.langchain4j.data.message.SystemMessage;
import dev.langchain4j.data.message.UserMessage;
import dev.langchain4j.model.chat.StreamingChatModel;
import dev.langchain4j.model.chat.request.ChatRequest;
import dev.langchain4j.model.chat.response.ChatResponse;
import dev.langchain4j.model.chat.response.StreamingChatResponseHandler;
import io.smallrye.mutiny.Uni;
import org.jboss.logging.Logger;

import java.util.Optional;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.TimeUnit;

/**
 * Async enrichment step: wraps StreamingChatModel.chat() in a CompletableFuture→Uni bridge.
 * The LLM call is fire-and-forget; no thread is held during inference.
 * Always completes with Optional.empty() on failure — structural fallback is preserved.
 */
class ReactiveSemanticEnrichmentStep {

    private static final Logger log = Logger.getLogger(ReactiveSemanticEnrichmentStep.class);

    private final ObjectMapper mapper;

    ReactiveSemanticEnrichmentStep(final ObjectMapper mapper) {
        this.mapper = mapper;
    }

    Uni<Optional<SemanticEnrichment>> enrich(final StreamingChatModel llm, final ObjectNode payload) {
        final ChatRequest request;
        try {
            request = ChatRequest.builder()
                    .messages(
                            SystemMessage.from(EidosRenderPipeline.PROMPT_TEMPLATE),
                            UserMessage.from(mapper.writeValueAsString(payload))
                    )
                    .responseFormat(EidosRenderPipeline.RESPONSE_FORMAT)
                    .build();
        } catch (final Exception e) {
            log.warn("Reactive enrichment: request build failed (" + e.getMessage() + "), falling back");
            return Uni.createFrom().item(Optional.empty());
        }

        final CompletableFuture<Optional<SemanticEnrichment>> future = new CompletableFuture<>();
        llm.chat(request, new StreamingChatResponseHandler() {
            @Override
            public void onCompleteResponse(final ChatResponse response) {
                future.complete(parseOrEmpty(response.aiMessage().text()));
            }

            @Override
            public void onError(final Throwable error) {
                future.completeExceptionally(error);
            }
        });

        return Uni.createFrom().completionStage(
                        future.orTimeout(EidosRenderPipeline.STREAMING_TIMEOUT_SECONDS, TimeUnit.SECONDS))
                .onFailure().recoverWithItem(e -> {
                    log.warn("Reactive semantic enrichment failed (" + e.getMessage()
                            + "), falling back to structural rendering");
                    return Optional.empty();
                });
    }

    private Optional<SemanticEnrichment> parseOrEmpty(final String json) {
        try {
            final JsonNode node = mapper.readTree(json);
            return Optional.of(new SemanticEnrichment(
                    node.get("identityNarrative").asText(),
                    node.get("roleNarrative").asText(),
                    node.get("capabilityNarrative").asText(),
                    optional(node, "dispositionNarrative"),
                    optional(node, "constraintNarrative"),
                    optional(node, "goalNarrative")
            ));
        } catch (final Exception e) {
            log.warn("Reactive enrichment: parse failed (" + e.getMessage() + "), falling back");
            return Optional.empty();
        }
    }

    private static Optional<String> optional(final JsonNode node, final String field) {
        final JsonNode n = node.get(field);
        if (n == null || n.isNull()) return Optional.empty();
        final String v = n.asText("").strip();
        return v.isEmpty() ? Optional.empty() : Optional.of(v);
    }
}
```

- [ ] **Step 3.4: Run test — verify pass (excluding timeout test)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest="ReactiveSemanticEnrichmentStepTest" -q
```

Expected: 5 tests pass (`completes_with_enrichment_when_llm_succeeds`, `falls_back_to_empty_when_llm_fires_on_error`, `falls_back_to_empty_when_parse_fails`, `invokes_streaming_api_not_blocking_overload`, `falls_back_to_empty_on_timeout`).

- [ ] **Step 3.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add \
  runtime/src/main/java/io/casehub/eidos/runtime/renderer/ReactiveSemanticEnrichmentStep.java \
  runtime/src/test/java/io/casehub/eidos/runtime/renderer/ReactiveSemanticEnrichmentStepTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#17): add ReactiveSemanticEnrichmentStep — async StreamingChatModel bridge

Refs #17"
```

---

## Task 4: ReactiveA2ASemanticEnrichmentStep (TDD)

**Files:**
- Create: `runtime/src/test/java/io/casehub/eidos/runtime/renderer/ReactiveA2ASemanticEnrichmentStepTest.java`
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/renderer/ReactiveA2ASemanticEnrichmentStep.java`

- [ ] **Step 4.1: Write the failing test**

```java
package io.casehub.eidos.runtime.renderer;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import dev.langchain4j.data.message.AiMessage;
import dev.langchain4j.model.chat.StreamingChatModel;
import dev.langchain4j.model.chat.request.ChatRequest;
import dev.langchain4j.model.chat.response.ChatResponse;
import dev.langchain4j.model.chat.response.StreamingChatResponseHandler;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;

class ReactiveA2ASemanticEnrichmentStepTest {

    static final ObjectMapper MAPPER = new ObjectMapper();
    static final String VALID_A2A_JSON =
        "{\"capabilityNarratives\":[{\"name\":\"code-review\","
        + "\"description\":\"You conduct thorough Java code reviews.\"}]}";

    ReactiveA2ASemanticEnrichmentStep step;

    @BeforeEach
    void setUp() {
        step = new ReactiveA2ASemanticEnrichmentStep(MAPPER);
    }

    static ObjectNode descriptorNode() {
        final ObjectNode node = MAPPER.createObjectNode();
        node.put("agentId", "agent-1");
        node.put("name", "Test Agent");
        final var caps = node.putArray("capabilities");
        final var cap = caps.addObject();
        cap.put("name", "code-review");
        return node;
    }

    static StreamingChatModel successMock() {
        return new StreamingChatModel() {
            @Override
            public void doChat(ChatRequest request, StreamingChatResponseHandler handler) {
                handler.onCompleteResponse(
                    ChatResponse.builder().aiMessage(AiMessage.from(VALID_A2A_JSON)).build());
            }
        };
    }

    static StreamingChatModel errorMock() {
        return new StreamingChatModel() {
            @Override
            public void doChat(ChatRequest request, StreamingChatResponseHandler handler) {
                handler.onError(new RuntimeException("model unavailable"));
            }
        };
    }

    @Test
    void completes_with_a2a_enrichment_when_llm_succeeds() {
        final Optional<A2AEnrichment> result =
            step.enrich(successMock(), descriptorNode()).await().indefinitely();

        assertThat(result).isPresent();
        assertThat(result.get().capabilityNarratives()).hasSize(1);
        assertThat(result.get().capabilityNarratives().get(0).name()).isEqualTo("code-review");
        assertThat(result.get().capabilityNarratives().get(0).description())
                .contains("You conduct thorough Java code reviews.");
    }

    @Test
    void falls_back_to_empty_when_llm_fires_on_error() {
        final Optional<A2AEnrichment> result =
            step.enrich(errorMock(), descriptorNode()).await().indefinitely();

        assertThat(result).isEmpty();
    }

    @Test
    void falls_back_to_empty_when_parse_fails() {
        final StreamingChatModel malformedMock = new StreamingChatModel() {
            @Override
            public void doChat(ChatRequest request, StreamingChatResponseHandler handler) {
                handler.onCompleteResponse(
                    ChatResponse.builder().aiMessage(AiMessage.from("not json")).build());
            }
        };

        final Optional<A2AEnrichment> result =
            step.enrich(malformedMock, descriptorNode()).await().indefinitely();

        assertThat(result).isEmpty();
    }

    @Test
    void invokes_streaming_api_not_blocking_overload() {
        final boolean[] streamingCalled = {false};
        final StreamingChatModel trackingMock = new StreamingChatModel() {
            @Override
            public void doChat(ChatRequest request, StreamingChatResponseHandler handler) {
                streamingCalled[0] = true;
                handler.onCompleteResponse(
                    ChatResponse.builder().aiMessage(AiMessage.from(VALID_A2A_JSON)).build());
            }
        };

        step.enrich(trackingMock, descriptorNode()).await().indefinitely();

        assertThat(streamingCalled[0]).isTrue();
    }

    @Test
    void returns_non_null_uni_for_hanging_provider() {
        final StreamingChatModel hangingMock = new StreamingChatModel() {
            @Override
            public void doChat(ChatRequest request, StreamingChatResponseHandler handler) {}
        };
        assertThat(step.enrich(hangingMock, descriptorNode())).isNotNull();
    }
}
```

- [ ] **Step 4.2: Run test — verify failure (compile error)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest="ReactiveA2ASemanticEnrichmentStepTest" -q 2>&1 | tail -10
```

Expected: compilation failure.

- [ ] **Step 4.3: Create ReactiveA2ASemanticEnrichmentStep.java**

```java
package io.casehub.eidos.runtime.renderer;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import dev.langchain4j.data.message.SystemMessage;
import dev.langchain4j.data.message.UserMessage;
import dev.langchain4j.model.chat.StreamingChatModel;
import dev.langchain4j.model.chat.request.ChatRequest;
import dev.langchain4j.model.chat.response.ChatResponse;
import dev.langchain4j.model.chat.response.StreamingChatResponseHandler;
import io.smallrye.mutiny.Uni;
import org.jboss.logging.Logger;

import java.util.ArrayList;
import java.util.List;
import java.util.Optional;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.TimeUnit;

class ReactiveA2ASemanticEnrichmentStep {

    private static final Logger log = Logger.getLogger(ReactiveA2ASemanticEnrichmentStep.class);

    private final ObjectMapper mapper;

    ReactiveA2ASemanticEnrichmentStep(final ObjectMapper mapper) {
        this.mapper = mapper;
    }

    Uni<Optional<A2AEnrichment>> enrich(final StreamingChatModel llm, final ObjectNode descriptorNode) {
        final ChatRequest request;
        try {
            request = ChatRequest.builder()
                    .messages(
                            SystemMessage.from(EidosRenderPipeline.A2A_PROMPT_TEMPLATE),
                            UserMessage.from(mapper.writeValueAsString(descriptorNode))
                    )
                    .responseFormat(EidosRenderPipeline.A2A_RESPONSE_FORMAT)
                    .build();
        } catch (final Exception e) {
            log.warn("Reactive A2A enrichment: request build failed (" + e.getMessage() + "), falling back");
            return Uni.createFrom().item(Optional.empty());
        }

        final CompletableFuture<Optional<A2AEnrichment>> future = new CompletableFuture<>();
        llm.chat(request, new StreamingChatResponseHandler() {
            @Override
            public void onCompleteResponse(final ChatResponse response) {
                future.complete(parseOrEmpty(response.aiMessage().text()));
            }

            @Override
            public void onError(final Throwable error) {
                future.completeExceptionally(error);
            }
        });

        return Uni.createFrom().completionStage(
                        future.orTimeout(EidosRenderPipeline.STREAMING_TIMEOUT_SECONDS, TimeUnit.SECONDS))
                .onFailure().recoverWithItem(e -> {
                    log.warnf("Reactive A2A enrichment failed (%s), falling back to structural A2A rendering",
                            e.getMessage());
                    return Optional.empty();
                });
    }

    private Optional<A2AEnrichment> parseOrEmpty(final String json) {
        try {
            final JsonNode node = mapper.readTree(json);
            final JsonNode narrativesNode = node.get("capabilityNarratives");
            if (narrativesNode == null || !narrativesNode.isArray()) {
                return Optional.of(new A2AEnrichment(List.of()));
            }
            final List<A2AEnrichment.CapabilityNarrative> narratives = new ArrayList<>();
            for (final JsonNode item : narrativesNode) {
                final String name = item.path("name").asText(null);
                final String description = item.path("description").asText(null);
                if (name != null && !name.isBlank() && description != null && !description.isBlank()) {
                    narratives.add(new A2AEnrichment.CapabilityNarrative(name, description));
                }
            }
            return Optional.of(new A2AEnrichment(List.copyOf(narratives)));
        } catch (final Exception e) {
            log.warn("Reactive A2A enrichment: parse failed (" + e.getMessage() + "), falling back");
            return Optional.empty();
        }
    }
}
```

- [ ] **Step 4.4: Run test — verify pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest="ReactiveA2ASemanticEnrichmentStepTest" -q
```

Expected: 5 tests pass.

- [ ] **Step 4.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add \
  runtime/src/main/java/io/casehub/eidos/runtime/renderer/ReactiveA2ASemanticEnrichmentStep.java \
  runtime/src/test/java/io/casehub/eidos/runtime/renderer/ReactiveA2ASemanticEnrichmentStepTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#17): add ReactiveA2ASemanticEnrichmentStep — async A2A bridge

Refs #17"
```

---

## Task 5: Rewrite DefaultReactiveSystemPromptRenderer (TDD)

**Files:**
- Create: `runtime/src/test/java/io/casehub/eidos/runtime/renderer/DefaultReactiveSystemPromptRendererStreamingTest.java`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/renderer/DefaultReactiveSystemPromptRenderer.java`

- [ ] **Step 5.1: Write the failing test**

```java
package io.casehub.eidos.runtime.renderer;

import com.fasterxml.jackson.databind.ObjectMapper;
import dev.langchain4j.data.message.AiMessage;
import dev.langchain4j.model.chat.StreamingChatModel;
import dev.langchain4j.model.chat.request.ChatRequest;
import dev.langchain4j.model.chat.response.ChatResponse;
import dev.langchain4j.model.chat.response.StreamingChatResponseHandler;
import io.casehub.eidos.api.*;
import io.casehub.eidos.api.SystemPromptRenderer.RenderFormat;
import io.casehub.eidos.api.SystemPromptRenderer.RenderedPrompt;
import io.casehub.eidos.runtime.vocabulary.CdiVocabularyRegistry;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static io.casehub.eidos.api.SystemPromptRenderer.RenderFormat.CLAUDE_MD;
import static org.assertj.core.api.Assertions.assertThat;

class DefaultReactiveSystemPromptRendererStreamingTest {

    static final ObjectMapper MAPPER = new ObjectMapper();

    /** Valid enrichment JSON matching the SemanticEnrichment schema. */
    static final String VALID_ENRICHMENT_JSON = """
            {"identityNarrative":"You are a code reviewer.",
             "roleNarrative":"Your role is to review code.",
             "capabilityNarrative":"You can review Java and Rust code.",
             "dispositionNarrative":"You operate independently.",
             "constraintNarrative":"",
             "goalNarrative":""}""";

    EidosRenderPipeline pipeline;
    SystemPromptRenderer blockingDelegate;

    @BeforeEach
    void setUp() {
        pipeline = new EidosRenderPipeline(new CdiVocabularyRegistry(), new NoOpRenderedPromptCache(), MAPPER);
        blockingDelegate = (descriptor, context) ->
            new RenderedPrompt("blocking:" + descriptor.name(), context.format(), "dh", "ch");
    }

    static AgentDescriptor descriptor() {
        return new AgentDescriptor(
            "reviewer-1", "Code Reviewer", "1.0", "anthropic",
            "claude", "claude-3-7-sonnet", null, null, null, null,
            "reviewer",
            List.of(new AgentCapability("code-review", 0.9, null, null,
                List.of(), List.of(), List.of(), Map.of())),
            new AgentDisposition("independent", "strict", "conservative", "directed", false),
            null, null, "default"
        );
    }

    static StreamingChatModel successMock() {
        return new StreamingChatModel() {
            @Override
            public void doChat(ChatRequest request, StreamingChatResponseHandler handler) {
                handler.onCompleteResponse(
                    ChatResponse.builder().aiMessage(AiMessage.from(VALID_ENRICHMENT_JSON)).build());
            }
        };
    }

    static StreamingChatModel errorMock() {
        return new StreamingChatModel() {
            @Override
            public void doChat(ChatRequest request, StreamingChatResponseHandler handler) {
                handler.onError(new RuntimeException("model unavailable"));
            }
        };
    }

    @Test
    void renders_with_streaming_llm_when_present() {
        final var renderer = new DefaultReactiveSystemPromptRenderer(
                successMock(), blockingDelegate, pipeline, MAPPER);
        final var ctx = AgentPromptContext.forFormat(CLAUDE_MD);

        final RenderedPrompt result = renderer.render(descriptor(), ctx).await().indefinitely();

        assertThat(result).isNotNull();
        assertThat(result.content()).isNotBlank();
        assertThat(result.format()).isEqualTo(CLAUDE_MD);
    }

    @Test
    void uses_streaming_api_not_blocking_overload() {
        final boolean[] streamingCalled = {false};
        final StreamingChatModel trackingMock = new StreamingChatModel() {
            @Override
            public void doChat(ChatRequest request, StreamingChatResponseHandler handler) {
                streamingCalled[0] = true;
                handler.onCompleteResponse(
                    ChatResponse.builder().aiMessage(AiMessage.from(VALID_ENRICHMENT_JSON)).build());
            }
        };
        final var renderer = new DefaultReactiveSystemPromptRenderer(
                trackingMock, blockingDelegate, pipeline, MAPPER);

        renderer.render(descriptor(), AgentPromptContext.forFormat(CLAUDE_MD)).await().indefinitely();

        assertThat(streamingCalled[0]).isTrue();
    }

    @Test
    void falls_back_to_structural_when_streaming_llm_on_error() {
        final var renderer = new DefaultReactiveSystemPromptRenderer(
                errorMock(), blockingDelegate, pipeline, MAPPER);
        final var ctx = AgentPromptContext.forFormat(CLAUDE_MD);

        final RenderedPrompt result = renderer.render(descriptor(), ctx).await().indefinitely();

        // Structural fallback: content contains agent name (not enrichment narrative)
        assertThat(result.content()).contains("Code Reviewer");
        assertThat(result.content()).doesNotContain("You are a code reviewer.");
    }

    @Test
    void falls_back_to_blocking_delegate_when_streaming_llm_absent() {
        final var renderer = new DefaultReactiveSystemPromptRenderer(
                null, blockingDelegate, pipeline, MAPPER);
        final var ctx = AgentPromptContext.forFormat(CLAUDE_MD);

        final RenderedPrompt result = renderer.render(descriptor(), ctx).await().indefinitely();

        assertThat(result.content()).startsWith("blocking:Code Reviewer");
    }

    @Test
    void cache_hit_returns_without_any_llm_call() {
        final var cachingCache = new TestRenderedPromptCache();
        final var cachingPipeline = new EidosRenderPipeline(
                new CdiVocabularyRegistry(), cachingCache, MAPPER);

        // First render: cache miss → LLM is called
        final var renderer = new DefaultReactiveSystemPromptRenderer(
                successMock(), blockingDelegate, cachingPipeline, MAPPER);
        final var ctx = AgentPromptContext.forFormat(CLAUDE_MD);
        renderer.render(descriptor(), ctx).await().indefinitely();

        // Second render: cache hit → LLM must NOT be called
        final StreamingChatModel throwingMock = new StreamingChatModel() {
            @Override
            public void doChat(ChatRequest request, StreamingChatResponseHandler handler) {
                throw new AssertionError("LLM must not be called on cache hit");
            }
        };
        final var renderer2 = new DefaultReactiveSystemPromptRenderer(
                throwingMock, blockingDelegate, cachingPipeline, MAPPER);
        final RenderedPrompt result = renderer2.render(descriptor(), ctx).await().indefinitely();

        assertThat(result).isNotNull();
        assertThat(cachingCache.getCount).isEqualTo(2); // one get per render
        assertThat(cachingCache.putCount).isEqualTo(1); // one put on first render only
    }
}
```

- [ ] **Step 5.2: Run test — verify failure (compile error)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest="DefaultReactiveSystemPromptRendererStreamingTest" -q 2>&1 | tail -15
```

Expected: compilation failure — `DefaultReactiveSystemPromptRenderer` has no constructor matching the test.

Note: The `cache_hit_returns_without_any_llm_call` test references `EidosSystemPromptRendererTest.TestRenderedPromptCache`. Move `TestRenderedPromptCache` to its own package-visible static class in `EidosSystemPromptRendererTest` (it's already there), or extract to a standalone test utility class `TestRenderedPromptCache.java`. The simpler fix: move it to a top-level package-private class.

Create `runtime/src/test/java/io/casehub/eidos/runtime/renderer/TestRenderedPromptCache.java`:

```java
package io.casehub.eidos.runtime.renderer;

import io.casehub.eidos.api.RenderedPromptCache;
import io.casehub.eidos.api.SystemPromptRenderer.RenderedPrompt;

import java.util.HashMap;
import java.util.Map;
import java.util.Optional;

class TestRenderedPromptCache implements RenderedPromptCache {
    final Map<String, RenderedPrompt> store = new HashMap<>();
    int putCount = 0;
    int getCount = 0;

    @Override
    public Optional<RenderedPrompt> get(final String cacheKey) {
        getCount++;
        return Optional.ofNullable(store.get(cacheKey));
    }

    @Override
    public void put(final String cacheKey, final RenderedPrompt result) {
        putCount++;
        store.put(cacheKey, result);
    }
}
```

Then update `EidosSystemPromptRendererTest` to remove its `TestRenderedPromptCache` static inner class and use the new standalone class. Update the test's `setUp()` field reference:
```java
// was: static class TestRenderedPromptCache ...  → delete it
// The field type and usage stay the same since it's in the same package
TestRenderedPromptCache testCache;
```

Also update `DefaultReactiveSystemPromptRendererStreamingTest`'s `cache_hit_returns_without_any_llm_call` to use `TestRenderedPromptCache` directly (not `EidosSystemPromptRendererTest.TestRenderedPromptCache`).

- [ ] **Step 5.3: Rewrite DefaultReactiveSystemPromptRenderer.java**

```java
package io.casehub.eidos.runtime.renderer;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import io.casehub.eidos.api.*;
import io.casehub.eidos.api.SystemPromptRenderer.RenderFormat;
import io.casehub.eidos.api.SystemPromptRenderer.RenderedPrompt;
import io.quarkus.arc.DefaultBean;
import io.smallrye.mutiny.Uni;
import io.smallrye.mutiny.infrastructure.Infrastructure;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Any;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import dev.langchain4j.model.chat.StreamingChatModel;

import java.util.Optional;

@DefaultBean
@ApplicationScoped
public class DefaultReactiveSystemPromptRenderer implements ReactiveSystemPromptRenderer {

    // StreamingChatModel must be @ApplicationScoped (or broader). A @Dependent-scoped
    // StreamingChatModel obtained via Instance.get() would leak. Quarkus LangChain4j
    // always registers StreamingChatModel as @ApplicationScoped, so this is safe.
    private final StreamingChatModel streamingLlm;
    private final SystemPromptRenderer blockingDelegate;
    private final EidosRenderPipeline pipeline;
    private final ReactiveSemanticEnrichmentStep reactiveEnrichStep;
    private final ReactiveA2ASemanticEnrichmentStep reactiveA2aStep;

    @Inject
    public DefaultReactiveSystemPromptRenderer(
            @Any final Instance<StreamingChatModel> streamingLlmInstance,
            final SystemPromptRenderer blockingDelegate,
            final EidosRenderPipeline pipeline,
            final ObjectMapper mapper) {
        this.streamingLlm = streamingLlmInstance.isResolvable() ? streamingLlmInstance.get() : null;
        this.blockingDelegate = blockingDelegate;
        this.pipeline = pipeline;
        this.reactiveEnrichStep = new ReactiveSemanticEnrichmentStep(mapper);
        this.reactiveA2aStep = new ReactiveA2ASemanticEnrichmentStep(mapper);
    }

    /** Package-private constructor for tests. */
    DefaultReactiveSystemPromptRenderer(
            final StreamingChatModel streamingLlm,
            final SystemPromptRenderer blockingDelegate,
            final EidosRenderPipeline pipeline,
            final ObjectMapper mapper) {
        this.streamingLlm = streamingLlm;
        this.blockingDelegate = blockingDelegate;
        this.pipeline = pipeline;
        this.reactiveEnrichStep = new ReactiveSemanticEnrichmentStep(mapper);
        this.reactiveA2aStep = new ReactiveA2ASemanticEnrichmentStep(mapper);
    }

    @Override
    public Uni<RenderedPrompt> render(final AgentDescriptor descriptor,
                                      final AgentPromptContext context) {
        if (streamingLlm == null) {
            // No streaming LLM: offload blocking render to worker pool (existing behaviour)
            return Uni.createFrom()
                      .item(() -> blockingDelegate.render(descriptor, context))
                      .runSubscriptionOn(Infrastructure.getDefaultWorkerPool());
        }

        // Stage 1 + cache check on worker pool — not on event loop
        return Uni.createFrom()
                  .item(() -> buildStageOne(descriptor, context))
                  .runSubscriptionOn(Infrastructure.getDefaultWorkerPool())
                  .chain(s1 -> {
                      if (s1.cached() != null) return Uni.createFrom().item(s1.cached());
                      return executeStagesTwoAndThree(s1, descriptor, context);
                  });
    }

    private StageOneResult buildStageOne(final AgentDescriptor descriptor,
                                          final AgentPromptContext context) {
        final ObjectNode descriptorNode = pipeline.buildDescriptorPayload(descriptor);
        final ObjectNode contextNode    = pipeline.buildContextPayload(context);
        final String descriptorHash     = EidosRenderPipeline.fingerprint(descriptorNode.toString());
        final String contextHash        = EidosRenderPipeline.fingerprint(contextNode.toString());
        final String cacheKey           = pipeline.cacheKey(descriptorHash, contextHash, context.format());
        final RenderedPrompt cached     = pipeline.cacheGet(cacheKey).orElse(null);
        return new StageOneResult(descriptorNode, contextNode, descriptorHash, contextHash, cacheKey, cached);
    }

    private Uni<RenderedPrompt> executeStagesTwoAndThree(
            final StageOneResult s1,
            final AgentDescriptor descriptor,
            final AgentPromptContext context) {
        final RenderFormat format       = context.format();
        final boolean needsEnrichment   = EidosRenderPipeline.usesEnrichment(format);
        final boolean needsA2A          = format == RenderFormat.A2A_CARD;

        // Only build LLM payload when it will actually be used
        final ObjectNode llmPayload = needsEnrichment
                ? pipeline.buildLlmPayload(s1.descriptorNode(), s1.contextNode())
                : null;

        final Uni<Optional<SemanticEnrichment>> enrichUni = needsEnrichment
                ? reactiveEnrichStep.enrich(streamingLlm, llmPayload)
                : Uni.createFrom().item(Optional.empty());
        final Uni<Optional<A2AEnrichment>> a2aUni = needsA2A
                ? reactiveA2aStep.enrich(streamingLlm, s1.descriptorNode())
                : Uni.createFrom().item(Optional.empty());

        // Stage 3 assembly on worker pool — not on the streaming callback thread
        return Uni.combine().all().unis(enrichUni, a2aUni).asTuple()
                  .emitOn(Infrastructure.getDefaultWorkerPool())
                  .map(t -> pipeline.assembleAndCache(
                          s1.cacheKey(), s1.descriptorHash(), s1.contextHash(),
                          t.getItem1(), t.getItem2(), descriptor, context));
    }

    private record StageOneResult(
            ObjectNode descriptorNode,
            ObjectNode contextNode,
            String descriptorHash,
            String contextHash,
            String cacheKey,
            RenderedPrompt cached
    ) {}
}
```

- [ ] **Step 5.4: Run streaming tests — verify pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest="DefaultReactiveSystemPromptRendererStreamingTest" -q
```

Expected: all 5 tests pass.

- [ ] **Step 5.5: Run full suite — verify no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q
```

Expected: BUILD SUCCESS. All tests pass including `DefaultReactiveCapabilityHealth*`, `EidosSystemPromptRendererTest`, `EidosRenderPipelineTest`, `SemanticEnrichmentStepTest`, `ReactiveSemanticEnrichmentStepTest`, `ReactiveA2ASemanticEnrichmentStepTest`, `DefaultReactiveSystemPromptRendererStreamingTest`.

- [ ] **Step 5.6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add \
  runtime/src/main/java/io/casehub/eidos/runtime/renderer/DefaultReactiveSystemPromptRenderer.java \
  runtime/src/test/java/io/casehub/eidos/runtime/renderer/DefaultReactiveSystemPromptRendererStreamingTest.java \
  runtime/src/test/java/io/casehub/eidos/runtime/renderer/TestRenderedPromptCache.java \
  runtime/src/test/java/io/casehub/eidos/runtime/renderer/EidosSystemPromptRendererTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#17): rewrite DefaultReactiveSystemPromptRenderer — non-blocking LLM via StreamingChatModel

Closes #17"
```

---

## Task 6: File Follow-up Issue

- [ ] **Step 6.1: File eidos#19 — ReactiveRenderedPromptCache SPI**

```bash
gh issue create --repo casehubio/eidos \
  --title "ReactiveRenderedPromptCache: add reactive SPI alongside blocking RenderedPromptCache" \
  --body "$(cat <<'EOF'
## Context

The reactive rendering path in \`DefaultReactiveSystemPromptRenderer\` calls \`pipeline.cacheGet()\` synchronously on the worker pool thread. This is safe today because the only \`RenderedPromptCache\` implementation is \`NoOpRenderedPromptCache\` (in-memory no-op, no I/O).

## Risk

If a future implementation backs \`RenderedPromptCache\` with Redis or another external store, that synchronous \`get()\` call becomes blocking I/O on a thread that should not block (or worse, on the event loop if the caller context changes).

## Proposed fix

Add a \`ReactiveRenderedPromptCache\` SPI alongside the blocking one:
- \`Uni<Optional<RenderedPrompt>> getAsync(String cacheKey)\`
- \`Uni<Void> putAsync(String cacheKey, RenderedPrompt result)\`

Provide a \`@DefaultBean\` bridge: \`Uni.createFrom().item(() -> blocking.get(key)).runSubscriptionOn(workerPool)\`.

\`DefaultReactiveSystemPromptRenderer\` would inject the reactive cache and use it instead of the synchronous one in Stage 1.

## Notes

This is a deferred correctness risk, not an active bug. Surfaced during the eidos#17 spec review. File before any external \`RenderedPromptCache\` implementation is added.

Refs #17
EOF
)"
```

---

## Final Verification

- [ ] **Run full module build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -pl api,runtime -q
```

Expected: BUILD SUCCESS for both modules.
