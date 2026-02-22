# OpenClaw CLI Reference

> Command reference for the OpenClaw CLI. Covers agent management, turns, skills, cron, messaging, config, sandbox, sessions, and diagnostics.

## 1. CLI Overview

All commands follow the pattern `openclaw <command> [options]`. On a remote server you might wrap this via SSH or a `just` shortcut.

### 1.1 Command Categories

Commands fall into two tiers:

| Tier | Examples | Gateway needed | Safe to explore |
|---|---|---|---|
| **Offline** | `agents list`, `health`, `skills list`, `config get`, `status` | No | Yes |
| **Gateway (WebSocket)** | `agent`, `cron`, `sessions`, `message send`, `system event` | Yes (pairing) | Read-only: yes; Mutations: caution |

---

## 2. Agent Management

### 2.1 Agent CRUD Commands

```bash
# List agents
openclaw agents list [--json] [--bindings]

# Add a new headless agent
openclaw agents add <id> \
  --workspace <dir> \
  --model <provider/model> \
  --non-interactive \
  [--json]

# Example: add an agent
openclaw agents add researcher \
  --workspace /home/openclaw/.openclaw/workspace-researcher \
  --model anthropic/claude-sonnet-4-6 \
  --non-interactive

# Set agent identity from IDENTITY.md
openclaw agents set-identity \
  --agent researcher \
  --from-identity \
  --workspace /home/openclaw/.openclaw/workspace-researcher

# Or set identity directly
openclaw agents set-identity \
  --agent researcher \
  --name "Researcher" \
  --emoji "🔬"

# Delete an agent
openclaw agents delete   # (interactive — careful!)
```

---

## 3. Running Agent Turns

The core command for triggering agent work — run a single LLM turn for a specific agent:

```bash
# Run a single agent turn
openclaw agent \
  --agent <id> \
  --message <text> \
  [--json] \
  [--session-id <id>] \
  [--thinking <off|minimal|low|medium|high>] \
  [--timeout <seconds>] \
  [--deliver]          # send reply back to channel
  [--verbose <on|off>]

# Example: send a task to an agent
openclaw agent --agent researcher --message "Analyze the latest data and summarize findings." --json

# Example: deliver response to the agent's bound channel
openclaw agent --agent assistant --message "Send the daily report" --deliver
```

### 3.1 Key Options

| Option | Purpose |
|---|---|
| `--agent <id>` | Target specific agent |
| `--message <text>` | Prompt for this turn |
| `--session-id <id>` | Resume existing session |
| `--deliver` | Send reply to bound channel |
| `--json` | Machine-readable output |
| `--thinking <level>` | Extended thinking (`off`, `minimal`, `low`, `medium`, `high`) |
| `--timeout <seconds>` | Override 600s default |
| `--verbose on` | Persist verbose logging |

### 3.2 Reply Routing

```bash
# Agent reply goes to a different channel than the session
openclaw agent --agent assistant --message "Send status" \
  --deliver \
  --reply-channel telegram \
  --reply-to "<chat-id>"
```

---

## 4. Skills

```bash
# List all skills
openclaw skills list

# Check readiness (shows missing deps)
openclaw skills check

# Detailed info on a skill
openclaw skills info <name>
```

Skills are markdown files (SKILL.md) that extend agent capabilities. Bundled skills ship with OpenClaw; custom skills are placed in each agent's workspace (`skills/<name>/SKILL.md`).

---

## 5. Cron & Heartbeat

### 5.1 Cron (Gateway-dependent)

Cron jobs are managed via the Gateway WebSocket — requires pairing to work from CLI.

```bash
# Add a recurring job
openclaw cron add \
  --name "watcher" \
  --agent assistant \
  --every 2m \
  --message "Check for updates" \
  [--json]

# Add a scheduled job (cron syntax)
openclaw cron add \
  --name "daily-report" \
  --agent assistant \
  --cron "0 6 * * *" \
  --tz "UTC" \
  --message "Compile and send daily report" \
  --deliver

# Add a one-shot delayed job
openclaw cron add \
  --name "reminder" \
  --agent assistant \
  --at "+30m" \
  --message "Reminder text" \
  --delete-after-run

# List jobs
openclaw cron list [--json]

# Run a job immediately (debug)
openclaw cron run <id>

# Enable/disable
openclaw cron enable <id>
openclaw cron disable <id>

# Remove a job
openclaw cron rm <id>

# View run history
openclaw cron runs [--json]
```

### 5.2 Heartbeat

```bash
# Heartbeat controls
openclaw system heartbeat --help

# Trigger a heartbeat manually
openclaw system heartbeat
```

Heartbeat runs at a configurable interval per agent. Use cron for finer scheduling control.

### 5.3 System Events

```bash
# Enqueue a system event
openclaw system event --help

# Trigger agent wake-up from external systems
openclaw system event "deployment completed" --agent devops
```

---

## 6. Message & Channel Commands

### 6.1 Sending Messages

```bash
# Send via a channel (e.g. Telegram)
openclaw message send \
  --channel telegram \
  --target <chat-id> \
  --message "Text here"

# Read recent messages
openclaw message read --channel telegram --target <chat-id>
```

### 6.2 Channel Status

```bash
# Current channel state
openclaw status
# → Telegram: ON, OK, 1 account
```

---

## 7. Config Management

```bash
# Read a config value
openclaw config get <dot.path>
# Example: openclaw config get agents.defaults.model

# Set a config value
openclaw config set <dot.path> <value>

# Remove a config value
openclaw config unset <dot.path>
```

**Warning:** `config set` modifies `openclaw.json` directly. In production, prefer managing config through your deployment tooling (e.g. Ansible templates) for reproducibility.

---

## 8. Sandbox & Security

```bash
# List sandbox containers
openclaw sandbox list

# Explain effective policy
openclaw sandbox explain

# Recreate containers after config change
openclaw sandbox recreate --all
openclaw sandbox recreate --agent <id>

# Security audit
openclaw security audit [--deep]
```

---

## 9. Session & Memory

```bash
# List sessions
openclaw sessions [--json] [--active <minutes>]

# Memory search (if enabled)
openclaw memory status
openclaw memory search --query "deployment notes"
openclaw memory index --force
```

---

## 10. Diagnostics

```bash
# Full health check
openclaw health

# Detailed status (channels, sessions, security)
openclaw status [--all]

# Doctor (auto-fix common issues)
openclaw doctor

# Tail gateway logs
openclaw logs [-f]
```

---

## 11. Test-Driven Recipes

### 11.1 Smoke Tests (no side effects)

```bash
# 1. Verify gateway is running
openclaw health

# 2. List agents and confirm structure
openclaw agents list --json

# 3. Check skill readiness
openclaw skills check

# 4. Verify channel connectivity
openclaw status
```

### 11.2 Agent Creation Test (adds state)

```bash
# Create a test agent
openclaw agents add test-agent \
  --workspace /home/openclaw/.openclaw/test-agent \
  --model anthropic/claude-sonnet-4-6 \
  --non-interactive --json

# Verify it appears
openclaw agents list --json

# Set identity
openclaw agents set-identity --agent test-agent --name "Tester" --emoji "🧪"

# Clean up
openclaw agents delete  # (interactive — select test-agent)
```

### 11.3 Agent Turn Test (uses tokens, creates session)

```bash
# Send a simple message to an agent (costs tokens!)
openclaw agent --agent main --message "Reply with exactly: SMOKE_OK" --json

# Check the response contains SMOKE_OK
```

### 11.4 Cron Test (requires Gateway pairing)

```bash
# After pairing:
openclaw cron add --name "test-ping" --agent main --at "+1m" --message "Ping" --delete-after-run --json
openclaw cron list --json
# Wait 1 minute, then:
openclaw cron runs --json
```

---

## References

- [external-communication.md](../agents/external-communication.md) — `openclaw agent` CLI deep dive
- [process-execution.md](process-execution.md) — Background process execution and PTY handling
- OpenClaw CLI docs: `openclaw <command> --help`
