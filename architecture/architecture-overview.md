# OpenClaw – Architecture Exploration

## Architecture at a Glance

```
┌─────────────────────────────────────────────────────────────┐
│                     Gateway (WebSocket RPC)                  │
│   Sessions · Config Reload · Cron · Webhooks · Auth          │
├──────────┬──────────┬───────────┬───────────┬───────────────┤
│ Channels │  Agents  │  Plugins  │  Skills   │    Memory     │
├──────────┼──────────┼───────────┼───────────┼───────────────┤
│ Telegram │ Agent 1  │ Extensions│ 50+ Skill │ LanceDB /     │
│ WhatsApp │ Agent 2  │ (40+)     │ Markdown  │ File-based    │
│ Slack    │ ...      │ Hooks     │ Files     │ Vector Search │
│ Discord  │          │ Tools     │           │               │
│ Signal   │          │ Providers │           │               │
│ Matrix   │          │           │           │               │
│ IRC ...  │          │           │           │               │
└──────────┴──────────┴───────────┴───────────┴───────────────┘
                              │
                    ┌─────────┴─────────┐
                    │   Pi Agent Core   │
                    │ (LLM Call Loop,   │
                    │  Tool Execution,  │
                    │  Streaming)       │
                    └───────────────────┘
```

OpenClaw is a **modular, plugin-driven personal AI assistant** with a local gateway as control center. The design resembles a message broker: a central server orchestrates messages between any number of messaging channels and AI agents.

> **See also**: `docs/concepts/architecture.md` (gateway architecture, components, client flows)

---

## 1. Main Directories

| Directory | Purpose |
|---|---|
| `src/gateway/` | WebSocket server, session management, config reload, chat orchestration |
| `src/agents/` | Agent runtime, **identity/persona management**, tool definitions |
| `src/config/` | Configuration loading, validation (Zod schemas), types |
| `src/channels/` | Built-in messaging channels (WhatsApp, Telegram, Slack, Discord, Signal) |
| `src/plugins/` | Plugin loader, discovery, registry, manifest handling |
| `src/memory/` | Memory search, embeddings, vector store |
| `src/cli/` | CLI commands (`gateway`, `agent`, `message send`, `onboard`, `doctor`) |
| `src/routing/` | Session key parsing, channel/peer resolution |
| `extensions/` | **40+ optional plugins** (channels, memory backends, auth providers) |
| `skills/` | **50+ Markdown-based skills** (GitHub, Coding, Apple Notes, Spotify...) |
| `apps/` | Native apps (macOS Menu Bar, iOS, Android) |

---

## 2. Identity – How OpenClaw Stores Its Persona

### 2.1 The Files in the Workspace

> **Source**: `src/agents/workspace.ts:23-30` (filename constants)
> **See also**: `docs/concepts/agent-workspace.md`, `docs/concepts/agent.md`

Each agent workspace (`~/.openclaw/workspace/`) has Markdown files:

| File | Content |
|---|---|
| **`AGENTS.md`** | Operating instructions and memory (`src/agents/workspace.ts:23`) |
| **`IDENTITY.md`** | Name, emoji, creature type, vibe, theme, avatar (`src/agents/workspace.ts:26`) |
| **`SOUL.md`** | Personality/character of the agent (`src/agents/workspace.ts:24`) |
| **`USER.md`** | Context about the user (`src/agents/workspace.ts:27`) |
| **`TOOLS.md`** | Available tools (`src/agents/workspace.ts:25`) |
| **`HEARTBEAT.md`** | Periodic health checks (`src/agents/workspace.ts:28`) |
| **`BOOTSTRAP.md`** | Startup context (`src/agents/workspace.ts:29`) |
| **`MEMORY.md`** | Long-term curated memory (`src/agents/workspace.ts:30`) |

### 2.2 IDENTITY.md Format

```markdown
- Name: Luna
- Emoji: 🌙
- Creature: AI companion
- Vibe: calm and curious
- Theme: midnight
- Avatar: ./avatar.png
```

### 2.3 Resolution Hierarchy

The identity is resolved via `src/agents/identity.ts` – from specific to general:

1. **Agent-specific config** in `config.json` → `agents[].identity`
2. **Workspace `IDENTITY.md`** file (Markdown parser in `src/agents/identity-file.ts`)
3. **Fallback defaults** (built-in default values)

Key functions:
- `resolveAgentIdentity(cfg, agentId)` → Full identity (`src/agents/identity.ts:6`)
- `resolveIdentityName(cfg, agentId)` → Name only (`src/agents/identity.ts:60`)
- `resolveAgentAvatar(cfg, agentId)` → Avatar (file, URL, or data URI) (`src/agents/identity-avatar.ts:85`)
- `resolveAckReaction(cfg, agentId)` → Emoji for message acknowledgement (`src/agents/identity.ts:13`)

### 2.4 Device Identity

Separately, there is a **device identity** in `src/infra/device-identity.ts`:
- Persistent device ID in `~/.openclaw/.device-id`
- Used for device pairing and authentication

---

## 3. Plugin/Extension System

Extensions register themselves via a **plugin API**:

```typescript
export default {
  id: "my-plugin",
  register(api: OpenClawPluginApi) {
    api.registerChannel(channelPlugin)
    api.registerTool(toolFactory)
    api.registerHook("message-before-send", handler)
    api.registerProvider(providerPlugin)
    api.registerMemoryBackend(memoryImpl)
    api.registerSkill(skillCode)
  }
}
```

Each plugin has an `openclaw.plugin.json` manifest (`src/plugins/manifest.ts:7`). Discovery happens via `discoverOpenClawPlugins()` (`src/plugins/discovery.ts:301`):
1. `node_modules/@openclaw/*`
2. Workspace `extensions/`
3. Configured paths in `config.plugins.extensionDirs[]`

> **See also**: `docs/tools/plugin.md` (plugin quick start, install, config), `docs/plugins/manifest.md` (manifest schema)

### 3.1 Hook System

> **Source**: `src/hooks/internal-hooks.ts:12` (event types), `src/hooks/hooks.ts` (public API)

The internal hook system uses event types: `command`, `session`, `agent`, `gateway`, `message` (`src/hooks/internal-hooks.ts:12`). These are combined with actions (e.g., `gateway:startup`, `message:received`, `session:start`).

Extensible lifecycle hooks (via plugins):
- **Model hooks**: `model-override`, `model-discovery`
- **Tool hooks**: `before-tool-call`, `after-tool-call`, `tool-result-guard`
- **Message hooks**: `message-before-send`, `message-after-send`
- **Session hooks**: `session-before-create`, `session-after-close`
- **Gateway hooks**: `gateway-startup`, `gateway-ready`, `config-reload`

---

## 4. Gateway & Multi-Agent

The gateway server is the heart of the system – a **WebSocket RPC server** with methods like:

```
agent.*      → Agent operations
chat.*       → Chat send/stream
channels.*   → Channel status
config.*     → Config get/patch/reload
sessions.*   → Session management
auth.*       → Authentication & pairing
```

**Multi-Agent**: You can configure multiple isolated agents, each with its own workspace, its own identity, and its own skills.

> **See also**: `docs/concepts/multi-agent.md` (multi-agent routing, bindings, isolation)

### 4.1 Gateway Startup Sequence (`src/gateway/boot.ts`)

> **Source**: `src/gateway/boot.ts:138` (`runBootOnce()`)

1. Load and validate configuration
2. Discover plugins and channels
3. Migrate legacy config
4. Initialize auth profiles
5. Build model catalog
6. Start channel listeners
7. Register webhook handlers
8. Start cron jobs
9. Open WebSocket server

---

## 5. Security Model

> **See also**: `docs/channels/pairing.md` (DM pairing), `docs/gateway/sandboxing.md` (sandbox config), `docs/security/` (threat model)

- **DM pairing** (default): Unknown senders must first be approved
- **Tool policies**: Allow/deny lists per agent
- **Sandbox**: OS-level sandboxing via Podman/Docker

---

## 6. Configuration

> **See also**: `docs/gateway/configuration.md` (config overview), `docs/gateway/configuration-reference.md` (full field reference)

Everything flows into a central **`openclaw.json`** (JSON5 format, located at `~/.openclaw/openclaw.json`):

```json
{
  "agents": {
    "list": [{
      "id": "main",
      "default": true,
      "workspace": "~/.openclaw/workspace",
      "model": "claude-3-5-sonnet",
      "identity": { "name": "Luna", "emoji": "🌙" },
      "skills": ["github", "coding-agent"],
      "memorySearch": { "enabled": true }
    }]
  },
  "channels": { /* Telegram, WhatsApp, etc. */ },
  "plugins": { /* Extension configuration */ }
}
```

### 6.1 Workspace Directory Structure

> **Source**: `src/agents/workspace.ts:23-33` (filename and state constants)

```
~/.openclaw/workspace/
├── .openclaw/workspace-state.json # Onboarding state
├── AGENTS.md                      # Operating instructions + memory
├── IDENTITY.md                    # Agent identity
├── SOUL.md                        # Personality
├── USER.md                        # User context
├── TOOLS.md                       # Tool documentation
├── HEARTBEAT.md                   # Health checks
├── BOOTSTRAP.md                   # Startup context
├── MEMORY.md                      # Long-term curated memory
├── extensions/                    # Local plugins
└── skills/                        # Workspace skills
```

### 6.2 File Locations

| What | Where |
|---|---|
| Main configuration | `~/.openclaw/openclaw.json` |
| Agent identity | `~/.openclaw/workspace/IDENTITY.md` |
| Agent sessions | `~/.openclaw/agents/{agentId}/sessions/*.jsonl` |
| Device ID | `~/.openclaw/.device-id` |
| Auth profiles | `~/.openclaw/agents/{agentId}/agent/auth-profiles.json` (see `docs/concepts/model-failover.md`) |
| Memory data | `~/.openclaw/memory/` |
| Cache | `~/.openclaw/cache/` |
| Credentials | `~/.openclaw/credentials/` |

---

## 7. Agent Runtime (Pi Agent Core)

> **See also**: `docs/concepts/agent.md` (agent runtime overview)

OpenClaw embeds **Pi Agent Core** (`src/agents/pi-embedded-*.ts`):
- LLM calls with streaming (`src/agents/pi-embedded-runner.ts`)
- Tool execution loop (`src/agents/pi-embedded-subscribe.tools.ts`)
- Session history management (`src/agents/pi-embedded-helpers.ts`)
- Compaction (summarization) for long contexts (`src/agents/pi-embedded-subscribe.handlers.compaction.ts`)

### 7.1 System Prompt Construction (`src/agents/system-prompt.ts`)

> **See also**: `docs/concepts/system-prompt.md` (full prompt structure and assembly)

1. Agent identity (name, vibe, creature)
2. User context (from `USER.md`)
3. Current time/timezone
4. Session rules
5. Tool policy
6. Workspace skills (injected as instructions)
7. SOUL.md content
8. Model-specific adjustments

---

## 8. Memory System

> **See also**: `docs/concepts/memory.md` (memory file layout and workflow)

Two implementations:
- **`memory-core`** – File-based (simple, no DB) (`src/memory/`)
- **`memory-lancedb`** – LanceDB vector store (production) (extension)

Features: Hybrid search (vector + keyword/BM25), session-aware, automatic background sync.

---

## 9. Skills System

> **See also**: `docs/tools/skills.md` (skill locations, precedence, gating), `docs/tools/creating-skills.md`

Skills are **Markdown files** with metadata:

```markdown
---
description: Brief description
tags: [tag1, tag2]
---

# Skill Title

Skill instructions in natural language...
```

Loaded from: `skills/*/index.md` (bundled), workspace `skills/`, or ClawHub (download).

---

## 10. Agent Routing in Detail

> **Source**: `src/routing/resolve-route.ts`, `src/routing/bindings.ts`, `src/routing/session-key.ts`
> **See also**: `docs/channels/channel-routing.md` (routing rules per channel), `docs/concepts/session.md` (session management, DM scoping)

### 10.1 How Is It Decided Which Agent Responds?

When a message arrives via a channel (Telegram, Discord, Web UI, ...), the channel adapter calls `resolveAgentRoute()` (`src/routing/resolve-route.ts:295`). This function receives the **message context** and returns the matching agent + session key.

**Input**:
```typescript
resolveAgentRoute({
  cfg: OpenClawConfig,
  channel: "telegram",              // Which channel?
  accountId: "default",             // Which bot account?
  peer: { kind: "direct", id: "+4917..." },  // Who is writing?
  parentPeer: null,                 // Thread parent (for Discord/Slack)
  guildId: null,                    // Discord server
  teamId: null,                     // MS Teams team
  memberRoleIds: [],                // Discord roles of the sender
})
```

**Output**:
```typescript
{
  agentId: "support",
  channel: "telegram",
  accountId: "default",
  sessionKey: "agent:support:telegram:direct:+4917...",
  mainSessionKey: "agent:support:main",
  matchedBy: "binding.peer"   // Why this agent was chosen
}
```

### 10.2 The Binding System

Routing is **not solely based on the channel**. Bindings can match on various criteria:

```
Binding match criteria (combinable):
┌─────────────────────────────────────────────────┐
│  channel    → "telegram", "discord", "slack"... │
│  accountId  → "default", "biz", "*" (wildcard)  │
│  peer       → { kind: "direct", id: "+4917..." }│  ← specific contact!
│  guildId    → Discord server ID                  │
│  teamId     → MS Teams team ID                   │
│  roles      → ["admin-role-id"]                  │  ← Discord roles!
└─────────────────────────────────────────────────┘
```

**Example config**: Different agents depending on context on the same channel:

```json
{
  "bindings": [
    {
      "agentId": "support",
      "match": {
        "channel": "telegram",
        "peer": { "kind": "group", "id": "12345" }
      }
    },
    {
      "agentId": "personal",
      "match": {
        "channel": "telegram",
        "peer": { "kind": "direct", "id": "+4917xxx" }
      }
    },
    {
      "agentId": "fallback",
      "match": { "channel": "telegram", "accountId": "*" }
    }
  ]
}
```

**Result**: Telegram group 12345 → `support`, DM from +4917xxx → `personal`, everything else on Telegram → `fallback`.

### 10.3 Priority Order (Tiers)

The matching logic checks bindings in a fixed priority order (`src/routing/resolve-route.ts:366-434`):

```
Highest priority
    │
    ▼  1. binding.peer          → Exact peer match (contact/group)
    ▼  2. binding.peer.parent   → Thread inherits from parent channel
    ▼  3. binding.guild+roles   → Discord guild + role match
    ▼  4. binding.guild         → Discord guild (without roles)
    ▼  5. binding.team          → MS Teams team
    ▼  6. binding.account       → Account level (accountId ≠ "*")
    ▼  7. binding.channel       → Channel wildcard (accountId = "*")
    │
    ▼  8. default               → Default agent (no binding matched)
Lowest priority
```

**Important**: A more specific match always wins. A peer binding beats a guild binding, which beats an account binding, and so on.

### 10.4 Pre-filtering

Before the tiers are evaluated, the binding list is **pre-filtered** by `channel` + `accountId`. Only bindings that match the channel and account enter the tier evaluation (`getEvaluatedBindingsForChannelAccount`, `src/routing/resolve-route.ts:179`). The result is cached.

### 10.5 Thread Inheritance

For threads (e.g., Discord threads, Slack threads), the system first attempts to match the thread itself. If no binding exists for the thread peer, the **parent channel** is checked (Tier 2: `binding.peer.parent`). This way, a thread automatically inherits the agent of the channel in which it was created.

### 10.6 Role-based Routing (Discord)

Discord-specific: Bindings can match on the message sender's roles. If a user with the "admin" role writes in a Discord channel, a different agent can respond compared to a user without that role.

```json
{
  "agentId": "opus-agent",
  "match": {
    "channel": "discord",
    "guildId": "999999",
    "roles": ["admin-role-id"]
  }
}
```

### 10.7 Default Agent Fallback

If no binding matches (`resolveDefaultAgentId` in `src/agents/agent-scope.ts:61`):
1. Agent with `default: true` in `agents.list` (line 66)
2. First agent in `agents.list` (line 71)
3. Hardcoded: `"main"` (constant `DEFAULT_AGENT_ID`)

### 10.8 Web UI

The Web UI selects the agent **implicitly via the session key**. When you open a session with key `agent:support:main`, you are talking to the "support" agent. There is no automatic routing – the session key is set when the session is created.

### 10.9 DM Scoping

Separate from agent routing, `dmScope` determines how DM sessions are constructed (`src/routing/session-key.ts:150`):

| Mode | Session Key | Effect |
|---|---|---|
| `"main"` (default) | `agent:X:main` | All DMs in one session |
| `"per-peer"` | `agent:X:direct:{peerId}` | One session per contact |
| `"per-channel-peer"` | `agent:X:telegram:direct:{peerId}` | Per contact per channel |
| `"per-account-channel-peer"` | `agent:X:telegram:tasks:direct:{peerId}` | Maximum isolation |

**Identity links** can merge peers across channels:
```json
{ "identityLinks": { "alice": ["telegram:111111", "discord:222222"] } }
```
→ Alice on Telegram and Discord shares one session.

---

## 11. Can a Person Talk to Multiple Agents?

**Yes, in several ways:**

### 11.1 Via Bindings (automatic, context-based)

A person can talk to different agents in different contexts:

```json
{
  "bindings": [
    {
      "agentId": "work",
      "match": { "channel": "telegram", "peer": { "kind": "group", "id": "work-group-123" } }
    },
    {
      "agentId": "personal",
      "match": { "channel": "telegram", "peer": { "kind": "direct", "id": "+4917xxx" } }
    }
  ]
}
```

→ The same person gets the "work" agent in the work group and the "personal" agent in the DM.

### 11.2 Via the Web UI (manual, session key)

In the Web UI you can create sessions with different agent keys:
- `agent:main:main` → Main agent
- `agent:support:main` → Support agent
- `agent:researcher:main` → Research agent

### 11.3 Via Different Channels

Different channels can route to different agents via bindings:
- Telegram → `personal`
- Slack → `work`
- Discord → `gaming`

### 11.4 Limitation: No Agent Switching Within a Session

Within a running session, the agent is **fixed**. The agent ID is baked into the session key (`agent:main:telegram:direct:+4917...`). You cannot say "now connect me to the other agent" in the middle of a conversation.

---

## 12. Agent-to-Agent: The Subagent System

> **Source**: `src/agents/subagent-spawn.ts`, `src/agents/subagent-registry.ts`, `src/agents/tools/sessions-spawn-tool.ts`
> **See also**: `docs/tools/subagents.md` (usage, slash commands, config)

**Yes, agents can spawn other agents!** This is not a theoretical feature – it is a fully implemented system with its own lifecycle management.

### 12.1 The `sessions_spawn` Tool

Every agent has access to the `sessions_spawn` tool, which allows it to start sub-agents:

```typescript
// What the agent calls:
sessions_spawn({
  task: "Research the latest Kubernetes CVEs",
  label: "k8s-cve-research",
  agentId: "researcher",     // Optional: different agent
  model: "anthropic/claude-opus-4-6",  // Optional: different model
  thinking: "high",          // Optional: thinking level
  runTimeoutSeconds: 300,    // Optional: timeout
  cleanup: "keep"            // "keep" or "delete" (keep session?)
})
```

### 12.2 Spawn Flow

```
Agent "main" (in session agent:main:telegram:direct:+4917...)
    │
    │  sessions_spawn({ task: "...", agentId: "researcher" })
    │
    ▼
┌─────────────────────────────────────────────────┐
│  1. Check permission:                           │
│     - allowAgents in subagents config            │
│     - maxSpawnDepth (default: 1)                 │
│     - maxChildrenPerAgent (default: 5)           │
│                                                   │
│  2. Create child session:                        │
│     "agent:researcher:subagent:{random-uuid}"    │
│                                                   │
│  3. Send task to gateway:                        │
│     callGateway({ method: "agent", ... })        │
│                                                   │
│  4. Register run in SubagentRegistry             │
│                                                   │
│  5. Child runs in the background                 │
└─────────────────────────────────────────────────┘
    │
    │  (async – child works autonomously)
    │
    ▼
Child "researcher" completes task
    │
    ▼
Announce flow: Result is sent back
to the requester chat
```

### 12.3 Security Boundaries

The config controls what is allowed (`src/config/types.agents.ts:37-41`, defaults in `src/config/types.agent-defaults.ts:241-247`):

```json
{
  "agents": {
    "list": [{
      "id": "main",
      "subagents": {
        "allowAgents": ["researcher", "coder"],  // Which agents may main spawn?
        "model": "anthropic/claude-sonnet-4-6"    // Default model for subagents
      }
    }],
    "defaults": {
      "subagents": {
        "maxConcurrent": 1,         // Max parallel subagent runs (global)
        "maxSpawnDepth": 1,         // Max nesting depth (no sub-sub-agent by default)
        "maxChildrenPerAgent": 5    // Max active children per session
      }
    }
  }
}
```

**Important**:
- `allowAgents: ["*"]` → may spawn any agent
- `maxSpawnDepth: 1` (default) → **no sub-sub-agent** (agent can spawn a subagent, but subagent cannot spawn another one)
- `maxSpawnDepth: 2` → subagents may also spawn their own subagents (chains)

### 12.4 Announce Flow (Reporting Results Back)

When the subagent finishes (`src/agents/subagent-registry.ts`):

1. **Lifecycle event** "end" is received
2. **SubagentRegistry** detects completion
3. **Announce flow** (`src/agents/subagent-announce.ts`) summarizes the result
4. Result is injected as a **user message** into the requester session
5. The requester agent sees the result and can react to it

→ This is real **agent-to-agent communication**: the main agent delegates, the sub-agent works, and the result comes back automatically.

### 12.5 Session Key Format for Subagents

```
agent:{targetAgentId}:subagent:{random-uuid}
```

Example: `agent:researcher:subagent:a1b2c3d4-...`

### 12.6 Persistence & Crash Recovery

The SubagentRegistry is persisted to disk (`src/agents/subagent-registry.store.ts`). After a gateway restart, open runs are restored and resumed. There is retry logic with backoff for result delivery (via `src/agents/subagent-announce-queue.ts`).

---

## 13. Model Provider & Auth System

> **Source**: `src/agents/model-auth.ts`, `src/agents/model-selection.ts`, `src/agents/auth-profiles/`
> **See also**: `docs/concepts/model-providers.md` (provider setup reference), `docs/concepts/oauth.md` (OAuth flows), `docs/concepts/model-failover.md` (auth rotation and fallback)

### 13.1 Model Reference: `provider/model`

Yes, in the config `"anthropic/claude-opus-4-6"` is sufficient – that is the **ModelRef**:

```
"anthropic/claude-opus-4-6"
     │              │
     ▼              ▼
  Provider       Model ID
```

If no provider is specified (e.g., just `"claude-opus-4-6"`), it falls back to the default provider (Anthropic). However, this is deprecated – always use `provider/model`.

### 13.2 Supported Providers

> Individual provider docs: `docs/providers/anthropic.md`, `docs/providers/openai.md`, `docs/providers/ollama.md`, etc.

| Provider ID | Service | Auth Methods |
|---|---|---|
| `anthropic` | Claude API | OAuth token, setup token, API key |
| `openai` | OpenAI API | API key (`OPENAI_API_KEY`) |
| `openai-codex` | OpenAI Codex OAuth | OAuth (ChatGPT web session) |
| `google` | Gemini API | API key (`GEMINI_API_KEY`) |
| `google-gemini-cli` | Gemini CLI | OAuth (extension) |
| `google-antigravity` | Google Cloud Code | OAuth (extension) |
| `xai` | Grok (xAI) | API key |
| `openrouter` | OpenRouter | API key |
| `amazon-bedrock` | AWS Bedrock | AWS SDK / IAM |
| `ollama` | Local Ollama | API key (optional) |
| `vllm` | Local vLLM | API key (optional) |
| `litellm` | LiteLLM Gateway | API key |
| `github-copilot` | GitHub Copilot | GitHub token |
| `together` | Together AI | API key |
| `huggingface` | HF Inference | HF token |
| `minimax-portal` | MiniMax | OAuth (extension) |
| `qwen-portal` | Qwen/Alibaba | OAuth (extension) |
| `moonshot` | Kimi/Moonshot | API key |
| `nvidia` | Nvidia | API key |
| `venice` | Venice AI | API key |
| `cloudflare-ai-gateway` | Cloudflare AI | API key |
| `vercel-ai-gateway` | Vercel AI | API key |
| Custom | OpenAI/Anthropic-compatible | Configurable |

### 13.3 Auth Resolution: What Happens When You Set `anthropic/claude-opus-4-6`?

The flow in `src/agents/model-auth.ts:135` (`resolveApiKeyForProvider()`):

```
Agent config: model = "anthropic/claude-opus-4-6"
                │
                ▼  parseModelRef() → { provider: "anthropic", model: "claude-opus-4-6" }
                │
                ▼  resolveApiKeyForProvider("anthropic")
                │
    ┌───────────┴───────────────────────────────────┐
    │                                               │
    │  1. Explicit profile ID?                      │
    │     → Yes: Resolve directly                   │
    │                                               │
    │  2. Walk through auth profile store:          │
    │     resolveAuthProfileOrder("anthropic")      │
    │     → Sorted by: OAuth > Token > API Key      │
    │     → Round-robin for multiple profiles       │
    │     → Cooldown handling for rate limits        │
    │     → First valid profile wins                 │
    │                                               │
    │  3. Check env variables:                      │
    │     → ANTHROPIC_OAUTH_TOKEN                   │
    │     → ANTHROPIC_API_KEY                       │
    │                                               │
    │  4. Check custom provider config:             │
    │     → models.providers.anthropic.apiKey        │
    │                                               │
    │  5. Nothing found → Error                     │
    └───────────────────────────────────────────────┘
```

**Important**: You do **not** need to manually link provider and auth. The provider name in the ModelRef (`anthropic/...`) is automatically mapped to the matching auth profile store.

### 13.4 Auth Profile Store

Stored in `~/.openclaw/agents/{agentId}/auth-profiles.json`:

```json
{
  "profiles": {
    "anthropic:default": {
      "type": "token",
      "provider": "anthropic",
      "token": "sk-ant-...",
      "email": "user@example.com",
      "expires": 1740000000000
    },
    "openai:default": {
      "type": "api_key",
      "provider": "openai",
      "key": "sk-..."
    },
    "google:default": {
      "type": "api_key",
      "provider": "google",
      "key": "AIza..."
    }
  },
  "order": {
    "anthropic": ["anthropic:default"],
    "openai": ["openai:default"]
  }
}
```

### 13.5 Profile Priority with Multiple Credentials

If you have multiple credentials for a provider (e.g., 2 Anthropic API keys + 1 OAuth token):

1. **OAuth** is preferred (score 0)
2. **Token** (score 1)
3. **API Key** (score 2)
4. Within the same type: **round-robin** by `lastUsed` (oldest first)
5. Profiles in **cooldown** (rate limit) are moved to the end

### 13.6 Special Case: OpenAI Codex OAuth

If you set `openai/gpt-5.3-codex` as the model, the provider is automatically rewritten to `openai-codex` (`src/agents/model-selection.ts:110-127`). This then uses the Codex OAuth flow instead of the API key.

### 13.7 Fallback Chains per Agent

```json
{
  "agents": {
    "list": [{
      "id": "main",
      "model": {
        "primary": "anthropic/claude-opus-4-6",
        "fallbacks": ["openai/gpt-4o", "google/gemini-2.5-pro"]
      }
    }]
  }
}
```

If Anthropic throws a rate limit, OpenClaw automatically falls back to the next fallback.

### 13.8 Env Variables per Provider

| Provider | Env Variable |
|---|---|
| Anthropic | `ANTHROPIC_OAUTH_TOKEN`, `ANTHROPIC_API_KEY` |
| OpenAI | `OPENAI_API_KEY` |
| Google | `GEMINI_API_KEY` |
| xAI | `XAI_API_KEY` |
| OpenRouter | `OPENROUTER_API_KEY` |
| GitHub Copilot | `COPILOT_GITHUB_TOKEN`, `GH_TOKEN` |
| Together | `TOGETHER_API_KEY` |
| Hugging Face | `HUGGINGFACE_HUB_TOKEN`, `HF_TOKEN` |
| Ollama | `OLLAMA_API_KEY` |

### 13.9 Setup Recommendation for the 3 Subscriptions

```bash
# Interactive setup:
openclaw onboard

# 1. Anthropic (Claude Max):
#    → "Anthropic" → "setup-token"
#    → In another terminal: claude setup-token
#    → Paste the token

# 2. OpenAI Pro:
#    → "OpenAI" → "OpenAI Codex OAuth" (uses web subscription)
#    → OR: set OPENAI_API_KEY (separate credit!)

# 3. Google Gemini:
#    → "Google" → "Gemini API key" (from AI Studio)
#    → OR: "Gemini CLI OAuth" (extension)
```

---

## Open Questions for Further Exploration

- [x] ~~How exactly does routing of incoming messages to the correct agent work?~~
- [x] ~~Can a person talk to multiple agents?~~
- [x] ~~Can agents talk to agents (agent-to-agent)?~~
- [x] ~~How does model/provider configuration and auth work?~~
- [ ] How does session management work in detail?
- [ ] How are skills injected into the system prompt?
- [ ] How does the memory system (embeddings, retrieval) work in detail?
- [ ] What does the message flow from receipt to response look like concretely?
- [ ] How does config hot-reload work?

---

## References

### Source Files

| File | Topics Covered |
|---|---|
| `src/agents/identity.ts` | Agent identity resolution (name, emoji, ack reaction) |
| `src/agents/identity-file.ts` | IDENTITY.md Markdown parser |
| `src/agents/identity-avatar.ts` | Avatar resolution (file, URL, data URI) |
| `src/agents/workspace.ts` | Workspace file constants (AGENTS.md, SOUL.md, etc.) |
| `src/agents/agent-scope.ts` | Default agent ID resolution, workspace dir resolution |
| `src/agents/system-prompt.ts` | System prompt assembly |
| `src/agents/pi-embedded-runner.ts` | Agent runtime (LLM call loop, streaming) |
| `src/agents/pi-embedded-subscribe.ts` | Event subscription for agent runs |
| `src/agents/pi-embedded-subscribe.handlers.compaction.ts` | Context compaction/summarization |
| `src/agents/pi-embedded-helpers.ts` | Session history helpers |
| `src/agents/subagent-spawn.ts` | Subagent spawn logic |
| `src/agents/subagent-registry.ts` | Subagent lifecycle management |
| `src/agents/subagent-registry.store.ts` | Subagent persistence to disk |
| `src/agents/subagent-announce.ts` | Subagent result announcement flow |
| `src/agents/subagent-announce-queue.ts` | Retry queue for result delivery |
| `src/agents/tools/sessions-spawn-tool.ts` | `sessions_spawn` tool definition |
| `src/agents/model-auth.ts` | API key/auth resolution per provider |
| `src/agents/model-selection.ts` | Model ref parsing, provider rewriting (e.g., Codex) |
| `src/config/types.agents.ts` | Agent config types (subagents, allowAgents) |
| `src/config/types.agent-defaults.ts` | Agent defaults (maxSpawnDepth, maxChildrenPerAgent) |
| `src/config/zod-schema.agent-runtime.ts` | Zod validation for agent runtime config |
| `src/config/zod-schema.agent-defaults.ts` | Zod validation for agent defaults |
| `src/routing/resolve-route.ts` | Agent routing (tier-based binding matching) |
| `src/routing/bindings.ts` | Binding type definitions |
| `src/routing/session-key.ts` | Session key construction, dmScope, identityLinks |
| `src/gateway/boot.ts` | Gateway startup sequence |
| `src/plugins/discovery.ts` | Plugin discovery (`discoverOpenClawPlugins`) |
| `src/plugins/manifest.ts` | Plugin manifest filename constant |
| `src/hooks/internal-hooks.ts` | Internal hook event system |
| `src/hooks/hooks.ts` | Public hook API (register, trigger, etc.) |
| `src/infra/device-identity.ts` | Device identity (persistent device ID) |

### Official Documentation (docs/)

| Document | Topics Covered |
|---|---|
| `docs/concepts/architecture.md` | Gateway architecture, components, client flows |
| `docs/concepts/agent.md` | Agent runtime, workspace contract, bootstrap |
| `docs/concepts/agent-workspace.md` | Workspace location, layout, backup strategy |
| `docs/concepts/multi-agent.md` | Multi-agent routing, bindings, isolation |
| `docs/concepts/session.md` | Session management, DM scoping, identity links |
| `docs/concepts/system-prompt.md` | System prompt structure and assembly |
| `docs/concepts/memory.md` | Memory file layout, automatic flush |
| `docs/concepts/model-providers.md` | Provider setup reference, API key rotation |
| `docs/concepts/model-failover.md` | Auth profile rotation, model fallback chains |
| `docs/concepts/oauth.md` | OAuth token exchange, storage, multi-account |
| `docs/channels/channel-routing.md` | Per-channel routing rules, session key shapes |
| `docs/channels/pairing.md` | DM pairing, node pairing |
| `docs/gateway/configuration.md` | Configuration overview, common patterns |
| `docs/gateway/configuration-reference.md` | Full field-by-field config reference |
| `docs/gateway/sandboxing.md` | Docker/Podman sandboxing config |
| `docs/tools/skills.md` | Skills: locations, precedence, gating |
| `docs/tools/subagents.md` | Subagent slash commands, spawn behavior |
| `docs/tools/plugin.md` | Plugin discovery, install, config |
| `docs/plugins/manifest.md` | Plugin manifest schema (`openclaw.plugin.json`) |
| `docs/providers/*.md` | Individual provider setup docs |
| `docs/security/` | Threat model, security contributing guide |
