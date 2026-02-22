# OpenClaw Ecosystem – Extensions, Skills & Orchestration

## 1. Extensions (36 Total)

Extensions are deep infrastructure plugins: Channels, Auth Providers, Memory Backends, Hooks.

Source: Each extension has a manifest at `extensions/<name>/openclaw.plugin.json` (36 manifests total).
See also: `docs/tools/plugin.md`

### 1.1 Messaging Channels (18)

See also: `docs/channels/index.md` and per-channel docs under `docs/channels/<name>.md`

| Extension | Platform | Description |
|---|---|---|
| `telegram` | Telegram | Bot/DM Integration |
| `discord` | Discord | Server/DM/Thread Integration |
| `slack` | Slack | Workspace Integration |
| `whatsapp` | WhatsApp | Business API |
| `signal` | Signal | Secure Messaging |
| `imessage` | iMessage | Apple Ecosystem |
| `bluebubbles` | iMessage | Via BlueBubbles Relay App (macOS) |
| `googlechat` | Google Chat | Workspace Integration |
| `msteams` | MS Teams | Teams Channels/DMs |
| `matrix` | Matrix/Element | Federated Protocol |
| `irc` | IRC | Classic Chat |
| `mattermost` | Mattermost | Self-hosted Team Chat |
| `nextcloud-talk` | Nextcloud | Federated Chat |
| `feishu` | Feishu/Lark | ByteDance Platform |
| `line` | LINE | Japanese Platform |
| `zalo` / `zalouser` | Zalo | Vietnamese Platform |
| `tlon` | Tlon/Urbit | Decentralized Platform |
| `nostr` | Nostr | Decentralized Protocol |
| `twitch` | Twitch | Streaming Chat |

### 1.2 Voice

| Extension | Description |
|---|---|
| `voice-call` | Phone Calls via Twilio/Telnyx/Plivo |

### 1.3 Auth Providers (5)

See also: `docs/concepts/oauth.md`, `docs/concepts/model-providers.md`

| Extension | Description |
|---|---|
| `google-gemini-cli-auth` | Gemini CLI OAuth |
| `google-antigravity-auth` | Google Cloud Code Assist OAuth |
| `minimax-portal-auth` | MiniMax Portal OAuth |
| `qwen-portal-auth` | Qwen/Alibaba Portal OAuth (`extensions/qwen-portal-auth/openclaw.plugin.json` exists) |
| `copilot-proxy` | GitHub Copilot Proxy |

### 1.4 Memory Backends (2)

See also: `docs/concepts/memory.md`

| Extension | Description |
|---|---|
| `memory-core` | File-based Memory (simple, kind: "memory") (`extensions/memory-core/openclaw.plugin.json`) |
| `memory-lancedb` | LanceDB Vector Store with Embeddings (kind: "memory") (`extensions/memory-lancedb/openclaw.plugin.json`) |

### 1.5 Specialized (5+)

| Extension | Description |
|---|---|
| `llm-task` | Generic JSON LLM Tool for Workflows |
| `diagnostics-otel` | OpenTelemetry Diagnostics |
| `thread-ownership` | Slack Thread Ownership (prevents agent collisions) |
| `lobster` | Typed Workflows with Approvals |
| `open-prose` | OpenProse VM Skill Pack |

---

## 2. Skills (50+ Total)

Skills are user-facing capabilities -- mostly CLI wrappers or API integrations in Markdown.

Source: Bundled skills live at `skills/*/SKILL.md` (60+ bundled skill directories).
See also: `docs/tools/skills.md`, `docs/tools/creating-skills.md`

### 2.1 Category Overview

| Category | Skills | Examples |
|---|---|---|
| **Code & Dev** | 4 | `coding-agent`, `github`, `gh-issues`, `clawhub` |
| **Notes** | 5 | `apple-notes`, `bear-notes`, `notion`, `obsidian` |
| **Tasks** | 2 | `apple-reminders`, `things-mac` |
| **Communication** | 5 | `discord`, `slack`, `bluebubbles`, `imsg`, `himalaya` (Email) |
| **Smart Home / IoT** | 2 | `openhue` (Philips Hue), `eightctl` (Eight Sleep) |
| **Audio/Music** | 4 | `spotify-player`, `sonoscli`, `blucli`, `sherpa-onnx-tts` |
| **Media** | 4 | `video-frames`, `camsnap`, `gifgrep`, `nano-pdf` |
| **AI/Image Generation** | 3 | `openai-image-gen`, `nano-banana-pro`, `gemini` |
| **Utilities** | 7 | `weather`, `goplaces`, `canvas`, `tmux`, `healthcheck`, `model-usage`, `blogwatcher` |
| **Meta** | 2 | `skill-creator`, `session-logs` |

### 2.2 Particularly Interesting Skills

**`coding-agent`** -- Delegates to Codex/Claude Code/Pi Agents. Requires `pty: true`.
Source: `skills/coding-agent/SKILL.md:3` ("Requires a bash tool that supports pty:true"), `skills/coding-agent/SKILL.md:18`

**`gh-issues`** -- 6-phase orchestrator that automatically fixes GitHub Issues (Source: `skills/gh-issues/SKILL.md`):
1. Load issues
2. Analyze & prioritize
3. Create branches
4. Spawn up to 8 parallel sub-agents (one per issue)
5. Create PRs
6. Summarize results

**`skill-creator`** -- Bootstraps new skills (init, package, validate). Source: `skills/skill-creator/SKILL.md`

**`canvas`** -- Displays HTML content on connected OpenClaw Nodes (Mac/iOS/Android). Source: `skills/canvas/SKILL.md`

---

## 3. Creating Skills

### 3.1 Directory Structure

```
my-skill/
├── SKILL.md                 # Frontmatter + Instructions
├── package.json             # Node Dependencies (optional)
├── scripts/                 # Executable Scripts
│   ├── my-script.sh
│   └── my-script.py
├── references/              # Additional Documentation
│   └── examples.md
└── bin/                     # Precompiled Binaries
    └── my-tool
```

### 3.2 SKILL.md Format

```yaml
---
name: skill-name
description: What this skill does
homepage: https://project.url
metadata:
  openclaw:
    emoji: "🎯"
    requires:
      bins: ["binary-name"]          # Required CLI tools
      config: ["path.to.config"]     # Required config keys
    install:
      - id: "brew"
        kind: "brew"
        formula: "package-name"
        bins: ["binary"]
      - id: "go"
        kind: "go"
        module: "github.com/org/pkg@latest"
        bins: ["binary"]
user-invocable: true                 # Usable as /slash-command
allowed-tools: ["tool-name"]         # Restrict tool access
---

# Skill Title

Instructions in natural language for the agent...

## Examples
...
```

### 3.3 Skill Sources (Loading Order)

Precedence (highest wins on name conflict):

1. **Workspace skills**: `<workspace>/skills/` (per-agent, highest precedence)
2. **Managed/local skills**: `~/.openclaw/skills/` (shared across agents)
3. **Bundled skills**: shipped with the install (`skills/*/SKILL.md` in the OpenClaw repo, lowest precedence)
4. **Extra dirs**: via `skills.load.extraDirs` in config (lowest precedence)
5. **Extensions**: Via `openclaw.plugin.json` -> `"skills": ["./skills"]`
6. **ClawHub**: Download from the Skill Registry (`clawhub` skill) -- see `docs/tools/clawhub.md`

Source: `docs/tools/skills.md:15-26`, `docs/tools/skills-config.md`

### 3.4 Differences from Claude Code Skills

| Aspect | OpenClaw Skills | Claude Code Skills |
|---|---|---|
| **Scope** | Workspace-scoped, filterable per agent | Global for the CLI |
| **Format** | SKILL.md with YAML Frontmatter + Install info | Markdown (simpler) |
| **Distribution** | ClawHub Registry (publish/install) | Local |
| **Integration** | Hooks, Channels, Bindings, Sub-agents | CLI context only |
| **Invocation** | `/slash-commands`, automatically in context | `/slash-commands` |
| **Dependencies** | Declarative `requires.bins` + Install instructions | No formal dependency management |
| **Context** | Agent config, Channels, Auth, other Skills | Codebase only |

---

## 4. Orchestration -- All Options

See also: `docs/automation/hooks.md`, `docs/automation/webhook.md`, `docs/automation/cron-jobs.md`, `docs/tools/subagents.md`, `docs/concepts/multi-agent.md`

### 4.1 Overview

```
┌─────────────────────────────────────────────┐
│           Orchestration Methods              │
├──────────────┬──────────────────────────────┤
│ Declarative  │ Bindings (Channel → Agent)   │
│              │ Tool Policies (Allow/Deny)    │
│              │ Sandbox Config                │
├──────────────┼──────────────────────────────┤
│ Event-driven │ Hooks (20+ Lifecycle Events)  │
│              │ Webhooks (HTTP Inbound)        │
│              │ Auto-Reply (Group Gating)      │
├──────────────┼──────────────────────────────┤
│ Time-based   │ HEARTBEAT.md (periodic)       │
│              │ BOOT.md (once at startup)      │
│              │ Cron Jobs                      │
├──────────────┼──────────────────────────────┤
│ Agent-driven │ sessions_spawn (Sub-agents)   │  src/agents/tools/sessions-spawn-tool.ts:35
│              │ sessions_send (Cross-Session)  │  src/agents/tools/sessions-send-tool.ts:42
│              │ sessions_list (Monitoring)     │  src/agents/tools/sessions-list-tool.ts:35
└──────────────┴──────────────────────────────┘
```

### 4.2 BOOT.md -- Once at Startup

Executed per configured agent when the gateway starts.
Source: `src/gateway/boot.ts:35-50`, `src/hooks/bundled/boot-md/HOOK.md`
See also: `docs/reference/templates/BOOT.md`

```markdown
# BOOT.md
Check system status, initialize connections, send startup message.
If sending a message, use message tool then return NO_REPLY.
```

### 4.3 HEARTBEAT.md -- Periodic

Executed repeatedly at a configurable interval (default: `30m`).
Source: `src/auto-reply/heartbeat.ts:8` (`DEFAULT_HEARTBEAT_EVERY = "30m"`), `src/infra/heartbeat-runner.ts`
See also: `docs/gateway/heartbeat.md`, `docs/reference/templates/HEARTBEAT.md`, `docs/automation/cron-vs-heartbeat.md`

```markdown
# HEARTBEAT.md
Check for updates, monitor status, poll external services.
```

### 4.4 Hooks -- Event-driven Logic

Source: `src/hooks/internal-hooks.ts:12` (event types), `src/plugins/registry.ts:490` (plugin registration)
See also: `docs/automation/hooks.md`

```typescript
// In a plugin:
api.registerHook(
  ["command:new", "message:received"],
  async (event) => { /* Handler */ },
  { entry: hookEntry }
);
```

**Hook Event Types** (from `src/hooks/internal-hooks.ts:12`):
- General types: `command`, `session`, `agent`, `gateway`, `message`
- Each type has specific actions via `type:action` format.

**Available Hook Events** (from `docs/automation/hooks.md:237-262`):
- **Command events**: `command:new`, `command:reset`, `command:stop`
- **Agent events**: `agent:bootstrap`
- **Gateway events**: `gateway:startup`
- **Message events**: `message:received`, `message:sent`

Hooks can also listen to the general type (e.g., `command` catches all command actions).
Source: `src/hooks/internal-hooks.ts:182-192`

---

## 5. Plugin SDK – Creating Custom Extensions

### 5.1 Minimal Plugin

Source: `src/plugins/types.ts` (type definitions), `src/plugins/registry.ts:478-502` (API construction)
See also: `docs/tools/plugin.md`

```typescript
// extensions/my-plugin/index.ts
import type { OpenClawPluginApi } from "openclaw/plugin-sdk";

export default {
  id: "my-plugin",
  register(api: OpenClawPluginApi) {
    api.registerTool("my-tool", myToolImpl);
    api.registerHook(["message:received"], myHandler);
  },
};
```

### 5.2 Plugin Manifest

```json
// extensions/my-plugin/openclaw.plugin.json
{
  "id": "my-plugin",
  "name": "My Plugin",
  "description": "What it does",
  "channels": ["my-channel"],
  "skills": ["./skills"],
  "configSchema": {
    "type": "object",
    "properties": { }
  }
}
```

### 5.3 Plugin API -- Key Methods

Source: `src/plugins/registry.ts:489-501`

| Method | Purpose |
|---|---|
| `api.registerTool()` | Register a tool for the agent (line 489) |
| `api.registerHook()` | Register an event handler (line 490) |
| `api.registerHttpHandler()` | Register a webhook handler (line 492) |
| `api.registerHttpRoute()` | Register an HTTP route (line 493) |
| `api.registerChannel()` | Register a new messaging channel (line 494) |
| `api.registerProvider()` | Register an auth provider (line 495) |
| `api.registerGatewayMethod()` | Register a Gateway RPC method (line 496) |
| `api.registerCli()` | Register CLI subcommands (line 497) |
| `api.registerService()` | Register a background service (line 498) |
| `api.registerCommand()` | Register a CLI command (line 499) |
| `api.on()` | Register a typed hook handler (line 501) |

Note: There is no `api.registerMemoryBackend()` -- memory backends are registered as plugins with `kind: "memory"` (`src/plugins/types.ts:37`).

---

## 6. Skill Updates & Management

### 6.1 Overview of Update Methods

| Method | Purpose | Type |
|---|---|---|
| **ClawHub CLI** | Download/update skills from registry | External |
| **OpenClaw CLI** | List skills, inspect, check dependencies | Read-Only |
| **Gateway API** | Change skill config (enable/disable, API keys, env) | RPC |
| **File Watching** | Automatic detection of file changes | Automatic |

### 6.2 ClawHub CLI (Registry Skills)

See also: `docs/tools/clawhub.md`

ClawHub is a **separate CLI tool** -- like npm for skills:

```bash
clawhub update <slug>                    # Update a single skill
clawhub update --all                     # Update all installed skills
clawhub update <slug> --version 2.0.0   # Specific version
clawhub update <slug> --force            # Overwrite local changes
```

Tracking via `.clawhub/lock.json` (analogous to `package-lock.json`).

### 6.3 OpenClaw CLI (Read-Only)

The built-in commands are **informational only**, no updates possible:

```bash
openclaw skills list          # List all available skills
openclaw skills info <name>   # Details about a skill
openclaw skills check         # Check: ready vs. missing dependencies
```

Source: `src/cli/skills-cli.ts`

### 6.4 Gateway API (Config Changes)

Two RPC methods via the Gateway WebSocket API:

- **`skills.update`** (line 146) -- Enable/disable skills, set API keys and env vars
  - Writes to the config file via `writeConfigFile()` (line 201)
  - Does **not** change the skill code, only the configuration
- **`skills.install`** (line 114) -- Install dependencies for a skill (brew, npm, go, etc.)
  - Uses the `install` entries from the SKILL.md frontmatter

Source: `src/gateway/server-methods/skills.ts:114,146,201`
See also: `docs/tools/skills-config.md`

### 6.5 Hot-Reload via File Watching

OpenClaw uses **chokidar** for automatic detection of skill changes (`src/agents/skills/refresh.ts:3`):

**Watched Directories:**
- `<workspace>/skills/*/SKILL.md`
- `<workspace>/.agents/skills/*/SKILL.md`
- `~/.openclaw/skills/*/SKILL.md`
- Additional folders via `config.skills.load.extraDirs`

**Process on Change:**
1. chokidar detects file change (with debounce)
2. `bumpSkillsSnapshotVersion()` is called
3. Registered listeners are notified
4. Next agent session loads the updated skill automatically

**No gateway restart required** for changes to SKILL.md files.

**However:** Changes to the `skills:` config section (in the OpenClaw configuration) trigger a **full gateway restart**.

Source: `src/agents/skills/refresh.ts`

### 6.6 Skill Discovery Model

OpenClaw uses **pull-based discovery** (not push):

```
ClawHub downloads skill to disk → chokidar detects → version bump → session loads lazily
```

- Skills are **not** centrally cached, but read from the filesystem on demand
- Each new agent session checks the snapshot version and reloads if necessary
- Remote nodes (connected devices) can provide additional binaries that make skills eligible

Source: `src/agents/skills/workspace.ts`, `src/infra/skills-remote.ts`

---

## References

### Source Files

| File | Relevant Section(s) |
|---|---|
| `extensions/*/openclaw.plugin.json` | Section 1 (36 extension manifests) |
| `skills/*/SKILL.md` | Section 2 (60+ bundled skills) |
| `skills/coding-agent/SKILL.md` | Section 2.2 (coding-agent, pty requirement) |
| `skills/gh-issues/SKILL.md` | Section 2.2 (gh-issues orchestrator) |
| `src/plugins/types.ts` | Section 5 (Plugin SDK types, PluginKind at line 37) |
| `src/plugins/registry.ts` | Section 5.3 (Plugin API methods, lines 489-501) |
| `src/hooks/internal-hooks.ts` | Section 4.4 (Hook event types, line 12; trigger logic, lines 182-192) |
| `src/hooks/types.ts` | Section 4.4 (HookEntry types, hook metadata) |
| `src/gateway/boot.ts` | Section 4.2 (BOOT.md execution, lines 35-50) |
| `src/hooks/bundled/boot-md/HOOK.md` | Section 4.2 (boot-md bundled hook) |
| `src/auto-reply/heartbeat.ts` | Section 4.3 (Heartbeat defaults, line 8) |
| `src/infra/heartbeat-runner.ts` | Section 4.3 (Heartbeat execution) |
| `src/agents/tools/sessions-spawn-tool.ts` | Section 4.1 (sessions_spawn, line 35) |
| `src/agents/tools/sessions-send-tool.ts` | Section 4.1 (sessions_send, line 42) |
| `src/agents/tools/sessions-list-tool.ts` | Section 4.1 (sessions_list, line 35) |
| `src/cli/skills-cli.ts` | Section 6.3 (OpenClaw CLI skill commands) |
| `src/gateway/server-methods/skills.ts` | Section 6.4 (skills.install at line 114, skills.update at line 146) |
| `src/agents/skills/refresh.ts` | Section 6.5 (chokidar watcher, bumpSkillsSnapshotVersion at line 105) |
| `src/agents/skills/workspace.ts` | Section 6.6 (skill discovery) |
| `src/infra/skills-remote.ts` | Section 6.6 (remote node skill eligibility) |

### Documentation Cross-References

| Doc Page | Relevant Section(s) |
|---|---|
| `docs/tools/plugin.md` | Section 1, 5 (Plugin/extension overview) |
| `docs/tools/skills.md` | Section 2, 3.3 (Skills loading, precedence, per-agent vs shared) |
| `docs/tools/creating-skills.md` | Section 3 (Creating custom skills) |
| `docs/tools/skills-config.md` | Section 3.3, 6.4 (Skills config schema, extraDirs, install options) |
| `docs/tools/clawhub.md` | Section 6.2 (ClawHub public registry) |
| `docs/tools/subagents.md` | Section 4.1 (Sub-agent spawning, announce-back behavior) |
| `docs/automation/hooks.md` | Section 4.4 (Hook events, handler format, bundled hooks) |
| `docs/automation/webhook.md` | Section 4.1 (Webhook inbound automation) |
| `docs/automation/cron-jobs.md` | Section 4.1 (Cron-based scheduling) |
| `docs/automation/cron-vs-heartbeat.md` | Section 4.3 (When to use cron vs heartbeat) |
| `docs/gateway/heartbeat.md` | Section 4.3 (Heartbeat configuration, cadence, targets) |
| `docs/reference/templates/BOOT.md` | Section 4.2 (BOOT.md template) |
| `docs/reference/templates/HEARTBEAT.md` | Section 4.3 (HEARTBEAT.md template) |
| `docs/channels/index.md` | Section 1.1 (Channel overview) |
| `docs/concepts/memory.md` | Section 1.4 (Memory backends and file layout) |
| `docs/concepts/multi-agent.md` | Section 4 (Multi-agent routing, isolated workspaces) |
| `docs/concepts/oauth.md` | Section 1.3 (Auth providers) |
| `docs/concepts/model-providers.md` | Section 1.3 (Model/auth provider configuration) |
