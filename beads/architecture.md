# 16 — Beads-Based Multi-Agent Architecture

## Status

> **Migrated to implementation:** The architecture decisions from this research have been consolidated into [`b4arena/docs/architecture.md`](../b4arena/docs/architecture.md). This document remains as the detailed research record. The authoritative architecture reference is now in b4arena.

## Question

How can multiple OpenClaw agents with distinct identities ("souls") communicate and coordinate via the [Beads protocol](https://github.com/steveyegge/beads) while minimizing token waste? How do agents get notified of new work without consuming LLM tokens for polling?

## TL;DR

Beads provides a git-backed, DAG-based issue tracker with built-in messaging (`bd mail`) and atomic task claiming (`bd claim`). OpenClaw's existing infrastructure — cron (isolated mode), heartbeat, plugins, webhooks, and the `SOUL.md`/`IDENTITY.md` identity system — maps cleanly onto Beads. The key architectural principle comes from the [Four-Tier Execution Framework](https://brenner-axiom.github.io/docs/research/2026-02-20-system-tooling-token-savings.html): **polling is a deterministic shell operation (Tier 1, zero tokens); only triage and task execution require LLM sessions (Tier 4).**

**No existing OpenClaw ↔ Beads integration exists in the community.** This would be a first.

### Related Documents

- [agent-to-agent.md](../agents/agent-to-agent.md) — existing A2A mechanisms (`sessions_spawn`, `sessions_send`, channel-based)
- [process-execution.md](../architecture/process-execution.md) — background exec with event-driven wake-up (zero token cost pattern)
- [architecture-overview.md](../architecture/architecture-overview.md) — agent identity system (SOUL.md, IDENTITY.md, USER.md)
- [external-communication.md](../agents/external-communication.md) — full inventory of all external agent communication methods (CLI, HTTP, WS, MCP, community tools)

---

## 1. Beads Protocol Overview

### 1.1 What Beads Is

Beads is a distributed, git-backed **graph issue tracker optimized for AI agents** (18k+ stars, v0.52+, 250+ contributors). It replaces flat markdown plans with dependency-aware task graphs for long-horizon multi-agent work.

- **Install:** `npm install -g @beads/bd` or `brew install beads`
- **GitHub:** https://github.com/steveyegge/beads

### 1.2 Data Model

**Two-layer storage:**

| Layer | Location | Purpose |
|---|---|---|
| Dolt SQL DB | `.beads/dolt/` | Fast queries, cell-level merge, full history |
| JSONL | `.beads/issues.jsonl` | Git-tracked, human-readable diffs |
| Remote | standard git | Distribution |

**Issue schema:** id, title, description, status (`open` → `in_progress` → `blocked` → `closed` → `tombstone`), priority, type, labels, dependencies, comments, audit events, timestamps, attribution.

**Hierarchy:** `bd-a3f8` (epic) → `bd-a3f8.1` (task) → `bd-a3f8.1.1` (subtask)

### 1.3 DAG & Dependency Graph

Work is a **directed acyclic graph**:

| Link Type | Blocking | Hierarchical | Direction |
|---|---|---|---|
| `blocks` | Yes | No | One-way |
| `parent_id` | No | Yes | One-way |
| `relates_to` | No | No | Bidirectional |
| `replies_to` | No | No | One-way (threads) |
| `duplicate_of` | No | No | One-way (auto-closes) |
| `superseded_by` | No | No | One-way (auto-closes) |

Children are parallel by default. Only explicit `blocks` edges create sequence. "Molecules" are epics with execution intent — they enable compound cross-session agent traversal.

### 1.4 Concurrency Handling

- **Hash-based IDs** prevent merge collisions (4–6+ char alphanumeric hashes from UUIDs)
- **Dolt cell-level merge** — only changed cells conflict, not whole rows
- **Exclusive lock** (`.beads/.exclusive-lock`) — cooperative file-based lock, checked at cycle start
- **Multi-writer:** Server mode via Unix domain socket (`.beads/bd.sock`) using `dolt sql-server`
- **Atomic claim:** `bd update <id> --claim` prevents double-claiming

### 1.5 Messaging

`bd mail` — messages are first-class issues with `type: message`:

- Threading via `replies_to` dependency chains (`bd show msg-123 --thread`)
- Ephemeral lifecycle with `--older-than N` cleanup
- Same identity resolution as task operations
- Inter-agent and human-to-agent: seamless

### 1.6 Event Hooks

`.beads/hooks/` fire after `create`, `update`, and `close` events, receiving JSON payload. This is the integration point for external notification systems.

---

## 2. The Token-Saving Architecture

### 2.1 The Core Problem

From the [Token-Saving Patterns research](https://brenner-axiom.github.io/docs/research/2026-02-20-system-tooling-token-savings.html):

> "Tokens are compute budget. Every token spent on a task that doesn't require reasoning is a token unavailable for tasks that do."

**The intern test:** "If given to an intern, would they need judgment or just follow a checklist?" Checklist tasks belong in scripts (Tier 1); judgment calls require LLM backing (Tier 2–4).

A naive implementation — LLM-backed cron polling Beads every 2 minutes — would burn **1,000–3,300 tokens per poll** (system prompt + instruction parsing + tool calls) even when nothing changed. At 720 polls/day, that's 720K–2.4M tokens/day wasted on a deterministic yes/no question.

### 2.2 Four-Tier Execution Framework Applied to Beads

| Tier | Description | Beads Operation | Tokens |
|---|---|---|---|
| **1 — System Cron** | Deterministic scripts, zero reasoning | `bd ready --json`, `bd mail --unread --json`, notification formatting via jq/templates | **0** |
| **2 — LLM Cron** | Tasks requiring interpretation | Complex comment composition, status report generation | Yes (isolated session) |
| **3 — Heartbeat** | Multiple lightweight checks sharing context | Beads check as one item in `HEARTBEAT.md` checklist | Yes (shared session) |
| **4 — Pull Heartbeat** | Agent self-serves from async work queues | Triage, prioritize, dispatch to subagents | Yes (reasoning) |

### 2.3 Recommended Architecture

```
┌─────────────────────────────────────────────────────────┐
│  Tier 1: Beads Watcher (System Cron, every 2 min)       │
│  ─────────────────────────────────────────────────────── │
│  Pure shell. Zero tokens. Zero LLM involvement.         │
│                                                         │
│  for AGENT in alice bob carol; do                       │
│    READY=$(bd ready --json --agent=$AGENT)              │
│    MAIL=$(bd mail --unread --json --agent=$AGENT)       │
│                                                         │
│    if [ -n "$READY" ] || [ -n "$MAIL" ]; then           │
│      SUMMARY=$(echo "$READY $MAIL" | jq '...')          │
│                                                         │
│      # Wake via CLI (no token/URL management needed):   │
│      openclaw system event \                            │
│        --text "New beads for $AGENT: $SUMMARY" \        │
│        --mode now                                       │
│                                                         │
│      # Or for a full isolated agent turn:               │
│      # openclaw agent -m "Process beads: $SUMMARY" \   │
│      #   --agent $AGENT --json                          │
│    fi                                                   │
│  done                                                   │
└───────────────────────────┬─────────────────────────────┘
                            │ (only when work exists)
                            ▼
┌─────────────────────────────────────────────────────────┐
│  Tier 4: Agent Wake-Up (tokens start here)              │
│  ─────────────────────────────────────────────────────── │
│  OpenClaw Agent receives system event:                  │
│  "New beads: 3 tasks ready, 1 unread message"           │
│                                                         │
│  Agent reads full bead details via bd CLI                │
│  → Triages by priority and dependencies                 │
│  → Claims tasks: `bd update <id> --claim`               │
│  → Dispatches complex work to specialist subagents      │
│  → Writes results back: `bd update`, `bd comment`       │
└───────────────────────────┬─────────────────────────────┘
                            │ (for complex tasks)
                            ▼
┌─────────────────────────────────────────────────────────┐
│  Tier 4: Smart Dispatch                                 │
│  ─────────────────────────────────────────────────────── │
│  Main agent matches tasks to specialist agents          │
│  based on SOUL.md capabilities:                         │
│                                                         │
│  "Code review"  → sessions_send to coder-agent          │
│  "Deploy staging"→ Tier 1 script (zero tokens)          │
│  "Write analysis"→ sessions_send to analyst-agent       │
│  "Simple ack"   → bd comment directly (Tier 2)          │
└─────────────────────────────────────────────────────────┘
```

### 2.4 Tier Classification for Common Beads Operations

| Operation | Tier | Tokens? | Mechanism |
|---|---|---|---|
| Poll for new beads | 1 | No | System cron + `bd ready --json` |
| Format notification summary | 1 | No | jq / shell template |
| Claim a well-defined task | 1 | No | `bd update <id> --claim` in script |
| Triage: which beads are urgent? | 4 | Yes | Agent reasoning over `bd ready --json` output |
| Execute a task (code, analysis) | 4 | Yes | Agent + subagent dispatch |
| Write a comment on a bead | 2/4 | Yes | Depends on complexity |
| Create a new bead from findings | 2 | Yes | Isolated LLM cron session |
| Route deploy task to script | 1 | No | Pattern match in watcher script |

---

## 3. Agent Identity: Bridging OpenClaw Souls and Beads Roles

### 3.1 OpenClaw Identity System

Each OpenClaw agent has a workspace with persona files (see [architecture-overview.md](../architecture/architecture-overview.md)):

| File | Purpose |
|---|---|
| `SOUL.md` | Personality, communication style, values |
| `IDENTITY.md` | Name, role, capabilities, constraints |
| `USER.md` | Context about the human they serve |
| `AGENTS.md` | Knowledge of other agents in the system |

Source: `src/agents/workspace.ts` — agent workspace structure.

### 3.2 Beads Identity System

Beads uses role-based identity derived from git:

| Priority | Source | Example |
|---|---|---|
| 1 | Command-line flag | `--agent=alice` |
| 2 | Environment variable | `BEADS_AGENT=alice` |
| 3 | `git config` | `beads.role = alice` |
| 4 | System user | `$(whoami)` |

Two roles: **Maintainer** (SSH remote → shared `.beads/`) and **Contributor** (HTTPS fork → personal `~/.beads-planning/`).

### 3.3 Bridging Strategy

Each OpenClaw agent needs a corresponding Beads identity. The mapping:

```
OpenClaw Agent "alice"
├── SOUL.md          → personality & communication style
├── IDENTITY.md      → "I am Alice, a code review specialist"
├── ~/.gitconfig     → beads.role = alice
└── AGENTS.md        → knows about bob, carol

OpenClaw Agent "bob"
├── SOUL.md          → personality & communication style
├── IDENTITY.md      → "I am Bob, a deployment engineer"
├── ~/.gitconfig     → beads.role = bob
└── AGENTS.md        → knows about alice, carol
```

**Convention:** The OpenClaw `agentId` (as configured in `agents` config section) MUST match the `beads.role` value. The Tier 1 watcher script uses this ID to query per-agent beads.

Per-agent git config can be set via:
- Per-agent workspace `.gitconfig` with `GIT_CONFIG_GLOBAL` override
- Or `--agent=<agentId>` flag on every `bd` CLI call (simpler, no git config needed)

---

## 4. Notification Flow: Zero-Token Wake-Up

For a full inventory of all OpenClaw communication methods, see [external-communication.md](../agents/external-communication.md).

### 4.1 Available Wake-Up Mechanisms

OpenClaw provides multiple ways to reach agents externally. For the Beads watcher, the relevant ones are:

| Method | Token Cost | Use Case | Auth Required |
|---|---|---|---|
| `openclaw system event --text "..." --mode now` | 0 (until agent wakes) | **Preferred for Tier 1 watcher.** Injects system event, triggers heartbeat. Uses local gateway config automatically. | Gateway token (from config) |
| `openclaw agent -m "..." --agent X --json` | Yes (full turn) | Tier 4: when the agent should immediately process work. Returns structured JSON output. | Gateway token (from config) |
| `POST /hooks/wake` | 0 (until agent wakes) | HTTP alternative when CLI is unavailable (remote triggers, CI/CD). Requires explicit token and URL management. | Hook token (header) |
| `POST /hooks/agent` | Yes (isolated turn) | HTTP alternative for full agent turns. Async, returns `runId`. | Hook token (header) |
| WS RPC `wake` / `agent` | Varies | Programmatic access from WebSocket clients. | Gateway token |

**Why CLI over HTTP:** The `openclaw` CLI reads gateway connection details from the local config (`gateway.remote.url`, `gateway.remote.token`). No manual token or URL management needed. It falls back to embedded mode if the gateway is unreachable. HTTP webhooks require explicit `Authorization: Bearer <token>` headers and URL configuration — unnecessary complexity for a local cron script.

### 4.2 Option A: System Cron + CLI (Recommended)

The Tier 1 watcher runs as a system cronjob (`crontab` or systemd timer), completely outside OpenClaw. It uses the `openclaw` CLI to wake agents only when work exists.

**Advantages:**
- Maximum decoupling — works even if OpenClaw restarts
- Zero token cost for idle periods
- No token/URL management — CLI reads from config
- Follows the Token-Saving paper recommendation: "Default to simplest tier"

**Implementation:**

```bash
#!/bin/bash
# ~/.openclaw/scripts/beads-watcher.sh
# Crontab: */2 * * * * ~/.openclaw/scripts/beads-watcher.sh

AGENTS="alice bob carol"

for AGENT in $AGENTS; do
  READY=$(bd ready --json --agent="$AGENT" 2>/dev/null)
  MAIL=$(bd mail --unread --json --agent="$AGENT" 2>/dev/null)

  if [ -n "$READY" ] || [ -n "$MAIL" ]; then
    TASK_COUNT=$(echo "$READY" | jq -r 'length // 0')
    MAIL_COUNT=$(echo "$MAIL" | jq -r 'length // 0')
    SUMMARY="Tasks ready: $TASK_COUNT, Unread messages: $MAIL_COUNT"

    # Option 1: Wake only (zero tokens, agent processes on next heartbeat)
    openclaw system event --text "Beads for $AGENT: $SUMMARY" --mode now

    # Option 2: Full agent turn (tokens, but immediate processing)
    # openclaw agent -m "Process your beads queue: $SUMMARY" \
    #   --agent "$AGENT" --json >> /var/log/beads-watcher.log
  fi
done
```

### 4.3 Option B: OpenClaw Plugin with Background Service

A plugin registers a background service (via `services` in the plugin manifest) that polls Beads internally. The service is a plain TypeScript loop — no LLM involved.

**Advantages:**
- Integrated lifecycle (starts/stops with Gateway)
- Access to OpenClaw config (agent list, webhook tokens)
- Could use filesystem watch instead of polling

**Disadvantages:**
- Coupling to Gateway process
- Plugin infrastructure overhead for what is essentially a shell script

Source for plugin background services: `src/plugins/services.ts`

### 4.4 Option C: Beads Event Hooks → OpenClaw CLI

Beads fires hooks in `.beads/hooks/` after create/update/close events. A hook script could directly call `openclaw system event` to wake the relevant agent.

**Advantages:**
- Push-based, not poll-based — instant notification
- Zero polling overhead

**Disadvantages:**
- Only fires when a local `bd` command runs — not when a remote agent pushes to git
- Requires `git pull` to see remote changes first (back to polling for remote changes)

**Verdict:** Good as supplement to Option A, not as replacement.

### 4.5 Option D: HTTP Webhooks (Remote / CI/CD)

For cases where the watcher runs on a different machine (e.g., a CI/CD pipeline or monitoring system), use the HTTP endpoints:

```bash
# Wake only
curl -X POST https://gateway-host:18789/hooks/wake \
  -H "Authorization: Bearer $HOOK_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"text":"New beads from CI pipeline","mode":"now"}'

# Full agent turn
curl -X POST https://gateway-host:18789/hooks/agent \
  -H "Authorization: Bearer $HOOK_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"message":"Process CI beads","agentId":"ops","deliver":true,"channel":"slack","to":"#ops"}'
```

Requires `hooks.enabled: true` and `hooks.token` in OpenClaw config. The gateway must be network-reachable (reverse proxy or tunnel).

### 4.6 Recommendation

**Option A (System Cron + CLI) as primary**, with Option C (Beads hooks) as optimization for local operations, and Option D (HTTP) for remote triggers. This follows the Token-Saving paper's principle: "Default to simplest tier that solves the problem."

---

## 5. Human-in-the-Loop

### 5.1 How Humans Interact with Beads

Beads provides multiple interfaces for human participation:

**CLI:**
- `bd ready` — see unblocked tasks
- `bd create` — create new beads (assign work to agents)
- `bd comment` — add comments to existing beads
- `bd update --priority 0` — flag urgent tasks
- `bd mail` — send messages to agents

**Community Dashboards (see [Beads COMMUNITY_TOOLS.md](https://github.com/steveyegge/beads)):**

| Tool | Type | Features |
|---|---|---|
| `beads-dashboard` | Web (React) | Metrics, lead time analytics |
| `bdui` | TUI | Real-time tree+graph view |
| `beads-pm-ui` | Web (Next.js) | Gantt charts, quarterly goals |
| `beads-ui` | Web | Live updates, kanban board |
| `Beadster` | macOS native (Swift) | Desktop app |
| `Beadbox` | Desktop (Tauri) | Cross-platform app |

### 5.2 How Humans Interact via OpenClaw Channels

Humans can also interact through their preferred messaging channel (Telegram, Slack, Discord, etc.) by talking to the main orchestrator agent:

- "Create a bead for Alice to review PR #42" → agent runs `bd create`
- "What's the status of the deployment bead?" → agent runs `bd show`
- "Tell Bob to prioritize the security fix" → agent runs `bd update --priority 0`

This gives non-technical stakeholders access to the Beads system through natural language.

### 5.3 Observability

For monitoring the multi-agent Beads workflow:

1. **Beads-native:** `beads-dashboard` or `beads-pm-ui` for task flow visualization
2. **OpenClaw-native:** Langfuse Cloud via OTel (see [langfuse-audit-trail.md](../deployment/langfuse-audit-trail.md)) for token consumption and agent activity tracking
3. **Combined:** The Tier 1 watcher script can emit metrics (beads polled, wake-ups triggered, idle cycles) to a Prometheus endpoint or log file

---

## 6. Community Ecosystem: What Already Exists

### 6.1 Multi-Agent Coordination Tools

| Tool | Description | Relevance |
|---|---|---|
| `BeadHub` | Coordination server for AI agent teams (Python/TypeScript) | Closest to our use case — investigate further |
| `beads-orchestration` | Multi-agent orchestration skill (Node.js/Python) | Could be adapted as OpenClaw skill |
| `beads-compound` | Claude Code plugin, 28 agents (Bash/TypeScript) | Reference for multi-agent patterns |
| `beads-sdk` | TypeScript SDK, zero dependencies | Useful for plugin implementation |

### 6.2 Integration Points

| Integration | Status | Notes |
|---|---|---|
| Claude Code | Official (`bd setup claude`) | Hooks + MCP server |
| OpenCode | Community (`opencode-beads`) | Slash commands |
| VS Code | Community (3 extensions) | Editor integration |
| Neovim | Community (`nvim-beads`) | Editor integration |
| JetBrains | Community (`beads-manager`) | Editor integration |
| **OpenClaw** | **Does not exist** | **This document proposes one** |

### 6.3 MCP Server

Beads provides an MCP server installable via its plugin marketplace. This exposes:
- Tools: create, update, list, ready, mail, comment
- Resources: issue details, graphs
- Slash commands: `/beads:ready`, `/beads:create`

An OpenClaw plugin could potentially use the MCP server interface rather than shelling out to `bd` CLI.

---

## 7. Proposed Implementation Phases

### Phase 1: Minimal Viable Setup (Shell Scripts Only)

- Install Beads in the shared workspace
- Write `beads-watcher.sh` (Tier 1 system cron)
- Configure 2 OpenClaw agents with distinct SOUL.md and matching `beads.role`
- Manual `bd create` to assign work, verify wake-up and execution
- **No plugin code needed**

### Phase 2: OpenClaw Beads Plugin

- Background service for integrated lifecycle
- Agent tools: `beads_ready`, `beads_claim`, `beads_comment`, `beads_create`
- RPC methods for dashboard integration
- Optionally wrap the Beads MCP server

### Phase 3: Observability & Human UX

- Deploy a community dashboard (`beads-dashboard` or `beads-pm-ui`)
- Integrate Beads metrics into Langfuse/OTel pipeline
- Natural language Beads interaction via OpenClaw channels
- Token budget tracking per agent per bead

---

## 8. Architecture Decisions

### 8.1 Tier 1 Watcher: System Cron now, Plugin later

**Decision:** Phase 1 uses a system cronjob (`crontab`) with a shell script. Phase 2 adds an OpenClaw plugin with background service if tighter integration is needed (e.g., dynamic agent list from config, health monitoring).

**Rationale:** Follows the Token-Saving paper's principle: "Default to simplest tier." A `crontab` entry + 15 lines of Bash is sufficient for Phase 1. The plugin only becomes valuable when the agent list needs to be dynamic or when lifecycle management (auto-start/stop with Gateway) is required.

### 8.2 Identity Bridge: Convention

**Decision:** `agentId` = `beads.role`. The OpenClaw agent ID is used directly as the `--agent` flag in all `bd` CLI calls.

**Rationale:** Zero configuration. Gas Town and beads-village use the same convention. The only constraint: Beads agent names must follow OpenClaw's `agentId` naming rules (lowercase alphanumeric + hyphens).

### 8.3 Human-in-the-Loop: Both Dashboard and Channel

**Decision:** Use a Beads community dashboard (e.g., `bdui` TUI or `beads-dashboard` Web) for overview and triage. Use OpenClaw channels (Telegram/Slack/Discord) for quick natural-language interactions.

**Rationale:** No extra effort — the dashboard is a standalone community tool (deploy once), and channel interaction comes "for free" once agents have `bd` CLI access. Non-technical stakeholders use the channel; technical users use the dashboard.

### 8.4 Beads Repo Topology

**Decision:** Option B — **Standalone Beads Repo** with `BEADS_DIR` via direnv.

**Rationale:** The workspace is a software smithy with roles (PM, Architect, Eng. Manager) that have no code repo checkout. Per-repo `.beads/` with hydration requires cwd inside a repo — doesn't work for these roles. A standalone beads repo provides a single polling point for the Tier 1 watcher and zero scaling cost when projects are added. Beads is internal coordination ("Slack"), not project tracking — it doesn't need to travel with code.

See [multi-repo-topology.md](multi-repo-topology.md) §8 for the full decision record.

### 8.7 Beads vs. GitHub Issues: Separate Worlds

**Decision:** No sync. Beads = internal agent coordination (ephemeral). GitHub Issues = official project tracking (permanent). Manual references where needed.

### 8.5 Agent Interface: CLI `bd`

**Decision:** Agents interact with Beads via the `bd` CLI (`bd ready`, `bd claim`, `bd comment`, etc.) through OpenClaw's `exec` tool.

**Rationale:** Simplest start, zero dependencies, all community patterns (Gas Town, Claude Protocol, opencode-beads) are built on CLI. `bd prime` injects context via SessionStart hook. The `beads-sdk` (TypeScript) becomes relevant in Phase 2 for the plugin implementation; MCP server only if OpenClaw gains native MCP client support.

### 8.6 Dispatch: Tier 1 Shell Watcher with Label Routing

**Decision:** The Tier 1 shell watcher (system cron) handles all routing. **No agent ever polls.** The watcher reads `bd ready --json`, groups by label, and wakes the appropriate agent via `openclaw system event`. Only unlabeled/ambiguous beads trigger an orchestrator agent (Tier 4, tokens only when needed).

**Rationale:** Zero tokens for the routing decision. The agent only wakes up when there is work specifically for it. This is stricter than the Gas Town "self-service" pattern (where agents call `bd ready` inside an already-running session) because in our architecture, no session exists until the watcher creates one.

**Dispatch flow:**

```
Tier 1 Watcher (crontab, zero tokens)
│
│  bd ready --json | jq 'group_by(.labels[])'
│
├─ label "fe"    → openclaw system event --text "Beads for alice: ..." --mode now
├─ label "infra" → openclaw system event --text "Beads for bob: ..." --mode now
├─ label "be"    → openclaw system event --text "Beads for carol: ..." --mode now
├─ no label      → openclaw system event --text "Unrouted beads: ..." --mode now
│                  (wakes orchestrator agent for triage — tokens only here)
└─ bd mail       → per --agent routing (Beads native)
```

---

## References

### OpenClaw Source Files

- `src/cron/service.ts` — cron scheduler engine
- `src/cron/isolated-agent.ts` — isolated cron job runner
- `src/gateway/server-http.ts` — HTTP server, webhook endpoints (`/hooks/wake`, `/hooks/agent`)
- `src/agents/workspace.ts` — agent workspace structure (SOUL.md, IDENTITY.md)
- `src/plugins/services.ts` — plugin background services
- `src/plugins/manifest.ts` — plugin manifest loading
- `src/infra/heartbeat-wake.ts` — heartbeat wake-up mechanism
- `src/infra/system-events.ts` — system event injection

### OpenClaw CLI

- `src/cli/program/register.agent.ts` — `openclaw agent` command registration
- `src/cli/system-cli.ts` — `openclaw system event` command
- `src/cli/gateway-cli/call.ts` — `openclaw gateway call` raw RPC
- `src/gateway/call.ts` — `callGateway()` internal CLI→Gateway bridge

### OpenClaw Docs

- `docs/tools/exec.md` — exec tool parameters (background, notifyOnExit)
- `docs/concepts/session-tool.md` — session management, subagent spawning

### External Resources

- [Beads GitHub](https://github.com/steveyegge/beads) — protocol source and documentation
- [Beads COMMUNITY_TOOLS.md](https://github.com/steveyegge/beads/blob/main/docs/COMMUNITY_TOOLS.md) — community dashboards and integrations
- [Beads CLAUDE_INTEGRATION.md](https://github.com/steveyegge/beads/blob/main/docs/CLAUDE_INTEGRATION.md) — Claude Code integration guide
- [Token-Saving Patterns](https://brenner-axiom.github.io/docs/research/2026-02-20-system-tooling-token-savings.html) — Four-Tier Execution Framework (B4mad Industries)
