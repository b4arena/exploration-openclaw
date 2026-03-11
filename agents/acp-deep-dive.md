# ACP Deep Dive — ca-leash vs ACP/acpx

How OpenClaw's Agent Client Protocol (ACP) system works internally, and how it compares to ca-leash for running Claude Code agents in containers.

> **Cross-references**: [`coding-agents.md`](coding-agents.md) (PTY/exec path), [`security-and-sandbox.md`](../architecture/security-and-sandbox.md) (sandbox system), [`sandbox-container-lifecycle.md`](../deployment/sandbox-container-lifecycle.md) (container lifecycle)

---

## 1. Three Paths to Running Coding Agents in OpenClaw

OpenClaw offers three distinct integration paths for running Claude Code or other coding agents:

| Path | Mechanism | Where it runs | This doc? |
|---|---|---|---|
| **PTY/exec** | `exec` tool + `pty:true` | Sandbox container or host | See `coding-agents.md` |
| **ACP/acpx** | `sessions_spawn runtime="acp"` + acpx backend plugin | Gateway host (not sandbox) | **This doc** |
| **ca-leash** | Custom headless controller via `claude-agent-sdk` | Podman container (any host) | Comparison target |

The key structural difference: PTY/exec runs a process inside whatever execution context the agent already has (potentially sandboxed). ACP runs its harness on the Gateway host itself. ca-leash runs inside a container managed independently of OpenClaw.

---

## 2. ACP Architecture Internals

### 2.1 The acpx Backend Plugin

ACP is implemented through the `acpx` backend plugin at `extensions/acpx/`. The plugin exposes an `AcpxRuntime` class that implements the `AcpRuntime` interface from `openclaw/plugin-sdk/acpx`.

### 2.2 Spawn Model — New Process Per Turn

The critical architectural choice: **acpx spawns a fresh CLI process for every turn**, not a persistent daemon.

From `extensions/acpx/src/runtime.ts:301-313`, `runTurn()`:

```typescript
const child = spawnWithResolvedCommand(
  {
    command: this.config.command,   // acpx binary
    args,                           // buildPromptArgs() output
    cwd: state.cwd,
  },
  this.spawnCommandOptions,
);
child.stdin.end(input.text);        // prompt on stdin
```

The full args built by `buildPromptArgs()` (`runtime.ts:607-632`):

```
acpx --format json --json-strict --cwd <workdir>
     --permission-mode approve-reads
     --non-interactive-permissions fail
     --ttl <seconds>
     prompt --session <sessionName> --file -
```

The prompt text is written to the process's stdin. Output (events) is streamed back via stdout as NDJSON. Each turn = one `acpx ... prompt --session <name> --file -` invocation.

Source: `extensions/acpx/src/runtime.ts:274-382` (`runTurn()`), `extensions/acpx/src/runtime.ts:607-632` (`buildPromptArgs()`)

### 2.3 Agent Command Resolution

acpx has a built-in mapping from agent name to launch command, overridable via `~/.acpx/config.json`:

```typescript
const ACPX_BUILTIN_AGENT_COMMANDS: Record<string, string> = {
  codex: "npx @zed-industries/codex-acp",
  claude: "npx -y @zed-industries/claude-agent-acp",
  gemini: "gemini",
  opencode: "npx -y opencode-ai acp",
  pi: "npx pi-acp",
};
```

Resolution order: user config overrides → built-in map → agent name as-is.

Source: `extensions/acpx/src/runtime-internals/mcp-agent-command.ts:5-11` (built-in map), `extensions/acpx/src/runtime-internals/mcp-agent-command.ts:86-99` (`resolveAcpxAgentCommand()`)

### 2.4 Protocol — NDJSON Over Stdio

Events flow from the acpx process stdout as newline-delimited JSON. Event types parsed by `parsePromptEventLine()`:

| Event type | Meaning |
|---|---|
| `text` | Agent response text chunk |
| `thought` | Agent reasoning/thinking chunk |
| `tool_call` | Tool invocation event |
| `tool_call_update` | Tool invocation status update |
| `usage_update` | Context window usage (`used/size`) |
| `done` | Turn complete (with `stopReason`) |
| `error` | Runtime error (with `code`, `retryable`) |
| `agent_message_chunk` | Streaming output chunk |
| `agent_thought_chunk` | Streaming thought chunk |
| `available_commands_update` | Available slash commands changed |
| `current_mode_update` | Agent mode changed |
| `config_option_update` | Config option changed |
| `session_info_update` | Session metadata update |
| `plan` | Plan/task list from agent |

The parser also supports a JSON-RPC 2.0 envelope format: a `method: "session/update"` with a `params.update` payload is unwrapped and routed by its `sessionUpdate` tag.

Non-JSON lines are treated as status text events (not errors).

Source: `extensions/acpx/src/runtime-internals/events.ts:165-319` (`parsePromptEventLine()`)

### 2.5 Session Management — Handle Encoding

Session state is encoded as base64url JSON in the `runtimeSessionName` field of `AcpRuntimeHandle`:

```
acpx:v1:<base64url(JSON)>
```

The JSON payload (`AcpxHandleState`):

```json
{
  "name": "agent:claude:acp:550e8400-...",   // session key used in acpx CLI
  "agent": "claude",                           // agent id (e.g. "claude", "codex")
  "cwd": "/home/user/project",                 // working directory
  "mode": "persistent" | "oneshot",            // session lifetime mode
  "acpxRecordId": "...",                       // acpx internal record
  "backendSessionId": "...",                   // acpx backend session
  "agentSessionId": "..."                      // agent's own session id
}
```

Session lifecycle via acpx CLI subcommands:
- `acpx sessions ensure --name <name>` — resume or initialize
- `acpx sessions new --name <name>` — create fresh
- `acpx sessions close <name>` — close
- `acpx cancel --session <name>` — cancel active turn
- `acpx status --session <name>` — inspect state

Source: `extensions/acpx/src/runtime.ts:73-118` (encode/decode), `extensions/acpx/src/runtime.ts:195-272` (`ensureSession()`)

### 2.6 MCP Server Injection

When `mcpServers` are configured in the acpx plugin config, the plugin wraps the agent command with an MCP proxy. The proxy intercepts the `session/new` JSON-RPC call and injects `mcpServers` config before forwarding.

```typescript
// builds: node /path/to/mcp-proxy.mjs --payload <base64url>
const resolved = buildMcpProxyAgentCommand({
  targetCommand,                    // e.g. "npx -y @zed-industries/claude-agent-acp"
  mcpServers: toAcpMcpServers(this.config.mcpServers),
});
```

Source: `extensions/acpx/src/runtime-internals/mcp-agent-command.ts:101-113` (`buildMcpProxyAgentCommand()`), `extensions/acpx/src/runtime.ts:651-675` (`resolveRawAgentCommand()`)

### 2.7 Permission Model

Default values from `extensions/acpx/src/config.ts:60-61`:

```typescript
const DEFAULT_PERMISSION_MODE: AcpxPermissionMode = "approve-reads";
const DEFAULT_NON_INTERACTIVE_POLICY: AcpxNonInteractivePermissionPolicy = "fail";
```

| Config key | Default | Options |
|---|---|---|
| `permissionMode` | `approve-reads` | `approve-all`, `approve-reads`, `deny-all` |
| `nonInteractivePermissions` | `fail` | `fail`, `deny` |

Exit code 5 from the acpx process = permission denied (`ACPX_EXIT_CODE_PERMISSION_DENIED = 5`). When hit, OpenClaw surfaces the error with guidance to adjust `permissionMode`.

Source: `extensions/acpx/src/runtime.ts:49` (exit code 5), `extensions/acpx/src/runtime.ts:54-71` (error formatting), `extensions/acpx/src/config.ts:60-61` (defaults)

---

## 3. ACP Spawn Flow (`sessions_spawn`)

The full call chain when an agent triggers `sessions_spawn` with `runtime: "acp"`:

```
sessions_spawn(runtime="acp", task="...", agentId="claude")
  │
  ▼
spawnAcpDirect()                          [src/agents/acp-spawn.ts:305]
  │
  ├─ Policy gate 1: ACP enabled?          [acp-spawn.ts:314]
  │    isAcpEnabledByPolicy(cfg)
  │
  ├─ Policy gate 2: Requester sandboxed?  [acp-spawn.ts:328]
  │    resolveAcpSpawnRuntimePolicyError()
  │    → if sandboxed: return "forbidden"
  │
  ├─ Policy gate 3: Agent in allowlist?   [acp-spawn.ts:364]
  │    resolveAcpAgentPolicyError(cfg, agentId)
  │
  ├─ Generate session key                 [acp-spawn.ts:372]
  │    "agent:<agentId>:acp:<uuid>"
  │
  ├─ Thread binding (optional)            [acp-spawn.ts:376]
  │    prepareAcpThreadBinding()
  │    → resolves Discord/Telegram conversation
  │
  ├─ Gateway: sessions.patch              [acp-spawn.ts:399]
  │    register session in Gateway
  │
  ├─ acpManager.initializeSession()       [acp-spawn.ts:424]
  │    → runtime.ensureSession()          [manager.core.ts]
  │    → acpx sessions ensure --name ...  [runtime.ts:195]
  │
  ├─ Bind to thread (if requested)        [acp-spawn.ts:437]
  │    bindingService.bind()
  │    → creates Discord thread / Telegram topic
  │
  └─ Gateway: agent method                [acp-spawn.ts:553]
       dispatch initial task to new session
```

Source: `src/agents/acp-spawn.ts` (full file), `src/acp/control-plane/manager.core.ts` (session lifecycle)

---

## 4. The Sandbox Constraint — What It Actually Means

### 4.1 The Policy Gate

From `src/agents/acp-spawn.ts:102-104`, the explicit policy:

```typescript
if (requesterSandboxed) {
  return 'Sandboxed sessions cannot spawn ACP sessions because runtime="acp" runs on the host. Use runtime="subagent" from sandboxed sessions.';
}
```

This is a **one-way gate**: sandboxed agents cannot spawn ACP sessions.

### 4.2 What "Sandboxed" Means Here

OpenClaw's "sandbox" is its Docker-based code execution isolation (see `security-and-sandbox.md` §4). When an agent session runs inside a sandbox container (`sandbox.mode: "all"`), it is tagged as sandboxed. That tag is checked in `resolveSandboxRuntimeStatus()` at spawn time.

### 4.3 What This Does NOT Mean

- It does **not** mean Claude Code can't run in containers generally
- It does **not** mean ACP is inherently uncontainerizable
- It means OpenClaw's sandbox layer cannot wrap the acpx harness in the current architecture

### 4.4 Why This Constraint Exists

ACP runs its harness (acpx + the agent ACP adapter) as a process on the **Gateway host**, using the Gateway's Node.js runtime and network access. OC's sandbox system routes `exec` calls through `docker exec` into a container. These two execution models are orthogonal — the sandbox provides per-command container isolation, while ACP provides per-session agent isolation via the harness protocol.

ca-leash sidesteps this entirely because it doesn't use OpenClaw's sandbox subsystem. It manages its own Podman containers through systemd Quadlets.

---

## 5. Detailed Comparison: ca-leash vs ACP/acpx

| Dimension | ca-leash | ACP/acpx |
|---|---|---|
| **SDK** | `claude-agent-sdk` (Python) | `acpx` CLI wrapping `claude-agent-acp` (Node) |
| **Process model** | Persistent daemon per session | New process per turn |
| **IPC** | File-based (JSON files in `ipc/input/`, JSONL output stream) | stdio pipes per process |
| **Session resume** | `claude_session_id` stored in `state.json` | `runtimeSessionName` base64url handle |
| **Default permissions** | `bypassPermissions` (full auto) | `approve-reads` (configurable) |
| **Container-native** | Yes — runs inside Podman, installed as wheel | No — runs on Gateway host |
| **Orchestration** | Direct CLI or IPC messages | OpenClaw Gateway mediated |
| **Thread binding** | N/A | Discord/Telegram thread association |
| **MCP injection** | Via `--mcp-servers` CLI option | MCP proxy intercepting `session/new` JSON-RPC |
| **Runtime reconfig** | IPC messages (`set_model`, etc.) | `/acp set`, `/acp model`, `/acp permissions` |
| **Cancel/interrupt** | `_interrupt` sentinel file in IPC dir | `acpx cancel --session <name>` |
| **Multi-harness** | Claude Code only (by design) | Claude, Codex, OpenCode, Pi, Gemini |
| **Language** | Python | TypeScript / Node.js |
| **Crash resilience** | File-based state survives process crashes | Session state in acpx; turn state lost on crash |
| **Follow-up latency** | Zero (persistent connection) | Process spawn overhead per turn |

### 5.1 ca-leash Session Model

ca-leash's `SessionRunner` (`ca_leash/runner.py:18`) is a long-lived async process:
- Maintains a persistent `ClaudeCodeDriver` connection to claude-agent-sdk
- Polls `ipc/input/` directory for new IPC messages
- Appends responses to `output.jsonl`
- Stores `claude_session_id` in `state.json` for crash recovery

### 5.2 acpx Session Model

acpx's `runTurn()` (`extensions/acpx/src/runtime.ts:274`) spawns a new process:
- `acpx prompt --session <name> --file -` with prompt on stdin
- stdout streams NDJSON events until `done` or `error`
- Process exits; next turn spawns fresh

The session state (Claude's conversation history) lives inside the acpx/claude-agent-acp layer, referenced by `sessionName`.

---

## 6. Could ACP Replace ca-leash?

### 6.1 Current Blockers

**ACP runs on host, not in containers.** The architectural gap is in `spawnAcpDirect()` — it assumes the Gateway host environment. To containerize ACP, one would need to either:

1. Run acpx inside the container (pointing `config.command` to a containerized acpx binary)
2. Or run OpenClaw Gateway itself inside the target container environment

Neither is currently supported by OpenClaw's configuration model.

### 6.2 What We'd Gain

- **Multi-harness support**: Claude, Codex, OpenCode, Pi, Gemini via the same integration point
- **OpenClaw chat integration**: `/acp cancel`, `/acp steer`, `/acp model` operator controls from chat
- **Thread binding**: Automatic Discord thread / Telegram topic per ACP session
- **MCP injection**: Declarative MCP server config, no CLI flag management
- **OpenClaw telemetry**: Session transcripts, observability, token usage tracking

### 6.3 What We'd Lose

- **Persistent daemon model**: ca-leash's zero-latency follow-ups vs. ACP's per-turn spawn overhead
- **File-based IPC**: ca-leash's state survives process crashes; acpx session state can be lost
- **Container-native design**: ca-leash is purpose-built for Podman; ACP assumes host execution
- **Python ecosystem**: ca-leash integrates naturally with Python tooling (uv, pyproject.toml, Pydantic models)
- **`bypassPermissions` default**: ca-leash is fully non-interactive by default; ACP requires configuration to reach equivalent permissiveness

---

## 7. Could ca-leash Be Wrapped as an acpx Agent?

### 7.1 The acpx Agent Protocol

acpx supports custom agent commands via `--agent <command>` or config overrides. The agent process must implement the ACP stdio protocol:

- Read initial `session/new` JSON-RPC on stdin
- Write NDJSON events to stdout (`text`, `thought`, `tool_call`, `usage_update`, `done`, `error`)
- Support `session/close` lifecycle

### 7.2 The Adapter Sketch

A thin adapter process could translate between the two worlds:

```
OpenClaw Gateway
    │
    │ (acpx CLI)
    ▼
acpx --agent "ca-leash-acp-adapter" prompt --session ...
    │
    │ JSON-RPC stdin / NDJSON stdout
    ▼
ca-leash-acp-adapter  ← new thin adapter
    │
    │ file IPC (ipc/input/*.json, output.jsonl)
    ▼
ca-leash SessionRunner (inside Podman container)
    │
    │ claude-agent-sdk
    ▼
Claude Code
```

The adapter would:
1. Receive `session/new` JSON-RPC → start or resume a ca-leash session
2. Receive prompt text → write to ca-leash IPC input directory
3. Poll ca-leash `output.jsonl` → translate to acpx NDJSON events
4. On `session/close` → write ca-leash `_close` sentinel

### 7.3 Feasibility

- **Protocol fit**: ca-leash's file IPC maps cleanly to ACP's stdin/stdout events
- **Session resume**: ca-leash's `claude_session_id` + ca-leash session ID map to acpx `sessionName`
- **Main challenge**: Streaming — ca-leash uses file polling; the adapter would need to inotify-watch or poll the JSONL output file to forward events in real time
- **Implementation cost**: ~200-400 lines of Python or Node.js for the adapter

---

## 8. Recommendation

| Use Case | Recommendation |
|---|---|
| Container-native headless agent execution | **Keep ca-leash** — purpose-built, container-native, persistent daemon |
| Chat-driven orchestration (Discord/Telegram threads) | **Consider ACP** — thread binding, operator controls, multi-harness |
| Multi-harness flexibility (Codex, OpenCode, etc.) | **ACP natively** or **ca-leash adapter** if container isolation required |
| Unified integration (both worlds) | **Hybrid adapter**: OpenClaw orchestrates via ACP, adapter delegates to ca-leash containers |

**Current recommendation for b4arena:**

1. **Keep ca-leash** as the primary execution engine for container-native headless agent execution. It is already deployed, tested, and fits the Podman/systemd Quadlet infrastructure.

2. **Consider ACP** if/when the team needs chat-driven agent orchestration from Discord or Telegram (thread binding, `/acp cancel`, `/acp steer` operator commands).

3. **Consider the hybrid adapter** as a future integration point if ACP's operator controls and chat integration become valuable without sacrificing ca-leash's container isolation and persistent daemon model.

---

## References

### Source Files

| File | Topics Covered |
|---|---|
| `extensions/acpx/src/runtime.ts` | Core acpx runtime: `runTurn()`, `ensureSession()`, `buildPromptArgs()`, handle encode/decode |
| `extensions/acpx/src/runtime-internals/process.ts` | Process spawning via `child_process.spawn`, abort handling |
| `extensions/acpx/src/runtime-internals/events.ts` | NDJSON event parsing: all event types, JSON-RPC envelope support |
| `extensions/acpx/src/runtime-internals/mcp-agent-command.ts` | Agent→command mapping, MCP proxy command builder |
| `extensions/acpx/src/config.ts` | Config resolution, defaults (`approve-reads`, `fail`), `McpServerConfig` type |
| `src/agents/acp-spawn.ts` | Spawn flow: `spawnAcpDirect()`, policy gates, session key generation, thread binding |
| `src/acp/control-plane/manager.core.ts` | Session lifecycle: `AcpSessionManager`, `initializeSession()`, `resolveSession()` |
| `src/acp/control-plane/manager.types.ts` | Manager types: `AcpSessionResolution`, `SessionEntry`, `AcpSessionRuntimeOptions` |

### ca-leash Source Files

| File | Topics Covered |
|---|---|
| `ca_leash/runner.py` | `SessionRunner`: persistent daemon, IPC poll loop, `ClaudeCodeDriver` connection |
| `ca_leash/session.py` | Session creation, state management, `claude_session_id` persistence |
| `ca_leash/ipc.py` | File-based IPC: input JSON files, `_close`/`_interrupt` sentinels, JSONL output |
| `ca_leash/claude_driver.py` | `ClaudeCodeDriver`: wraps `claude-agent-sdk` |

### Exploration Cross-References

| Doc | Relevant Sections |
|---|---|
| [`coding-agents.md`](coding-agents.md) | PTY/exec path for running Claude Code via `exec` tool |
| [`security-and-sandbox.md`](../architecture/security-and-sandbox.md) | Sandbox system (§4), Docker runtime constraints (§4.2), sandbox-vs-host isolation |
| [`sandbox-container-lifecycle.md`](../deployment/sandbox-container-lifecycle.md) | Container creation timing, scoping, reuse patterns |
| [`subagent-deep-dive.md`](subagent-deep-dive.md) | `sessions_spawn` with `runtime="subagent"` as the alternative to ACP from sandboxed sessions |
