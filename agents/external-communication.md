# 17 — External Agent Communication

## Question

How can external systems (CI/CD pipelines, scripts, monitoring tools, other AI agents) communicate with and trigger OpenClaw agents?

## TL;DR

OpenClaw provides four main entry points for external systems:

| Method | Transport | Best for |
|---|---|---|
| **CLI `openclaw agent`** | Local process / WS RPC | Scripts, cron jobs, shell pipelines |
| **Webhook HTTP endpoints** | HTTP POST to Gateway | CI/CD, GitHub/GitLab events, monitoring alerts |
| **MCP bridge** | MCP over HTTP/SSE | Claude.ai, LLM tools acting as orchestrators |
| **`openclaw message send`** | Local process / WS RPC | Injecting raw messages into a channel session |

---

## 1. CLI: `openclaw agent`

The primary scripting interface. Executes a single agent turn through the same embedded runtime used for inbound chat messages.

### Basic syntax

```bash
openclaw agent --agent <id> --message "<text>"
```

### Key flags

| Flag | Description |
|---|---|
| `--agent <id>` | Target a configured agent by ID |
| `--session-id <id>` | Reuse an existing session |
| `--to <dest>` | Derive session from group/channel target |
| `--json` | Return structured JSON — use this for scripting |
| `--deliver` | Send reply into a channel (WhatsApp, Telegram, Discord, Slack, etc.) |
| `--channel <name>` | Specify delivery platform when using `--deliver` |
| `--thinking <level>` | Extended reasoning: `off|minimal|low|medium|high|xhigh` |
| `--verbose <mode>` | Output verbosity: `on|full|off` |
| `--timeout <seconds>` | Override agent timeout (default: 120) |
| `--local` | Force local embedded runtime, bypass Gateway |

### Scripting patterns

**Fire and capture output:**
```bash
result=$(openclaw agent --agent main --message "Summarize open PRs" --json)
echo "$result" | jq '.reply'
```

**Trigger agent and deliver reply to Slack:**
```bash
openclaw agent --agent ops \
  --message "Deploy failed in prod — check logs" \
  --deliver --channel slack
```

**CI/CD failure notification (GitHub Actions):**
```yaml
- name: Notify OpenClaw on failure
  if: failure()
  run: |
    openclaw agent --agent devops \
      --message "Workflow ${{ github.workflow }} failed on ${{ github.ref }}" \
      --deliver --channel telegram
```

**Fallback behavior:** If the Gateway is unreachable, the CLI falls back to local embedded execution automatically.

Source: [`docs/openclaw.ai/tools/agent-send`](https://docs.openclaw.ai/tools/agent-send); community discussion at [answeroverflow.com](https://www.answeroverflow.com/m/1472695028210995424?focus=1472695855227080775); full CLI reference at [deepwiki.com/openclaw/openclaw/12.2-agent-commands](https://deepwiki.com/openclaw/openclaw/12.2-agent-commands).

---

## 2. CLI: `openclaw message send`

Lower-level than `openclaw agent`. Injects a raw message directly into a channel session **without** running the agent turn. Useful for populating session context or sending pre-computed content.

```bash
openclaw message send --target <target> --message "<text>"
```

This bypasses the LLM entirely. Use `openclaw agent` when you want the agent to process and respond; use `openclaw message send` only to inject content.

Source: [deepwiki.com/openclaw/openclaw/12.2-agent-commands](https://deepwiki.com/openclaw/openclaw/12.2-agent-commands).

---

## 3. Webhook HTTP Endpoints

The Gateway exposes an HTTP webhook server (default on the same port as the WebSocket control plane, `127.0.0.1:18789`). Enable it in `openclaw.json`:

```json5
{
  hooks: {
    enabled: true,
    token: "your-shared-secret",
    path: "/hooks",                  // default
    defaultSessionKey: "hooks-main", // optional fixed session
    allowRequestSessionKey: false,   // keep false unless you need caller-driven sessions
    allowedAgentIds: ["devops"],     // restrict which agents hooks can target
  }
}
```

### Authentication

All requests must include the token via header (query-string tokens are rejected):

```
Authorization: Bearer <token>
# or
x-openclaw-token: <token>
```

### Endpoints

**`POST /hooks/wake`** — Enqueue a system event (heartbeat trigger):
```json
{ "text": "wake up", "mode": "now" }
```

**`POST /hooks/agent`** — Run an isolated agent turn:
```json
{
  "message": "Deploy approved for v2.3.1",
  "name": "GitHub",
  "agentId": "devops",
  "wakeMode": "now",
  "deliver": true,
  "channel": "slack",
  "model": "anthropic/claude-3-5-sonnet"
}
```

**`POST /hooks/<name>`** — Custom named mappings resolved via `hooks.mappings`. Supports built-in presets (e.g. `gmail` for Pub/Sub).

### GitHub / GitLab integration example

```bash
# From a GitHub Actions step or GitLab CI job:
curl -X POST http://127.0.0.1:18789/hooks/agent \
  -H 'x-openclaw-token: SECRET' \
  -H 'Content-Type: application/json' \
  -d '{
    "message": "PR #42 merged into main by alice",
    "name": "GitHub",
    "agentId": "devops",
    "deliver": true,
    "channel": "slack"
  }'
```

**Important:** The webhook port is on loopback by default. To receive events from external services (GitHub, GitLab), you need a reverse proxy (nginx, Caddy) or tunneling tool (ngrok, Tailscale Funnel) in front of it.

Source: [docs.openclaw.ai/automation/webhook](https://docs.openclaw.ai/automation/webhook); security advisories at GitLab Advisory Database confirm webhook secret validation is required.

---

## 4. MCP Bridge (Community Projects)

Two independent community projects expose OpenClaw as an MCP server, allowing Claude.ai, Claude Desktop, or any MCP-aware LLM tool to delegate tasks to OpenClaw.

### 4a. `freema/openclaw-mcp`

[github.com/freema/openclaw-mcp](https://github.com/freema/openclaw-mcp)

Architecture: your server runs both the OpenClaw Gateway (port 18789) and an MCP Bridge Server (port 3000). Claude.ai connects to the bridge via HTTPS with OAuth 2.1.

**Exposed tools:**

| Tool | Mode | Description |
|---|---|---|
| `openclaw_chat` | Sync | Send message, get immediate response |
| `openclaw_status` | Sync | Check Gateway health |
| `openclaw_chat_async` | Async | Queue message, get task ID |
| `openclaw_task_status` | Async | Poll task progress |
| `openclaw_task_list` | Async | View all queued tasks |
| `openclaw_task_cancel` | Async | Cancel a pending task |

**Setup (Docker):**
```bash
docker run -e OPENCLAW_URL=http://localhost:18789 \
           -e OPENCLAW_TOKEN=your-token \
           -e OAUTH_CLIENT_ID=... \
           -p 3000:3000 \
           freema/openclaw-mcp
```

**Setup (npx for Claude Desktop):**
```bash
npx openclaw-mcp
```

Source: [github.com/freema/openclaw-mcp](https://github.com/freema/openclaw-mcp); [playbooks.com/mcp/freema/openclaw-mcp](https://playbooks.com/mcp/freema/openclaw-mcp).

### 4b. `lunarpulse/openclaw-mcp-plugin`

[github.com/lunarpulse/openclaw-mcp-plugin](https://github.com/lunarpulse/openclaw-mcp-plugin)

A plugin (not a standalone bridge) that makes OpenClaw itself an MCP *client* — it connects to external MCP servers and exposes their tools to the OpenClaw agent runtime.

- Implements MCP Streamable HTTP transport with SSE
- Dynamically discovers tools from multiple MCP servers
- Registers a unified `mcp` tool inside OpenClaw (list + call actions)

This is the inverse direction from `freema/openclaw-mcp`: instead of external systems calling into OpenClaw, this lets the OpenClaw agent call out to MCP servers.

Source: [github.com/lunarpulse/openclaw-mcp-plugin](https://github.com/lunarpulse/openclaw-mcp-plugin).

### 4c. Native MCP client support (official — in progress)

Issue [#4834](https://github.com/openclaw/openclaw/issues/4834) (originally closed as "not planned") and [#8188](https://github.com/openclaw/openclaw/issues/8188) + [#13248](https://github.com/openclaw/openclaw/issues/13248) track community demand. A community PR (#21530 by `amor71`) proposed adding `mcp.servers` to agent config and bridging MCP tool schemas into OpenClaw's `AgentTool` interface. Status as of February 2026: community-driven, not officially merged.

There is also an official `mcp-hub` skill listed at [playbooks.com/skills/openclaw/skills/mcp-hub](https://playbooks.com/skills/openclaw/skills/mcp-hub) that appears to be a curated wrapper.

Source: [github.com/openclaw/openclaw/issues/4834](https://github.com/openclaw/openclaw/issues/4834); [safeclaw.io/blog/openclaw-mcp](https://safeclaw.io/blog/openclaw-mcp).

---

## 5. npm Package

OpenClaw is distributed as an npm package:

```bash
npm install -g openclaw@latest
openclaw onboard --install-daemon
```

Package: [npmjs.com/package/openclaw](https://www.npmjs.com/package/openclaw)

A sandbox wrapper is also available: [`@agentsandbox/openclaw-agentsandbox`](https://www.npmjs.com/package/@agentsandbox/openclaw-agentsandbox).

The npm package is the runtime itself, not a client SDK. There is no separate "openclaw-client" library for programmatic use — the CLI and webhooks are the scripting interfaces.

---

## 6. Community Ecosystem

### Notification and CI/CD patterns

- **GitHub skill** (`openclaw/skills/github`): manages Issues, PRs, repositories using `gh` CLI commands. [playbooks.com/skills/openclaw/skills/github](https://playbooks.com/skills/openclaw/skills/github)
- **`gitflow` skill**: monitors CI/CD pipeline status across GitHub repositories.
- **`gitlab-ci-skills`**: GitLab CI pipeline management.
- **`deploy-agent`**: multi-step full-stack deployment workflows.
- **`agentchat`**: real-time inter-agent messaging protocol for cross-instance communication.

### Alternative forks with extended integrations

- **`ComposioHQ/secure-openclaw`** ([github](https://github.com/ComposioHQ/secure-openclaw)): forks OpenClaw to add 500+ Composio app integrations (Notion, Linear, Salesforce, etc.), persistent memory, and scheduled reminders.
- **`coollabsio/openclaw`** ([github](https://github.com/coollabsio/openclaw)): fully automated Docker images with pre-configured integrations.
- **`HKUDS/nanobot`** ([github](https://github.com/HKUDS/nanobot)): ultra-lightweight OpenClaw variant, useful as a scripting target.

### Curated skill collection

[github.com/VoltAgent/awesome-openclaw-skills](https://github.com/VoltAgent/awesome-openclaw-skills) — 3,002 community skills (after filtering out spam/malicious entries). Notable categories relevant to external communication:

- Git platforms: `github`, `gitlab-api`, `gitea`, `bitbucket-automation`
- Notifications: `mailchannels` (email), `whatsapp-styling-guide`
- API aggregation: `danube-tools` (100+ APIs including Gmail, GitHub, Notion)
- Agent networking: `moltbook` (social network for agents), `agentchat`, `uid-life` (agent marketplace registry)

---

## 7. Gateway WebSocket RPC

Both `openclaw agent` and `openclaw message send` communicate with the Gateway via its WebSocket control plane at `ws://127.0.0.1:18789`. This is an internal RPC interface — there is no published REST API spec for direct HTTP calls to agent endpoints outside of the webhook system.

The Gateway protocol is documented internally as "Gateway Protocol" and "Bridge Protocol" in the official docs, but these are not a public REST API in the traditional sense. The recommended external interface is the webhook system (section 3 above) or the CLI (section 1).

---

## 8. Decision Guide

```
External system wants to trigger OpenClaw?
│
├─ From the same machine (script, cron, CI runner)?
│   └─ Use: openclaw agent --json
│
├─ From a remote system (GitHub webhook, monitoring alert)?
│   └─ Use: POST /hooks/agent with shared secret
│        └─ Expose via reverse proxy or ngrok
│
├─ From Claude.ai or another LLM that speaks MCP?
│   └─ Use: freema/openclaw-mcp bridge
│
├─ Want OpenClaw to call external MCP tools?
│   └─ Use: lunarpulse/openclaw-mcp-plugin (plugin)
│
└─ Want pre-built integrations (GitHub, GitLab, email, 500+ apps)?
    └─ Use: skills (openclaw/skills/github etc.) or ComposioHQ fork
```

---

## References

| Resource | URL |
|---|---|
| Agent Send CLI docs | [docs.openclaw.ai/tools/agent-send](https://docs.openclaw.ai/tools/agent-send) |
| Webhook docs | [docs.openclaw.ai/automation/webhook](https://docs.openclaw.ai/automation/webhook) |
| Agent CLI commands (DeepWiki) | [deepwiki.com/openclaw/openclaw/12.2-agent-commands](https://deepwiki.com/openclaw/openclaw/12.2-agent-commands) |
| Community CLI discussion | [answeroverflow.com thread](https://www.answeroverflow.com/m/1472695028210995424?focus=1472695855227080775) |
| MCP bridge (freema) | [github.com/freema/openclaw-mcp](https://github.com/freema/openclaw-mcp) |
| MCP plugin (lunarpulse) | [github.com/lunarpulse/openclaw-mcp-plugin](https://github.com/lunarpulse/openclaw-mcp-plugin) |
| Native MCP issue | [github.com/openclaw/openclaw/issues/4834](https://github.com/openclaw/openclaw/issues/4834) |
| MCP integration guide (SafeClaw) | [safeclaw.io/blog/openclaw-mcp](https://safeclaw.io/blog/openclaw-mcp) |
| npm package | [npmjs.com/package/openclaw](https://www.npmjs.com/package/openclaw) |
| Awesome skills collection | [github.com/VoltAgent/awesome-openclaw-skills](https://github.com/VoltAgent/awesome-openclaw-skills) |
| ComposioHQ fork | [github.com/ComposioHQ/secure-openclaw](https://github.com/ComposioHQ/secure-openclaw) |
| GitHub skill | [playbooks.com/skills/openclaw/skills/github](https://playbooks.com/skills/openclaw/skills/github) |
| GitHub PR automation guide | [zenvanriel.nl openclaw-github-pr-review](https://zenvanriel.nl/ai-engineer-blog/openclaw-github-pr-review-automation-guide/) |
