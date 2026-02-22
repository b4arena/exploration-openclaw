# Coding Agents Integration

How to integrate external coding agents (Claude Code, Codex CLI, OpenCode, Aider, etc.) into OpenClaw as orchestrated sub-tools.

---

## 1. Which Coding Agents Can Be Integrated

OpenClaw ships a bundled `coding-agent` skill that already supports four CLI agents. Any additional CLI-based coding tool can be integrated using the same pattern.

| Agent | Binary | Gated By | Notes |
|---|---|---|---|
| **OpenAI Codex CLI** | `codex` | `anyBins` | Default model `gpt-5.2-codex`; requires a git repo to run |
| **Claude Code** | `claude` | `anyBins` | Anthropic's terminal coding agent |
| **OpenCode** | `opencode` | `anyBins` | Open-source alternative |
| **Pi Coding Agent** | `pi` | `anyBins` | `@mariozechner/pi-coding-agent`; supports multiple providers |
| **Gemini CLI** | `gemini` | `bins` | Separate `gemini` skill (one-shot mode only) |
| **Aider** | `aider` | Not bundled | Can be wrapped with the same PTY pattern |
| **Cursor** | N/A | Not bundled | IDE-based, not a headless CLI; not suitable for PTY wrapping |

Source: `skills/coding-agent/SKILL.md:6` (gating: `anyBins: ["claude", "codex", "opencode", "pi"]`), `skills/gemini/SKILL.md:1-23`

The `coding-agent` skill triggers when at least one of the four binaries is on PATH. It is referenced by other skills like `github` for code review delegation (`skills/github/SKILL.md:53-54`).

---

## 2. How the exec/PTY System Enables Running Interactive CLI Tools

### 2.1 The Core Problem

Coding agents are interactive terminal applications that need a pseudo-terminal (PTY) to render output correctly. Without PTY, they produce broken output or hang entirely.

### 2.2 PTY Architecture

OpenClaw's exec tool supports a `pty` parameter that allocates a pseudo-terminal via `@lydell/node-pty`:

```
exec({ command: "claude 'Fix bug'", pty: true, background: true })
  |
  v
bash-tools.exec-runtime.ts (line 407-414)
  → detects pty:true → creates SpawnPtyInput
  |
  v
process/supervisor/supervisor.ts
  → delegates to PTY adapter
  |
  v
process/supervisor/adapters/pty.ts (line 54)
  → import("@lydell/node-pty")
  → spawn(shell, args, { cols: 120, rows: 30, cwd, env })
  → returns unified stdin/stdout stream
```

Source: `src/agents/bash-tools.exec-runtime.ts:116-119` (PTY schema), `:407-414` (PTY spawn spec), `src/process/supervisor/adapters/pty.ts:45-66` (PTY adapter creation), `src/process/supervisor/types.ts:48` (`SpawnMode = "child" | "pty"`)

### 2.3 PTY Fallback

If PTY spawn fails (e.g., `node-pty` not available), OpenClaw retries without PTY automatically:

```
exec: PTY spawn failed (...); retrying without PTY for "..."
```

Source: `src/agents/bash-tools.exec-runtime.ts:469-472`

### 2.4 Background + Event-Driven Wake-Up

When `background: true` is set, the process runs asynchronously. On exit, the system:

1. Calls `maybeNotifyOnExit()` which enqueues a system event (`src/agents/bash-tools.exec-runtime.ts:228-251`)
2. Triggers `requestHeartbeatNow()` to wake the agent immediately (`src/infra/heartbeat-wake.ts`)
3. The agent sees exit code + output excerpt in its next prompt turn

This means **zero token cost** while waiting for a coding agent to finish.

### 2.5 The `process` Tool for Session Control

Once a background PTY session is running, the `process` tool provides full interactive control:

| Action | Purpose | Source |
|---|---|---|
| `list` | Enumerate sessions | `src/agents/bash-tools.process.ts:181` |
| `poll` | Check status + output | `src/agents/bash-tools.process.ts:282` |
| `log` | Paginated output retrieval | `src/agents/bash-tools.process.ts:376` |
| `write` | Raw stdin data | `src/agents/bash-tools.process.ts:446` |
| `send-keys` | TTY key sequences (tmux-style) | `src/agents/bash-tools.process.ts:472` |
| `submit` | Data + carriage return | `src/agents/bash-tools.process.ts:511` |
| `paste` | Bracketed paste for multi-line | `src/agents/bash-tools.process.ts:532` |
| `kill` | Terminate session | `src/agents/bash-tools.process.ts:565` |

Key encoding supports named keys (`Enter`, `Escape`, `Tab`), modifiers (`C-c`, `C-d`, `M-x`), function keys, and raw hex bytes. Source: `src/agents/pty-keys.ts:17-80`

### 2.6 DSR (Device Status Request) Handling

PTY sessions generate terminal escape sequences like cursor position queries. OpenClaw strips DSR requests from output and auto-responds with cursor position data to prevent hangs:

Source: `src/agents/bash-tools.exec-runtime.ts:428-439`, `src/agents/pty-dsr.ts` (imported at line 28)

---

## 3. Existing Skills That Wrap Coding Agents

### 3.1 `coding-agent` (Bundled)

The primary skill for coding delegation. Location: `skills/coding-agent/SKILL.md`

Key features:
- Supports Codex, Claude Code, OpenCode, Pi
- Requires `pty:true` for all invocations
- Documents foreground (one-shot) and background (long-running) patterns
- Includes patterns for PR review, batch operations, and parallel issue fixing
- Provides rules: always use PTY, respect tool choice, monitor with `process:log`, never start agents in the OpenClaw workspace directory

### 3.2 `gemini` (Bundled)

Uses Gemini CLI in **one-shot mode** (no PTY needed for simple Q&A). Location: `skills/gemini/SKILL.md`

### 3.3 `tmux` (Bundled)

Legacy pattern for persistent terminal sessions. Includes Claude Code-specific patterns for checking input prompts and approving actions. Location: `skills/tmux/SKILL.md:106-146`

### 3.4 `github` (Bundled)

References `coding-agent` for code review tasks, directing users away from `gh` CLI for actual code analysis. Source: `skills/github/SKILL.md:53-54`

---

## 4. How to Create a Custom Skill That Wraps an External Coding Agent

### 4.1 Skill Directory Structure

```
skills/my-coding-agent/
├── SKILL.md          ← Required: frontmatter + instructions
└── scripts/          ← Optional: helper scripts
```

Source: `docs/tools/creating-skills.md`, `skills/skill-creator/SKILL.md:46-61`

### 4.2 Minimal SKILL.md Template (Example: Aider)

```markdown
---
name: aider-agent
description: "Delegate coding tasks to Aider via background PTY process. Use when: (1) pair-programming style edits, (2) multi-file refactoring with git integration, (3) code review with diff awareness. Requires a bash tool that supports pty:true."
metadata:
  {"openclaw": {"emoji": "🤖", "requires": {"bins": ["aider"]}}}
---

# Aider Agent

## PTY Mode Required

Aider is an interactive terminal application. Always use `pty:true`.

## Quick Start

```bash
bash pty:true workdir:~/project command:"aider --yes-always 'Your task'"
```

## Background Mode

```bash
bash pty:true workdir:~/project background:true command:"aider --yes-always 'Refactor auth module'"
```

## Monitor

```bash
process action:log sessionId:XXX
process action:poll sessionId:XXX
```
```

### 4.3 Gating Requirements

Skills are filtered at load time. Use `metadata.openclaw.requires` to gate on binary availability:

- `bins`: All listed binaries must exist on PATH
- `anyBins`: At least one must exist
- `env`: Environment variables that must be set
- `config`: Config paths that must be truthy

Source: `docs/tools/skills.md:105-136`

### 4.4 Installation and Placement

Skills can live in three locations (precedence highest to lowest):

1. `<workspace>/skills/` (per-agent)
2. `~/.openclaw/skills/` (shared across agents)
3. Bundled skills (shipped with install)

Source: `docs/tools/skills.md:15-24`

After placing the skill, ask the agent to "refresh skills" or restart the gateway. The skills watcher can auto-detect changes when enabled (`skills.load.watch: true`). Source: `docs/tools/skills.md:253-265`

---

## 5. Architecture: Agent-Within-Agent Patterns

### 5.1 Two-Level Architecture

OpenClaw offers two ways to run coding agents, which can be combined:

| Level | Mechanism | Identity | Use Case |
|---|---|---|---|
| Process-level | `exec` + PTY + background | Just a process | Run a CLI tool, get result |
| Agent-level | `sessions_spawn` (sub-agents) | Full agent with skills | Orchestrate, plan, reason |

Source: `docs/tools/subagents.md` (sessions_spawn), `docs/tools/exec.md` (exec tool)

### 5.2 Recommended Pattern: Main + Code Manager

Based on community practice, the recommended architecture is:

```
Main Agent (depth 0)
  └─> Sub-Agent: code-manager (depth 1)
       ├─> PTY session: claude 'task A' (worktree /tmp/task-a)
       ├─> PTY session: codex 'task B' (worktree /tmp/task-b)
       └─> PTY session: aider 'task C' (worktree /tmp/task-c)
```

The code-manager sub-agent:
- Creates/tracks git worktrees for isolation
- Starts/stops background PTY sessions
- Monitors progress via `process log` (not continuous streaming)
- Handles integration sequencing (merge one at a time, test between merges)

Configuration for sub-agent spawning:

```json5
{
  agents: {
    defaults: {
      subagents: {
        maxSpawnDepth: 2,        // allow orchestrator pattern
        maxChildrenPerAgent: 5,  // max parallel workers
        maxConcurrent: 8,        // global concurrency cap
      }
    }
  }
}
```

Source: `docs/tools/subagents.md:93-109`

### 5.3 Direct PTY (Simpler Alternative)

For simpler setups, skip the sub-agent layer and run coding agents directly from the main agent:

```bash
bash pty:true workdir:~/project background:true command:"codex exec --full-auto 'Build feature X'"
```

The agent sleeps at zero token cost until the process exits and triggers a heartbeat wake-up.

Source: `skills/coding-agent/SKILL.md:70-93`

### 5.4 Auto-Notify on Completion

Append a sentinel to the coding agent's prompt to trigger immediate notification:

```bash
bash pty:true workdir:~/project background:true command:"codex --yolo exec 'Build REST API.

When completely finished, run: openclaw system event --text \"Done: Built REST API\" --mode now'"
```

Source: `skills/coding-agent/SKILL.md:255-274`

---

## 6. Practical Setup Guides

### 6.1 Codex CLI

**Install:** Already included with OpenAI tooling; ensure `codex` is on PATH.

**Config:** `~/.codex/config.toml` sets the default model (`gpt-5.2-codex`).

**Key flags:**
- `exec "prompt"` — one-shot, exits when done
- `--full-auto` — sandboxed, auto-approves in workspace
- `--yolo` — no sandbox, no approvals (fastest, most dangerous)

**Example:**
```bash
bash pty:true workdir:~/project background:true command:"codex exec --full-auto 'Add dark mode toggle'"
```

**Gotcha:** Codex refuses to run outside a git directory. For scratch work: `mktemp -d && cd $_ && git init && codex exec "prompt"`.

Source: `skills/coding-agent/SKILL.md:99-136`

### 6.2 Claude Code

**Install:** `npm install -g @anthropic-ai/claude-code` or via Homebrew.

**Example:**
```bash
bash pty:true workdir:~/project background:true command:"claude 'Refactor auth module'"
```

**Tip:** Claude Code supports `--print` mode for non-interactive output, but PTY mode is still recommended for full functionality.

Source: `skills/coding-agent/SKILL.md:158-166`

### 6.3 OpenCode

**Install:** See OpenCode documentation.

**Example:**
```bash
bash pty:true workdir:~/project command:"opencode run 'Your task'"
```

Source: `skills/coding-agent/SKILL.md:170-174`

### 6.4 Pi Coding Agent

**Install:** `npm install -g @mariozechner/pi-coding-agent`

**Features:** Supports multiple providers (OpenAI, Anthropic, etc.), has prompt caching.

**Example:**
```bash
bash pty:true workdir:~/project command:"pi 'Your task'"
bash pty:true command:"pi --provider openai --model gpt-4o-mini -p 'Summarize src/'"
```

Source: `skills/coding-agent/SKILL.md:178-191`

### 6.5 Gemini CLI

**Install:** `brew install gemini-cli`

**Usage:** One-shot mode only (no background PTY needed):
```bash
gemini "Answer this question..."
gemini --model <name> "Prompt..."
```

Source: `skills/gemini/SKILL.md:26-33`

### 6.6 Aider (Custom Skill Required)

**Install:** `pip install aider-chat`

**Usage:** Requires creating a custom skill (see Section 4.2). Key flags:
- `--yes-always` for non-interactive mode
- `--model` to select the backing LLM
- Works best with `pty:true` for terminal output

### 6.7 Parallel Issue Fixing (Any Agent)

```bash
# 1. Create isolated worktrees
git worktree add -b fix/issue-78 /tmp/issue-78 main
git worktree add -b fix/issue-99 /tmp/issue-99 main

# 2. Launch agents in parallel (with PTY)
bash pty:true workdir:/tmp/issue-78 background:true command:"codex --yolo 'Fix issue #78. Commit and push.'"
bash pty:true workdir:/tmp/issue-99 background:true command:"claude 'Fix issue #99. Commit when done.'"

# 3. Monitor
process action:list

# 4. Create PRs after completion
cd /tmp/issue-78 && git push -u origin fix/issue-78
gh pr create --title "fix: issue #78" --body "..."
```

Source: `skills/coding-agent/SKILL.md:195-219`

---

## 7. Limitations and Gotchas

### 7.1 PTY Is Mandatory

Without `pty:true`, coding agents produce broken output or hang. OpenClaw has a fallback that retries without PTY on failure, but the result is degraded.

Source: `skills/coding-agent/SKILL.md:14-26`, `src/agents/bash-tools.exec-runtime.ts:469-472`

### 7.2 Git Repository Requirement (Codex)

Codex CLI refuses to run outside a trusted git directory. Always ensure the working directory is a git repo or create a temporary one.

Source: `skills/coding-agent/SKILL.md:66`

### 7.3 Authentication

Each coding agent has its own auth mechanism:
- **Codex:** OpenAI API key or `codex auth login`
- **Claude Code:** Anthropic API key or OAuth
- **OpenCode:** Provider-specific
- **Gemini:** `gemini` interactive login flow (`skills/gemini/SKILL.md:42`)

Auth tokens must be available in the environment where OpenClaw runs. For sandboxed agents, secrets are injected via `skills.entries.<name>.env` or `skills.entries.<name>.apiKey` and are scoped to the agent run.

Source: `docs/tools/skills.md:73-75`, `docs/tools/skills.md:228-237`

### 7.4 Token Cost and Context Limits

- **OpenClaw agent tokens:** Zero cost while waiting for background processes (event-driven wake-up)
- **Coding agent tokens:** Each coding agent consumes its own API tokens independently
- **Context pollution:** Avoid streaming full TUI output into the orchestrating agent; use `process log` with offset/limit for targeted reads
- **Sub-agent context:** Sub-agents have their own context window; they only inject `AGENTS.md` + `TOOLS.md` (no `SOUL.md`, `IDENTITY.md`, etc.)

Source: `docs/tools/subagents.md:230`, `src/agents/bash-tools.exec-runtime.ts:228-251`

### 7.5 Sandboxing

- PTY works in the default (non-sandboxed) mode out of the box
- In Docker sandbox mode, PTY passes through `docker exec -it` but requires `node-pty` to be available
- The standard Docker image does **not** include tmux; custom images must install it
- Coding agent binaries must also be installed inside the container when sandboxing is enabled

Source: `skills/tmux/SKILL.md:5` (tmux requires bins), `docs/tools/skills.md:139-145` (sandbox bin installation), `src/agents/bash-tools.exec-runtime.ts:388-403` (Docker PTY passthrough)

### 7.6 Workspace Isolation

Never run coding agents in the OpenClaw workspace directory (`~/clawd/` or equivalent). They may read configuration files, soul docs, or modify the running instance.

Source: `skills/coding-agent/SKILL.md:234-235`

### 7.7 Conflict Prevention in Parallel Work

When multiple coding agents edit the same repository:
- Use git worktrees for isolation (one per agent session)
- Enforce single-writer rule for shared files
- Serialize merging: integrate one worker at a time, run tests between each

### 7.8 tmux Not Available in Docker

The standard OpenClaw Docker image does not include tmux. Use the native PTY approach instead, or build a custom image.

Source: `skills/tmux/SKILL.md:5`, `docs/install/docker.md`

---

## 8. Comparison: Native OpenClaw Coding vs External Coding Agent Wrapper

| Aspect | Native OpenClaw (agent writes code directly) | External Agent Wrapper (exec + PTY) |
|---|---|---|
| **Setup complexity** | None (built-in) | Install agent binary, configure auth |
| **Token cost** | OpenClaw agent tokens only | OpenClaw tokens + external agent tokens |
| **Code quality** | Depends on OpenClaw's model | Leverages specialized coding models (Codex, Claude) |
| **Autonomy** | Limited to single tool calls per turn | Full autonomous multi-step coding sessions |
| **Parallelism** | Sub-agents via `sessions_spawn` | Multiple PTY sessions, zero-cost waiting |
| **Context** | Shares OpenClaw's context window | Independent context per coding agent |
| **Orchestration** | Full tool suite (read, write, exec, browser) | Limited to what the CLI agent supports |
| **Monitoring** | Direct in conversation | Via `process log` / `process poll` |
| **Best for** | Simple edits, quick fixes | Large features, refactoring, PR reviews |

**Recommendation:** Use native OpenClaw for simple tasks (file edits, quick fixes). Use external coding agents for substantial coding work that benefits from specialized models and autonomous multi-step reasoning.

---

## References

### Source Files

| File | Topics Covered |
|---|---|
| `src/agents/bash-tools.exec.ts` | `exec` tool definition, parameters, PTY flag |
| `src/agents/bash-tools.exec-runtime.ts` | Process lifecycle, PTY spawn, DSR handling, maybeNotifyOnExit |
| `src/agents/bash-tools.process.ts` | `process` tool actions (list/poll/log/write/send-keys/submit/paste/kill) |
| `src/agents/pty-keys.ts` | PTY key encoding (named keys, modifiers, function keys, hex) |
| `src/process/supervisor/adapters/pty.ts` | PTY adapter using `@lydell/node-pty` |
| `src/process/supervisor/types.ts` | `SpawnMode`, `SpawnInput`, `ManagedRun` types |
| `src/agents/pty-dsr.ts` | Device Status Request stripping for PTY sessions |
| `src/infra/heartbeat-wake.ts` | Agent wake-up on background process exit |
| `src/infra/system-events.ts` | System event injection into agent prompt |
| `skills/coding-agent/SKILL.md` | Bundled coding agent skill (Codex, Claude, OpenCode, Pi) |
| `skills/gemini/SKILL.md` | Gemini CLI skill (one-shot mode) |
| `skills/tmux/SKILL.md` | tmux skill with Claude Code session patterns |
| `skills/github/SKILL.md` | GitHub skill referencing coding-agent for code review |
| `skills/skill-creator/SKILL.md` | Skill creation guide and best practices |

### Official Documentation

| Document | Relevance |
|---|---|
| `docs/tools/exec.md` | Exec tool parameters, PTY, background, approvals |
| `docs/tools/skills.md` | Skill format, gating, config, loading, security |
| `docs/tools/creating-skills.md` | Step-by-step skill creation guide |
| `docs/tools/subagents.md` | Sub-agent spawning, orchestration, nesting, tool policy |
| `docs/tools/skills-config.md` | Full skills configuration schema |

### External Resources

| Title | URL |
|---|---|
| OpenClaw coding-agent SKILL.md (GitHub) | https://github.com/openclaw/openclaw/blob/main/skills/coding-agent/SKILL.md |
| Best way to use Claude Code with OpenClaw (Discord) | https://www.answeroverflow.com/m/1473453929403650209 |
| OpenClaw vs Claude Code (GetAIPerks) | https://www.getaiperks.com/en/blogs/10-openclaw-vs-claude-code |
| OpenCode vs Claude Code vs Codex (ByteBridge) | https://bytebridge.medium.com/opencode-vs-claude-code-vs-openai-codex-a-comprehensive-comparison-of-ai-coding-assistants-bd5078437c01 |
| OpenClaw Skills Documentation | https://docs.openclaw.ai/tools/skills |
| Awesome OpenClaw Skills (VoltAgent) | https://github.com/VoltAgent/awesome-openclaw-skills |
