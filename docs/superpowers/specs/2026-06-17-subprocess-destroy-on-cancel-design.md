# Subprocess Destroy on Cancellation — Design Spec
**Issue:** eidos#52  
**Branch:** issue-52-subprocess-destroy-on-cancel  
**Repos affected:** casehub-platform (primary), casehub-eidos (none — eval behaviour unchanged)

---

## Problem

When `AgentProviderChatModel.doChat()` times out via `await().atMost(Duration)`, Mutiny cancellation propagates to `ClaudeAgentClient.buildEventStream()`, which calls `sdkClient.close().subscribe()`. `DefaultClaudeAsyncClient.close()` schedules the actual cleanup on Reactor's bounded-elastic pool, where `StreamingTransport.close()` runs `process.destroy()` + `waitFor(5s)` + `destroyForcibly()`.

When parallel enrichment calls all timeout simultaneously (observed: 16 at once in `evaluateRealWorldScenarios`), 16 subprocesses enter the "SIGTERM sent, waiting 5 seconds" state simultaneously. The 17th call (ProximityJudge) starts a new subprocess into a resource-starved environment and hangs indefinitely.

## Root Cause — Architectural Mismatch

`buildEventStream()` uses the SDK's **session mode** (`--input-format stream-json`): a persistent bidirectional subprocess designed for multi-turn conversations. It must be explicitly closed; close is async with a 5-second grace period before `destroyForcibly()`.

`invoke()` is **single-turn**. The right subprocess mode is **one-shot** (`-p "prompt" --output-format stream-json`): the subprocess exits naturally on completion. If cancelled, you call `destroyForcibly()` on a `Process` you already own — immediately, synchronously, with no grace period.

## Design

### Two subprocess modes — clearly separated

| Path | Mode | Process ownership | Cancel |
|---|---|---|---|
| `run()` → `buildEventStream()` | One-shot (`-p "prompt"`) | `ClaudeAgentClient` owns `Process` | `process.destroyForcibly()` — immediate |
| `openSession()` → `ClaudeAgentSession` | Session mode (SDK) | SDK owns | Unchanged — out of scope |

The output format (`--output-format stream-json`) is identical in both modes. SDK parsing classes (`ControlMessageParser`, `ParsedMessage`, `AssistantMessage`) are reused as-is.

---

### New class: `ClaudeOneShotProcess` (package-private, `agent-claude/`)

Owns a single-invocation subprocess lifetime.

**Construction:** accepts `AgentSessionConfig config`, `ClaudeAgentProperties properties`, `ObjectMapper objectMapper`. Builds and starts the subprocess immediately in the constructor.

**Command construction** (ported from `StreamingTransport.buildStreamingCommand()`, minus the session-mode flags):
```
<binary> -p "<userPrompt>" --output-format stream-json [--system-prompt "<sys>"]
         [--model "<model>"] [--mcp-config /tmp/file.json] [--verbose]
```
No `--input-format stream-json` (one-shot, no stdin protocol).  
MCP servers (`Stdio`, `Sse`, `Http`) written to a temp JSON file via `--mcp-config`, cleaned up in `close()`.

**`Multi<AgentEvent> stream()`:** reads stdout line-by-line from `process.getInputStream()`. Each non-empty line is parsed via SDK's `ControlMessageParser`. Lines that parse to `AssistantMessage` produce `AgentEvent.TextDelta(message.text())`. Lines that parse to `ResultMessage` complete the stream. Parse failures log a warning and continue. Stream completes when stdout reaches EOF (process exited naturally).

**Stderr drain:** reads stderr on a daemon thread; logs at WARN level. No effect on event stream.

**`long pid()`:** returns `process.pid()` for logging/diagnostics.

**`void destroyForcibly()`:** calls `process.destroyForcibly()` — synchronous, immediate, SIGKILL. Idempotent.

**`void close()`:** graceful cleanup — close streams, `process.destroy()`, `process.waitFor(5s)`, `process.destroyForcibly()` if needed. Used on normal completion and failure paths. **Not used on cancellation.**

---

### Changes to `ClaudeAgentClient`

**New field:**
```java
private final CopyOnWriteArraySet<ClaudeOneShotProcess> activeProcesses;
```
Initialised alongside `activeSessions` in all constructors. `activeSessions` (tracks `ClaudeAsyncClient` for the session path) is unchanged.

**`buildEventStream(AgentSessionConfig config)`** — replace SDK usage:

```
Before: ClaudeAsyncClient sdkClient = builder.build();
        activeSessions.add(sdkClient);
        ...
        Flux<String> textFlux = sdkClient.connect(prompt).textStream();
        // onCancellation → sdkClient.close().subscribe()

After:  ClaudeOneShotProcess proc = new ClaudeOneShotProcess(config, properties, objectMapper);
        activeProcesses.add(proc);
        ...
        // Multi<AgentEvent> = proc.stream()
        // wall-clock timeout → proc.destroyForcibly()
        // onCompletion / onFailure → proc.close() + activeProcesses.remove(proc)
        // onCancellation → proc.destroyForcibly() + activeProcesses.remove(proc)
```

The wall-clock `timeoutScheduler` still fires at `effectiveTimeout`. Its action changes from `sdkClient.close().subscribe()` to: set `timedOut.compareAndSet(false, true)` + `proc.destroyForcibly()`.

The `timedOut: AtomicBoolean` is retained. When `destroyForcibly()` kills the process, stdout closes, the blocking `readLine()` loop in `proc.stream()` sees EOF and emits completion. `buildEventStream()` intercepts that completion via `onCompletion().call(...)`: if `timedOut.get()` is true, it fails the stream with `AgentTimeoutException(effectiveTimeout)` instead. This is the same conversion pattern as the current implementation.

**`shutdown()` (@PreDestroy):**
```java
timeoutScheduler.shutdownNow();
activeProcesses.forEach(p -> { try { p.destroyForcibly(); } catch (Exception ignored) {} });
activeSessions.forEach(c -> { try { c.close().subscribe(); } catch (Exception ignored) {} });
```

**`openSession()` / `ClaudeAgentSession`:** no changes.

**Test constructor** `ClaudeAgentClient(ClaudeAgentProperties, Function<AgentSessionConfig, Multi<AgentEvent>>)`: add `activeProcesses` initialisation. `streamFactory` path bypasses `ClaudeOneShotProcess`, so all existing unit tests continue working without modification.

---

### Testing

**`ClaudeOneShotProcessTest` (new, unit):**
- Command construction: verify flags from various `AgentSessionConfig` + property combinations
- JSON parsing: pipe synthetic `--output-format stream-json` lines into stdin, assert `TextDelta` events
- `destroyForcibly()`: start `sleep 60` as subprocess, call `destroyForcibly()`, assert `!process.isAlive()` within 100 ms
- MCP config: verify temp file is written with correct JSON and cleaned up on `close()`
- Graceful close: verify `close()` waits for process exit (use a subprocess that exits immediately)

**`ClaudeAgentClientTest` (existing, unit):** no changes required — `streamFactory` bypasses `ClaudeOneShotProcess`.

**`ClaudeAgentClientIT` (existing, integration):** extend to verify that after a cancelled subscription, the subprocess PID (captured via `proc.pid()` exposed through a test hook or log inspection) is no longer alive. Assert within 500 ms.

---

### What does NOT change

- `AgentProviderChatModel` in eidos eval — correct as-is. `await().atMost(timeout)` is the right pattern; the platform fix makes cancellation effective.
- `ClaudeAgentSession` / `openSession()` path — separate concern, different lifecycle semantics, out of scope for eidos#52.
- `AgentProvider` / `AgentSession` SPIs — no signature changes.
- `ClaudeAgentProvider` — no changes (delegates to `ClaudeAgentClient.run()`).

---

### Constraints verified

- All current `invoke()` callers use `AgentSessionConfig.of(systemPrompt, userPrompt)` — no MCP servers, no permission callbacks. One-shot mode is safe.
- `ClaudeAgentChatModel` (langchain4j adapter) uses `openSession()`, not `invoke()` — unaffected.
- MCP types (`Stdio`, `Sse`, `Http`) in `AgentSessionConfig` are structurally supported; `--mcp-config` covers them in one-shot mode. In-process SDK MCP servers (`McpSdkServerConfig`) require the bidirectional control channel and are not representable via `AgentMcpServer` — no caller uses them.

---

### Platform coherence

`ClaudeOneShotProcess` lives in `agent-claude/` (same module as `ClaudeAgentClient`). Package-private — consumers interact through `AgentProvider` SPI only. No SPI changes, no cross-repo impact, no Flyway.

The one-shot/session split is analogous to HTTP request/response vs WebSocket: use the right protocol for the use case. The SPI contract (`AgentProvider.invoke()` → `Multi<AgentEvent>`) is unchanged.
