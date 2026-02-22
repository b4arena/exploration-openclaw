# Process Execution: tmux, PTY & Background Processes

## 1. Overview – Three Execution Paradigms

OpenClaw provides three ways to run external processes:

| Paradigm | Tool | Use Case | Agent Blocked? |
|---|---|---|---|
| **Synchronous** | `exec` (default) | Short commands (`ls`, `git status`) | Yes |
| **Background + Event** | `exec` + `background: true` | Long-running processes, parallel work | No – event-driven wake-up |
| **tmux (Skill)** | `exec` → `tmux send-keys` | SSH-persistent sessions, legacy pattern | No – manual polling |

**Recommended:** Background + Event for most async work. tmux only for special cases.

---

## 2. Background Process Lifecycle (Event-Driven)

### 2.1 The Flow

```
Agent starts command
  → exec({ command: "...", background: true })
  → Immediate return: { sessionId: "abc", status: "running" }
  → Agent is FREE – can do other work or sleep (ZERO token cost)

Process exits
  → maybeNotifyOnExit() fires
  → enqueueSystemEvent(summary)         ← result queued
  → requestHeartbeatNow()               ← agent wake-up (250ms delay)

Agent wakes up
  → System events injected into next prompt automatically
  → Agent sees exit code + output excerpt
  → Can use `process log` or `process poll` for full details
```

Source: `src/agents/bash-tools.exec-runtime.ts:228-251` (`maybeNotifyOnExit`), `:514` (`maybeNotifyOnExit` call on exit); `src/agents/bash-tools.exec.ts:1022-1028` (background return shape)

### 2.2 Key: No Polling Required

The agent does **not** poll for completion. The heartbeat system (same system that drives HEARTBEAT.md) delivers a wake-up signal when the process exits. System events are a separate channel alongside user messages – they get injected into the agent prompt without any user action.

Source: `src/infra/heartbeat-wake.ts` (imported at `src/agents/bash-tools.exec-runtime.ts:5`), `src/infra/system-events.ts` (imported at `src/agents/bash-tools.exec-runtime.ts:7`)

### 2.3 Configuration Parameters

| Parameter | Default | Description |
|---|---|---|
| `background` | `false` | Immediately background the process |
| `yieldMs` | `10000` | Background after N ms if not finished (alternative to `background`) |
| `notifyOnExit` | `true` | Wake agent via heartbeat when process exits |
| `notifyOnExitEmptySuccess` | `false` | Skip notification if exit code 0 and no output |
| `pty` | `false` | Allocate pseudo-terminal (for interactive CLIs) |

Source: `docs/tools/exec.md` (Parameters + Config sections); `src/agents/bash-tools.exec.ts:83-84` (`notifyOnExit`/`notifyOnExitEmptySuccess` types), `:231-232` (defaults), `:245-255` (schema with `yieldMs`, `background`, `pty`)

### 2.4 Comparison of Wait Strategies

| Strategy | Token Cost | Blocking? | Recommended? |
|---|---|---|---|
| `exec` synchronous | None extra, but agent is blocked | Yes | Only for short commands |
| `exec` + `background: true` | **Zero** during wait | No – event-driven | **Yes** |
| `process poll` with timeout | Wastes one tool call, spins at 250ms | Yes (up to 120s) | Only for very short waits | <!-- src/agents/bash-tools.process.ts:74 MAX_POLL_WAIT_MS=120_000 -->
| `process poll` without timeout | One tool call | No – immediate return | For status checks |

---

## 3. PTY Support (Native Interactive CLI)

### 3.1 What PTY Enables

PTY (pseudo-terminal) allows agents to interact with terminal applications that require a TTY – e.g., Claude Code CLI, interactive REPLs, TUI apps.

```json
// Start interactive CLI in background with PTY
{ "tool": "exec", "command": "claude", "pty": true, "background": true }
```

### 3.2 The `process` Tool – Controlling Running Sessions

| Action | Purpose |
|---|---|
| `list` | Enumerate running and finished sessions |
| `poll` | Check output and exit status (optional timeout) |
| `log` | Retrieve aggregated output (supports pagination) |
| `write` | Send raw data to stdin |
| `send-keys` | Send TTY key sequences (tmux-style encoding) |
| `submit` | Send carriage return |
| `paste` | Bracketed paste for multi-line text |
| `kill` | Terminate session |
| `clear` / `remove` | Clean up session from registry |

Source: `src/agents/bash-tools.process.ts:158-167` (action union type), `:181` (list), `:282` (poll), `:376` (log), `:446` (write), `:472` (send-keys), `:511` (submit), `:532` (paste), `:565` (kill), `:599` (clear), `:619` (remove). See also `docs/tools/exec.md` (Send keys / Submit / Paste examples)

### 3.3 Key Encoding

The `send-keys` action supports tmux-style key names:

```json
{ "tool": "process", "action": "send-keys", "sessionId": "abc",
  "keys": ["hello world", "Enter"] }
```

Supported keys include:
- Characters: `"h"`, `"e"`, `"l"`, `"l"`, `"o"`
- Named keys: `Enter`, `Escape`, `Tab`, `Backspace`, `Up`, `Down`, `Left`, `Right`
- Modifiers: `C-c` (Ctrl+C), `C-d` (Ctrl+D), `M-x` (Alt+X), `S-Tab` (Shift+Tab)
- Function keys: `F1`–`F12`, `Home`, `End`, `PageUp`, `PageDown`
- Hex bytes: `0x1b` (for raw escape sequences)

Source: `src/agents/pty-keys.ts:17-72` (`namedKeyMap`), `:74-80` (`modifiableNamedKeys`), `:8-9` (bracketed paste constants)

### 3.4 Pattern: Parallel Claude Code Sessions via PTY

```
Agent:
  1. exec({ command: "claude 'Fix bug #1'", pty: true, background: true })
     → sessionId: "s1"
  2. exec({ command: "claude 'Fix bug #2'", pty: true, background: true })
     → sessionId: "s2"
  3. Agent sleeps – ZERO token cost

  ... Claude Code sessions work autonomously ...

  4. Session "s1" exits → heartbeat → agent wakes, sees result
  5. Session "s2" exits → heartbeat → agent sees second result
```

---

## 4. tmux Skill (Legacy / Special Cases)

### 4.1 What It Is

The tmux skill is **not a tool** – it's documentation + two helper scripts that teach the agent how to use tmux commands via the `exec` tool.

```
skills/tmux/
├── SKILL.md              ← Instructions for the agent
└── scripts/
    ├── wait-for-text.sh  ← Polls tmux pane for text patterns (with timeout)
    └── find-sessions.sh  ← Discovers tmux sessions across multiple sockets
```

Source: `skills/tmux/SKILL.md`, `skills/tmux/scripts/wait-for-text.sh`, `skills/tmux/scripts/find-sessions.sh`

### 4.2 Helper Scripts

**`wait-for-text.sh`** – Synchronization primitive: waits for a regex pattern to appear in a tmux pane. (`skills/tmux/scripts/wait-for-text.sh:14` default timeout=15s, `:25` default interval=0.5s)

```bash
./wait-for-text.sh -t worker-3:0.0 -p "connected" -T 30
# Polls every 0.5s, timeout after 30s
```

**`find-sessions.sh`** – Discovers tmux sessions, supports multi-socket scanning. (`skills/tmux/scripts/find-sessions.sh:13`)

```bash
./find-sessions.sh -A -q worker    # Find all sessions matching "worker"
```

Environment: `OPENCLAW_TMUX_SOCKET_DIR` (primary), fallback to `${TMPDIR}/openclaw-tmux-sockets` (`skills/tmux/scripts/find-sessions.sh:23`)

### 4.3 Documented Session Pattern

The SKILL.md shows a multi-worker pattern (Source: `skills/tmux/SKILL.md:34-38`):

```
| Session    | Purpose                     |
| ---------- | --------------------------- |
| shared     | Primary interactive session  |
| worker-2–8 | Parallel worker sessions     |
```

Monitoring loop:
```bash
for s in shared worker-2 worker-3 worker-4 worker-5 worker-6 worker-7 worker-8; do
  echo "=== $s ==="
  tmux capture-pane -t $s -p 2>/dev/null | tail -5
done
```

### 4.4 When to Use tmux Over PTY

| Scenario | Use tmux | Use PTY |
|---|---|---|
| SSH-persistent sessions | Yes | No (dies with process) |
| 1Password CLI (needs persistent TTY) | Yes | Maybe |
| Quick interactive CLI | No | Yes |
| Parallel Claude Code sessions | Possible but heavyweight | **Preferred** |
| Container/sandbox | Requires tmux installed | Built-in |
| Agent needs to monitor output | Manual `capture-pane` | Native `process log` |

### 4.5 Container Caveat

The standard OpenClaw Docker image does **not** include tmux. Custom Dockerfiles must install it explicitly. The PTY approach works out of the box.

Source: `skills/tmux/SKILL.md:5` (metadata requires `bins: ["tmux"]`); see also `docs/install/docker.md` for Docker image configuration

---

## 5. Comparison: `sessions_spawn` vs `exec` + PTY

For parallel work, OpenClaw offers two levels of abstraction:

| Aspect | `sessions_spawn` (Subagent) | `exec` + PTY (Background Process) |
|---|---|---|
| **Level** | High – spawns a full agent | Low – starts a CLI process |
| **Identity** | Own agent ID, inherits skill snapshot | Just a process, no agent identity |
| **Orchestration** | `maxConcurrent`, `maxSpawnDepth`, `allowAgents` (`docs/tools/subagents.md:93-109`) | No built-in limits |
| **Communication** | `sessions_send` / `sessions_list` | `process send-keys` / `process log` |
| **Wake-up** | Event-driven (same heartbeat system) | Event-driven (`notifyOnExit`) |
| **Use Case** | Agent-level delegation ("fix this issue") | Tool-level delegation ("run this command") |
| **Token Cost** | Full agent turns for subagent | Only parent agent tokens + CLI cost |

**Rule of thumb:** Use `sessions_spawn` when the subagent needs to think (plan, reason, use tools). Use `exec` + PTY when you just need to run a CLI tool and get the result.

Source for comparison: `docs/tools/subagents.md` (sessions_spawn tool and config), `docs/tools/exec.md` (exec tool parameters and background behavior)

---

## References

### Source Files

| File | Topics Covered |
|---|---|
| `src/agents/bash-tools.exec.ts` | `exec` tool definition, parameters, background yield logic |
| `src/agents/bash-tools.exec-runtime.ts` | Process lifecycle, `maybeNotifyOnExit`, system event + heartbeat integration |
| `src/agents/bash-tools.process.ts` | `process` tool actions (list/poll/log/write/send-keys/submit/paste/kill/clear/remove) |
| `src/agents/pty-keys.ts` | PTY key encoding (named keys, modifiers, function keys, hex bytes, bracketed paste) |
| `src/infra/heartbeat-wake.ts` | `requestHeartbeatNow` – wakes agent on background process exit |
| `src/infra/system-events.ts` | `enqueueSystemEvent` – injects exit notifications into agent prompt |
| `skills/tmux/SKILL.md` | tmux skill instructions, session patterns, common commands |
| `skills/tmux/scripts/wait-for-text.sh` | Synchronization: poll tmux pane for regex match |
| `skills/tmux/scripts/find-sessions.sh` | Multi-socket tmux session discovery |

### Official Documentation (docs/)

| Document | Relevance |
|---|---|
| `docs/tools/exec.md` | Canonical reference for `exec` parameters, config, PTY, background, approvals, send-keys examples |
| `docs/tools/exec-approvals.md` | Exec approval flow, allowlist, safe bins |
| `docs/tools/subagents.md` | `sessions_spawn` tool, `maxConcurrent`, `maxSpawnDepth`, `allowAgents`, nested sub-agents |
| `docs/install/docker.md` | Docker image setup (tmux not included by default) |
