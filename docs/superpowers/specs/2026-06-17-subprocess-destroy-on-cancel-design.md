# Subprocess Destroy on Cancellation — Design Spec
**Issue:** eidos#52  
**Branch:** issue-52-subprocess-destroy-on-cancel  
**Repos affected:** casehub-platform (primary), casehub-eidos (none — eval behaviour unchanged)

---

## Problem

When `AgentProviderChatModel.doChat()` times out via `await().atMost(Duration)`, Mutiny cancellation propagates to `ClaudeAgentClient.buildEventStream()`, which calls `sdkClient.close().subscribe()`. `DefaultClaudeAsyncClient.close()` schedules cleanup on Reactor's bounded-elastic pool, where `StreamingTransport.close()` runs `process.destroy()` + `waitFor(5s)` + `destroyForcibly()` (lines 1157–1168 of the SDK source).

When parallel enrichment calls all timeout simultaneously (observed: 16 at once in `evaluateRealWorldScenarios`), 16 bounded-elastic threads are each blocked for up to 5 seconds on `process.waitFor()`. The 17th call (ProximityJudge) starts a new subprocess into a resource-starved environment and hangs indefinitely.

## Root Cause — Architectural Mismatch

`buildEventStream()` uses the SDK's **session mode** (`--input-format stream-json`): a persistent bidirectional subprocess designed for multi-turn conversations. It must be explicitly closed; close is async with a 5-second grace period before `destroyForcibly()`.

`invoke()` is **single-turn**. The correct subprocess mode is **one-shot** (`-p "prompt" --output-format stream-json`): the subprocess exits naturally on completion. On cancellation, `destroyForcibly()` is called on a `Process` reference the caller already owns — immediate, synchronous, no grace period, no scheduler dependency.

## Design

### Two subprocess modes — cleanly separated

| Path | Mode | Process ownership | Cancel |
|---|---|---|---|
| `run()` → `buildEventStream()` | One-shot (`-p "prompt"`) | `ClaudeAgentClient` owns `Process` directly | `process.destroyForcibly()` — immediate |
| `openSession()` → `ClaudeAgentSession` | Session mode (SDK) | SDK owns | Unchanged — out of scope for eidos#52 |

The output format (`--output-format stream-json`) is identical in both modes. SDK parsing classes (`ControlMessageParser`, `ParsedMessage`) are reused as-is for JSON line parsing.

---

### New class: `ClaudeOneShotProcess` (package-private, `agent-claude/`)

Owns a single-invocation subprocess: construction starts the process, `destroyForcibly()` kills it immediately, `close()` closes it gracefully.

#### Construction

Accepts `AgentSessionConfig config`, `ClaudeAgentProperties properties`. Builds and starts the subprocess immediately.

#### Command construction

Flags derived from `AgentSessionConfig` and `ClaudeAgentProperties`:

```
<binary>  -p "<userPrompt>"  --output-format stream-json  --verbose
          [--system-prompt "<systemPrompt>"]
          [--mcp-config /tmp/claude-mcp-<uuid>.json]
```

**`--verbose` is required.** Without it, `--output-format stream-json` emits only the final `ResultMessage` — no incremental `AssistantMessage` text deltas. The stream would produce zero `TextDelta` events before the final completion signal, yielding degenerate (non-streaming) behaviour.

**Session-mode flags excluded** (these are not applicable to one-shot):
- `--input-format stream-json` — the defining session-mode flag; one-shot has no stdin protocol
- `--permission-prompt-tool` — requires the bidirectional control channel IPC
- `--continue`, `--resume`, `--fork-session` — session resume/fork semantics
- `--include-partial-messages`, `--agents` — session streaming/multi-agent optimisations

**Applicable flags not yet surfaced by config (deferred — implement when config adds them):**
`--model`, `--allowedTools`, `--disallowedTools`, `--permission-mode`, `--max-turns`, `--max-budget-usd`, `--max-thinking-tokens`, `--append-system-prompt`, `--add-dir`, `--settings`, `--setting-sources`

`ClaudeAgentProperties` currently exposes only `binaryPath()`, `defaultTimeout()`, and `maxConcurrentSessions()`. `AgentSessionConfig` exposes `systemPrompt`, `userPrompt`, `mcpServers`, `timeout`, and `correlationId`. Only these fields are implemented for this issue.

#### MCP config file

When `config.mcpServers()` is non-empty, write a temp file with the MCP server JSON (same format and logic as `StreamingTransport.buildMcpConfigForCli()`) and pass `--mcp-config /tmp/file`. The temp file path is stored in a field. Deleted in `destroyForcibly()` and `close()` — both paths delete it idempotently.

#### Environment

Replicate `StreamingTransport.buildProcessEnvironment()` exactly:
1. **Whitelist inherited vars** (Unix): `HOME`, `LOGNAME`, `PATH`, `SHELL`, `TERM`, `USER`, `LANG`, `LC_ALL`, `LC_CTYPE`. Exclude shell functions (vars whose value starts with `()`).
2. **Always add** `ANTHROPIC_API_KEY` if present in `System.getenv()`.
3. **SDK identity markers**: `CLAUDE_CODE_ENTRYPOINT=sdk-java`, `CLAUDE_AGENT_SDK_JAVA_VERSION=<version>`.

Inheriting the full JVM environment is simpler, but behavioral parity and subprocess predictability outweigh the convenience.

#### `Multi<AgentEvent> stream()`

Returns a `Multi.createFrom().emitter(...)` backed by a blocking `readLine()` loop on process stdout. The emitter runs on whatever thread subscribes — `buildEventStream()` shifts this to the worker pool via `runSubscriptionOn(Infrastructure.getDefaultWorkerPool())`.

**The load-bearing contract:** when `destroyForcibly()` is called from outside (cancellation or timeout), the OS closes the subprocess's stdout pipe. The blocked `readLine()` call immediately returns null (EOF), the loop exits, and the worker thread is freed. No additional signalling is needed.

**Emitter registration:**
```java
em.onTermination(() -> destroyForcibly());  // defense-in-depth
```

This fires when the emitter is terminated for any reason (cancellation, failure, completion). Combined with the `onCancellation()` handler in `buildEventStream()`, this is belt-and-suspenders.

**Read loop logic:**

```
resultMessageSeen = false
while (line = stdout.readLine()) != null:
    ParsedMessage parsed = parser.parse(line)
    switch parsed:
        case ParsedMessage.RegularMessage(var message):
            if message instanceof AssistantMessage am and !am.text().isEmpty():
                em.emit(new AgentEvent.TextDelta(am.text()))
            else if message instanceof ResultMessage:
                resultMessageSeen = true
                // do NOT break — let loop drain remaining lines
        case ParsedMessage.RateLimitEventMessage:
            LOG.debug("rate limit event received, continuing")
        case ParsedMessage.Control, ControlResponseMessage, EndOfStream:
            // not expected in one-shot mode; skip silently
// EOF reached:
if timedOut.get():
    em.fail(new AgentTimeoutException(effectiveTimeout))
else if process.exitValue() != 0 and !resultMessageSeen:
    em.fail(new AgentProcessException("claude exited with code " + process.exitValue()))
else:
    em.complete()
```

`process.exitValue()` is available immediately after stdout closes because stdout closure implies process termination. The `if timedOut.get()` check takes priority so that a timeout-triggered kill is reported as `AgentTimeoutException`, not `AgentProcessException` (even though exit code 137/SIGKILL would otherwise satisfy the non-zero check).

If the emitter is already terminated (because cancellation fired before EOF), the `em.fail()` / `em.complete()` calls are no-ops — Mutiny guarantees idempotent emitter termination.

**Stderr** is drained on a daemon thread; logged at WARN. No effect on the event stream.

#### `void destroyForcibly()`

Idempotent (AtomicBoolean guard). Actions:
1. `process.destroyForcibly()` — immediate SIGKILL
2. Delete MCP temp file (if it exists)

Used on: cancellation path, wall-clock timeout path.

#### `void close()`

Graceful shutdown. Actions:
1. Close stdout reader, stderr reader, stdin writer
2. `process.destroy()` — SIGTERM
3. `process.waitFor(5, TimeUnit.SECONDS)` — wait for graceful exit
4. `process.destroyForcibly()` if still alive
5. Delete MCP temp file (idempotent)

Used on: normal completion path and failure path (when the process has already exited naturally).

#### `long pid()`

Returns `process.pid()`. Used by `ClaudeAgentClient.activeProcessPids()` for IT test verification.

---

### Changes to `ClaudeAgentClient`

#### New field

```java
private final CopyOnWriteArraySet<ClaudeOneShotProcess> activeProcesses;
```

Initialised alongside `activeSessions` in **all three constructors** (CDI inject, test, protected no-arg). `activeSessions` (tracking `ClaudeAsyncClient` for the session path) is unchanged.

#### `buildEventStream(AgentSessionConfig config)` — full replacement

```
Before:
  ClaudeAsyncClient sdkClient = builder.build()
  activeSessions.add(sdkClient)
  Flux<String> textFlux = sdkClient.connect(prompt).textStream()
  [wall-clock timeout → sdkClient.close().subscribe()]
  [onCancellation → sdkClient.close().subscribe()]

After:
  ClaudeOneShotProcess proc = new ClaudeOneShotProcess(config, properties)
  activeProcesses.add(proc)
  [wall-clock timeout → timedOut.compareAndSet(false, true) + proc.destroyForcibly()]
  Multi<AgentEvent> raw = proc.stream()
      .runSubscriptionOn(Infrastructure.getDefaultWorkerPool())
  eventStream = raw
      .onCompletion().invoke(() -> {
          timeoutFuture.cancel(false)
          proc.close()
          activeProcesses.remove(proc)
      })
      .onFailure().invoke(t -> {
          timeoutFuture.cancel(false)
          proc.close()
          activeProcesses.remove(proc)
      })
      .onCancellation().invoke(() -> {
          proc.destroyForcibly()
          activeProcesses.remove(proc)
      })
```

Note: `timedOut` CAS and `AgentTimeoutException` conversion remain in `buildEventStream()`. The completion path (`proc.close()`) handles the case where `timedOut == true` — `proc.stream()` has already emitted `AgentTimeoutException` via the emitter, so `buildEventStream()`'s completion intercept converts it to failure identically to the current implementation.

The `timedOut` CAS race (wall-clock timeout firing simultaneously with natural process completion) is an accepted tradeoff inherited from the current design.

#### `shutdown()` (@PreDestroy)

```java
timeoutScheduler.shutdownNow();
activeProcesses.forEach(p -> { try { p.destroyForcibly(); } catch (Exception ignored) {} });
activeSessions.forEach(c -> { try { c.close().subscribe(); } catch (Exception ignored) {} });
```

One-shot processes are killed immediately at shutdown; session clients are gracefully closed.

#### `Set<Long> activeProcessPids()` (package-private, for IT tests)

```java
Set<Long> activeProcessPids() {
    return activeProcesses.stream()
        .map(ClaudeOneShotProcess::pid)
        .collect(Collectors.toUnmodifiableSet());
}
```

---

### Testing

#### `ClaudeOneShotProcessTest` (new, unit, `agent-claude/`)

- **Command construction**: verify exact flags from various `AgentSessionConfig` + `ClaudeAgentProperties` combinations; assert excluded flags are absent
- **MCP config**: verify temp file is written with correct JSON on construction; verify file is deleted by both `destroyForcibly()` and `close()` (idempotent: second call is a no-op, no exception)
- **JSON dispatch**: pipe synthetic `--output-format stream-json` lines through the parser; assert `TextDelta` events for `AssistantMessage`, stream completion for `ResultMessage`, silent skip for `RateLimitEventMessage` and unknown subtypes
- **Exit-code check**: subprocess that exits with code 1 without emitting `ResultMessage` → stream fails with `AgentProcessException`; subprocess that exits with code 0 after `ResultMessage` → stream completes normally
- **`destroyForcibly()`**: start `sleep 60` subprocess; call `destroyForcibly()`; assert `!process.isAlive()` within 200 ms; assert MCP temp file deleted
- **`destroyForcibly()` idempotence**: call twice; no exception on second call

#### `ClaudeAgentClientTest` (existing, unit)

No changes. `streamFactory` bypasses `ClaudeOneShotProcess`; all existing semaphore and cancellation tests continue working.

#### `ClaudeAgentClientIT` (integration)

Extend with a test that:
1. Records `activeProcessPids()` before subscribing
2. Subscribes and immediately cancels
3. Asserts within 500 ms that no PID from step 1 is alive via `ProcessHandle.of(pid).map(ProcessHandle::isAlive).orElse(false)`

---

### What does NOT change

- `AgentProviderChatModel` in eidos eval — correct as-is; `await().atMost(timeout)` is the right pattern; the platform fix makes cancellation effective.
- `ClaudeAgentSession` / `openSession()` path — separate concern, different lifecycle semantics, out of scope.
- `AgentProvider` / `AgentSession` SPIs — no signature changes.
- `ClaudeAgentProvider` — no changes.

---

### Constraints verified

- All current `invoke()` callers use `AgentSessionConfig.of(systemPrompt, userPrompt)` — no MCP servers, no permission callbacks. One-shot mode is safe.
- `ClaudeAgentChatModel` (langchain4j adapter) uses `openSession()`, not `invoke()` — unaffected.
- MCP types (`Stdio`, `Sse`, `Http`) in `AgentSessionConfig` are structurally supported; `--mcp-config` covers them in one-shot mode. In-process SDK MCP servers require the bidirectional control channel and are not representable via `AgentMcpServer` — no caller uses them.
- `ParsedMessage` is a sealed interface with 5 permitted types: `RegularMessage`, `Control`, `ControlResponseMessage`, `RateLimitEventMessage`, `EndOfStream`. Only `RegularMessage` and `RateLimitEventMessage` are expected in one-shot output; the rest are defensively skipped.

---

### Platform coherence

`ClaudeOneShotProcess` is package-private in `agent-claude/`. Consumers interact through the `AgentProvider` SPI only. No SPI changes, no cross-repo impact, no Flyway.

The one-shot/session split mirrors HTTP request/response vs WebSocket: use the right transport for the use case. The SPI contract (`AgentProvider.invoke()` → `Multi<AgentEvent>`) is unchanged.
