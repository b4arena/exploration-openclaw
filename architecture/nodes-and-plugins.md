# Nodes and Plugin System

The OpenClaw node concept (companion devices) and the plugin system (extensibility architecture).

## Part 1: Nodes — Companion Devices

### What Is a Node?

A **node** is a companion device — macOS app, iOS/Android phone, headless Linux host — that connects to the Gateway via WebSocket with `role: "node"` and exposes a command surface (e.g. `canvas.*`, `camera.*`, `system.*`) invoked via `node.invoke`.

Nodes are **peripherals**, not additional gateways. There is one gateway; nodes extend its reach into the physical world.

Source: `docs/nodes/index.md:12`

### Node Identity

Every node has a stable UUID identity, generated once and persisted in `~/.openclaw/node.json`:

```ts
// src/node-host/config.ts:13-21
export type NodeHostConfig = {
  version: 1;
  nodeId: string;       // UUID, generated once via crypto.randomUUID()
  token?: string;
  displayName?: string;
  gateway?: NodeHostGatewayConfig;
};
```

The `--node-id <id>` CLI flag on `openclaw node run` can override the persisted ID (`docs/cli/node.md:58`).

### Pairing Lifecycle

Nodes must be approved before they can interact with the gateway:

1. **Request** — Node connects via WS and sends `node.pair.request` with `nodeId`, `displayName`, platform, capabilities, commands
2. **Pending** — Gateway stores a pending pairing request (`src/infra/node-pairing.ts:112-152`), TTL 5 minutes
3. **Approve** — Admin runs `openclaw nodes approve <requestId>` → generates and stores a token
4. **Verify** — Node stores its token in `~/.openclaw/node.json`; subsequent connections authenticate via `node.pair.verify`

### NodeRegistry — Runtime Connection Tracking

The `NodeRegistry` (`src/gateway/node-registry.ts:38-209`) is the in-memory index of connected nodes on the gateway:

| Field | Type | Purpose |
|---|---|---|
| `nodesById` | `Map<string, NodeSession>` | All connected nodes by `nodeId` |
| `nodesByConn` | `Map<string, string>` | WS connection ID → `nodeId` (for disconnect cleanup) |
| `pendingInvokes` | `Map<string, PendingInvoke>` | In-flight `node.invoke` requests awaiting response |

Key methods: `register()`, `unregister()`, `invoke()` (with 30s timeout), `listConnected()`, `sendEvent()`.

### NodeSession Type

```ts
// src/gateway/node-registry.ts:4-21
export type NodeSession = {
  nodeId: string; connId: string; client: GatewayWsClient;
  displayName?: string; platform?: string; version?: string;
  coreVersion?: string; uiVersion?: string; deviceFamily?: string;
  modelIdentifier?: string; remoteIp?: string;
  caps: string[]; commands: string[];
  permissions?: Record<string, boolean>;
  pathEnv?: string; connectedAtMs: number;
};
```

### Gateway Protocol (WebSocket RPC)

All node-related WS methods are defined in `src/gateway/protocol/schema/nodes.ts`:

| Method | Direction | Purpose |
|---|---|---|
| `node.pair.request` | Node→Gateway | Request pairing |
| `node.pair.list` | Client→Gateway | List pending/paired |
| `node.pair.approve` | Client→Gateway | Approve pending request |
| `node.pair.reject` | Client→Gateway | Reject pending request |
| `node.pair.verify` | Node→Gateway | Verify stored token |
| `node.rename` | Client→Gateway | Rename a paired node |
| `node.list` | Client→Gateway | List all known nodes |
| `node.describe` | Client→Gateway | Describe a single node |
| `node.invoke` | Client→Gateway | Invoke a command on a node |
| `node.invoke.result` | Node→Gateway | Return result of a command |
| `node.event` | Node→Gateway | Fire an event (voice, exec, push, etc.) |

Handlers: `src/gateway/server-methods/nodes.ts:262-786`

### Command Surface and Allowlisting

Commands a node exposes are declared in its `connect` handshake (`commands: string[]`). The gateway enforces a platform-keyed allowlist (`src/gateway/node-command-policy.ts`):

| Platform | Default commands |
|---|---|
| `ios` / `android` | `canvas.*`, `camera.list`, `location.get`, `device.info/status`, `contacts.*`, `calendar.*`, `reminders.*`, `photos.*`, `motion.*`, `system.notify` |
| `macos` | All of above + `system.run`, `system.which`, `system.notify`, `browser.proxy` |
| `linux` / `windows` | `system.run`, `system.which`, `system.notify`, `browser.proxy` |

**Dangerous commands** (require explicit `gateway.nodes.allowCommands`): `camera.snap`, `camera.clip`, `screen.record`, `contacts.add`, `calendar.add`, `reminders.add`, `sms.send` (`src/gateway/node-command-policy.ts:46-53`).

Config keys:
- `gateway.nodes.allowCommands: string[]` — extra commands to allow
- `gateway.nodes.denyCommands: string[]` — block even if declared

### Node Events

Nodes send events to the gateway via `node.event` (`src/gateway/server-node-events.ts`):

| Event | What it does |
|---|---|
| `voice.transcript` | Feeds text to the agent command runner |
| `agent.request` | Triggers an agent run (iOS share sheet, etc.) |
| `chat.subscribe` | Subscribe to live chat stream for a session |
| `chat.unsubscribe` | Unsubscribe from chat stream |
| `exec.started/finished/denied` | Converts to system events visible in chat |
| `push.apns.register` | Registers APNS token for push wakes |

### Headless Node Host

The headless node host (`src/node-host/`) runs on a remote machine to provide `system.run` remotely:

- **Runner** (`src/node-host/runner.ts:70-162`): Reads `~/.openclaw/node.json`, connects to Gateway WS, advertises capabilities (`system`, optionally `browser`), listens for `node.invoke.request`
- **Invoke handler** (`src/node-host/invoke.ts`): Handles `system.run`, `system.which`, `browser.proxy`. Enforces local exec approvals (`~/.openclaw/exec-approvals.json`). Sanitizes dangerous env vars (`NODE_OPTIONS`, `PYTHONHOME`, `DYLD_*`, `LD_*`, etc.)
- **Browser proxy**: If `cfg.nodeHost.browserProxy.enabled !== false`, also advertises `browser.proxy` cap

### Agent Exec Routing to Nodes

When `tools.exec.host` is `"node"`, all agent bash/exec runs are forwarded to a paired node instead of running locally on the gateway sandbox (`src/agents/bash-tools.exec-host-node.ts:25-42`).

Config: `tools.exec.node` specifies the default node binding by id/name (`src/config/types.tools.ts:187-188`).

### Node Wake via APNS

When `node.invoke` targets a disconnected node, the gateway attempts to wake it via Apple Push Notification Service (`src/gateway/server-methods/nodes.ts:89-241`):

1. **Wake attempt 1**: APNS background wake, wait up to 3s
2. **Wake attempt 2**: Force retry, wait up to 12s
3. **Fallback**: User-visible APNS alert (throttled to 1 per 10 min)

### Agent `nodes` Tool

The LLM-facing tool (`src/agents/tools/nodes-tool.ts`) exposes actions: `status`, `describe`, `pending`, `approve`, `reject`, `notify`, `camera_snap`, `camera_list`, `camera_clip`, `screen_record`, `location_get`, `run`, `invoke`.

The `run` action handles the full approval flow: if denied with `SYSTEM_RUN_DENIED: approval required`, it creates an `exec.approval.request` and retries with the user's decision.

### CLI Surface

```bash
# Node host management (run on the companion device)
openclaw node run    --host <gw-host> --port 18789 [--tls] [--node-id <id>]
openclaw node install --host ... [--runtime node|bun]
openclaw node status | stop | restart | uninstall

# Node management (run on the gateway or admin machine)
openclaw nodes list [--connected] [--last-connected 24h]
openclaw nodes pending
openclaw nodes approve <requestId>
openclaw nodes invoke --node <id|name|ip> --command <cmd> --params <json>
openclaw nodes run   --node <id|name|ip> <command...>
openclaw nodes camera snap/clip/list --node ...
openclaw nodes canvas snapshot/present/hide/navigate/eval --node ...
openclaw nodes location get --node ...
openclaw nodes notify --node ... --title ... --body ...
```

Node matching (`src/shared/node-match.ts:22-48`): query is matched against `nodeId` (exact), `remoteIp` (exact), `displayName` (normalized), or `nodeId` prefix (≥6 chars).

### Node Config Summary

| Config key | Type | Description |
|---|---|---|
| `tools.exec.host` | `"sandbox"\|"gateway"\|"node"` | Where agent exec runs |
| `tools.exec.node` | `string` | Default node id/name for `host=node` |
| `gateway.nodes.allowCommands` | `string[]` | Extra commands to allow |
| `gateway.nodes.denyCommands` | `string[]` | Block commands even if declared |
| `gateway.nodes.browser.mode` | `"auto"\|"manual"\|"off"` | Browser routing mode |
| `gateway.nodes.browser.node` | `string` | Pin browser proxy to specific node |
| `nodeHost.browserProxy.enabled` | `boolean` | Expose browser proxy on this node host |
| `nodeHost.browserProxy.allowProfiles` | `string[]` | Allowed browser profiles |

---

## Part 2: Plugin System

### Overview

Plugins are the primary extensibility mechanism in OpenClaw. Everything beyond the core gateway — channels, providers, memory, tools, voice — is a plugin.

### Plugin Manifest (`openclaw.plugin.json`)

Every plugin must ship an `openclaw.plugin.json` in its root directory (`src/plugins/manifest.ts:7-21`, `docs/plugins/manifest.md`).

**Required fields:**
```json
{
  "id": "my-plugin",
  "configSchema": { "type": "object", "additionalProperties": false, "properties": {} }
}
```

**Optional fields:**

| Field | Type | Purpose |
|---|---|---|
| `kind` | `"memory"` | Marks plugin as occupying an exclusive slot |
| `channels` | `string[]` | Channel IDs this plugin registers |
| `providers` | `string[]` | Provider IDs this plugin registers |
| `skills` | `string[]` | Relative paths to skill directories to auto-load |
| `name` | `string` | Display name |
| `description` | `string` | Short summary |
| `version` | `string` | Informational |
| `uiHints` | `Record<string, {...}>` | UI metadata for config fields (label, help, advanced, sensitive, placeholder) |

### Plugin Module Format

A plugin module exports a register function (`src/plugins/types.ts:229-242`, `src/plugins/loader.ts:118-139`):

```ts
// Option A: default function (simplest)
import type { OpenClawPluginApi } from "openclaw/plugin-sdk";

export default function (api: OpenClawPluginApi) {
  api.registerTool(...);
}

// Option B: object with register() or activate() method
export default {
  id: "my-plugin",
  name: "My Plugin",
  register(api) { ... },
};
```

Registration must be **synchronous** — async `register()` is silently ignored (`loader.ts:626-631`).

### Plugin API (`OpenClawPluginApi`)

All registration methods available on the `api` object (`src/plugins/types.ts:244-283`):

| Method | What it does |
|---|---|
| `api.registerTool(tool, opts?)` | Register an LLM agent tool. `opts.optional=true` = opt-in via `tools.alsoAllow` |
| `api.on(hookName, handler, opts?)` | Register a typed lifecycle hook |
| `api.registerChannel(registration)` | Register a messaging channel |
| `api.registerProvider(provider)` | Register an LLM provider |
| `api.registerService(service)` | Register a background service with start/stop |
| `api.registerCommand(command)` | Register a slash-command that bypasses the LLM |
| `api.registerCli(registrar, opts?)` | Add CLI subcommands |
| `api.registerHttpHandler(handler)` | Register a raw HTTP handler |
| `api.registerHttpRoute({path, handler})` | Register a path-matched HTTP route |
| `api.registerGatewayMethod(method, handler)` | Register a Gateway RPC method |
| `api.resolvePath(input)` | Resolve a user-relative path |

Context on `api`: `api.id`, `api.name`, `api.version`, `api.source`, `api.config` (global), `api.pluginConfig` (validated plugin-specific), `api.runtime`, `api.logger`.

### Plugin Kinds / Functional Types

Only one formal `kind` exists — `"memory"` (exclusive slot, `src/plugins/slots.ts:12-18`). All other types are defined by what they register:

| Functional type | Registration call | Examples |
|---|---|---|
| Channel | `registerChannel()` | slack, telegram, discord, signal, whatsapp, matrix, irc, line, etc. |
| Provider/auth | `registerProvider()` | google-gemini-cli-auth, copilot-proxy, qwen-portal-auth |
| Memory | `kind: "memory"` + `registerTool()` | memory-core, memory-lancedb |
| Tool | `registerTool()` | llm-task, voice-call, phone-control |
| Service | `registerService()` | diagnostics-otel |
| Hook/observer | `api.on()` | thread-ownership |
| Skills-only | manifest `"skills": [...]` | open-prose |
| Combined | Multiple registrations | voice-call (tool + service + CLI + gateway method) |

### Plugin SDK (`openclaw/plugin-sdk`)

Plugins import from the virtual module `"openclaw/plugin-sdk"` (`src/plugin-sdk/index.ts`, 466 lines of exports). The loader aliases this at runtime via Jiti (`loader.ts:405`).

Key exports:
- Channel building: `ChannelPlugin`, `ChannelMessagingAdapter`, `ChannelAuthAdapter`
- Core types: `OpenClawPluginApi`, `AnyAgentTool`, `OpenClawPluginService`, `ProviderAuthContext`
- HTTP: `GatewayRequestHandler`
- Webhook helpers: `normalizeWebhookPath()`, `resolveWebhookPath()`, `registerWebhookTarget()`
- File ops: `acquireFileLock()`, `withFileLock()`, `readJsonFileWithFallback()`, `writeJsonFileAtomically()`
- Media: `buildAgentMediaPayload()`
- Auth: `buildOauthProviderAuthResult()`
- Diagnostics: `emitDiagnosticEvent()`, `onDiagnosticEvent()`
- Per-channel helpers: Slack, Telegram, Discord, Signal, WhatsApp, iMessage, LINE

### Plugin Discovery

Plugins are discovered in priority order (`src/plugins/discovery.ts:537-605`):

1. **Config paths** (`plugins.load.paths`) — `origin: "config"`
2. **Workspace** — `<workspaceDir>/.openclaw/extensions/` — `origin: "workspace"`
3. **Global** — `~/.config/openclaw/extensions/` — `origin: "global"`
4. **Bundled** — resolved via `resolveBundledPluginsDir()` — `origin: "bundled"`

Within a directory: finds `.ts`/`.js`/`.mjs`/`.cjs` files directly, or reads `package.json` → `openclaw.extensions` array, or falls back to `index.ts`/`index.js`.

**Security checks** block plugins if: source escapes the plugin root (symlink attacks), path is world-writable, or suspicious ownership (not current user or root).

### Plugin Loading Pipeline

The full `loadOpenClawPlugins()` pipeline (`src/plugins/loader.ts:334-672`):

1. Discover candidates
2. Load manifest registry (validate all `openclaw.plugin.json` without executing code)
3. For each candidate:
   - Check enable state (allow/deny lists, `plugins.entries.<id>.enabled`)
   - Check `memory` slot conflicts
   - Validate config against manifest JSON Schema
   - Load module via **Jiti** (TypeScript transpiler, `loader.ts:509`)
   - Call `register(api)` synchronously
4. Cache result by workspace + config key
5. Initialize global hook runner

**Bundled-enabled-by-default** (`src/plugins/config-state.ts:16-20`): `device-pair`, `phone-control`, `talk-voice`.

### Plugin Hooks (Lifecycle Events)

The typed hook system (`api.on(hookName, handler, opts?)`) — `src/plugins/types.ts:298-644`:

**Agent hooks (sequential — can modify):**

| Hook | Return |
|---|---|
| `before_model_resolve` | `{modelOverride?, providerOverride?}` |
| `before_prompt_build` | `{systemPrompt?, prependContext?}` |
| `before_agent_start` | Combined of above two |

**Agent hooks (parallel — observe only):**

| Hook | Fires when |
|---|---|
| `llm_input` | Before LLM call |
| `llm_output` | After LLM response |
| `agent_end` | Agent run completes |
| `before_compaction` / `after_compaction` | Context compaction |
| `before_reset` | Session reset |

**Message hooks:**

| Hook | Type | Can modify? |
|---|---|---|
| `message_received` | parallel | No |
| `message_sending` | sequential | Yes (`{content?, cancel?}`) |
| `message_sent` | parallel | No |

**Tool hooks:**

| Hook | Type | Can modify? |
|---|---|---|
| `before_tool_call` | sequential | Yes (`{params?, block?, blockReason?}`) |
| `after_tool_call` | parallel | No |
| `tool_result_persist` | sync sequential | Yes (`{message?}`) |
| `before_message_write` | sync sequential | Yes (`{block?, message?}`) |

**Session hooks** (parallel): `session_start`, `session_end`
**Gateway hooks** (parallel): `gateway_start`, `gateway_stop`

Priority: `opts.priority` (higher = earlier, default 0).

### Plugin Runtime (`PluginRuntime`)

The `api.runtime` object gives plugins access to core platform internals (`src/plugins/runtime/types.ts:179-364`):

- `runtime.config.{loadConfig, writeConfigFile}` — read/write config
- `runtime.media.{loadWebMedia, detectMime, resizeToJpeg}` — media ops
- `runtime.tts.textToSpeechTelephony` — TTS for calls
- `runtime.tools.{createMemoryGetTool, createMemorySearchTool}` — memory tool factories
- `runtime.channel.*` — per-channel send/probe/monitor/pairing/routing/reactions
- `runtime.logging.{shouldLogVerbose, getChildLogger}`
- `runtime.state.resolveStateDir`
- `runtime.system.{enqueueSystemEvent, runCommandWithTimeout}`

### Plugin Installation

```bash
openclaw plugins install @openclaw/voice-call    # from npm
openclaw plugins install ./extensions/voice-call  # from local path
```

Source: `src/plugins/install.ts`. npm install uses `npm pack` + extraction + integrity check.

### Plugin Config in `openclaw.json`

```json5
{
  "plugins": {
    "enabled": true,
    "allow": ["slack", "my-plugin"],
    "deny": ["bad-plugin"],
    "load": { "paths": ["./my-extensions"] },
    "slots": { "memory": "memory-lancedb" },
    "entries": {
      "my-plugin": {
        "enabled": true,
        "config": { "someKey": "someValue" }
      }
    }
  }
}
```

Source: `src/plugins/config-state.ts`, `docs/plugins/manifest.md`

### Minimal Plugin Example

```ts
// index.ts
import type { OpenClawPluginApi } from "openclaw/plugin-sdk";

export default function register(api: OpenClawPluginApi) {
  api.registerTool({
    name: "my_tool",
    description: "Does a thing",
    parameters: {
      type: "object",
      properties: { input: { type: "string" } },
      required: ["input"],
    },
    async execute(_id, params) {
      return { content: [{ type: "text", text: params.input }] };
    },
  });

  api.on("session_start", async (event, ctx) => {
    api.logger.info(`Session started: ${event.sessionId}`);
  });
}
```

```json
// openclaw.plugin.json
{
  "id": "my-plugin",
  "name": "My Plugin",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  }
}
```

### Real Plugin Anatomy (by complexity)

**Minimal hook plugin** (`extensions/thread-ownership/`):
```
openclaw.plugin.json
index.ts               # api.on("message_received") + api.on("message_sending")
```

**Skills-only plugin** (`extensions/open-prose/`):
```
openclaw.plugin.json   # "skills": ["./skills"]
index.ts               # empty register(), skills loaded via manifest
skills/                # SKILL.md files
```

**Memory plugin** (`extensions/memory-lancedb/`):
```
openclaw.plugin.json   # "kind": "memory"
index.ts               # api.registerTool() + api.on()
```

**Provider auth plugin** (`extensions/google-gemini-cli-auth/`):
```
openclaw.plugin.json   # "providers": ["google-gemini-cli"]
index.ts               # api.registerProvider({id, label, auth: [...]})
```

**Complex combined plugin** (`extensions/voice-call/`):
```
openclaw.plugin.json   # 559 lines of configSchema + uiHints
index.ts               # registerTool() + registerService() + registerCli() + registerGatewayMethod()
src/service.ts
```

### Key File Index

| File | Purpose |
|---|---|
| **Nodes** | |
| `docs/nodes/index.md` | Authoritative node overview |
| `docs/cli/node.md` / `docs/cli/nodes.md` | CLI reference |
| `src/gateway/node-registry.ts` | `NodeRegistry`, `NodeSession`, `NodeInvokeResult` |
| `src/gateway/node-command-policy.ts` | Platform allowlists + policy |
| `src/gateway/server-methods/nodes.ts` | All gateway handlers for `node.*` methods |
| `src/gateway/server-node-events.ts` | Node event handling |
| `src/gateway/server-node-subscriptions.ts` | Chat stream subscriptions |
| `src/node-host/runner.ts` | Headless node host main loop |
| `src/node-host/invoke.ts` | Command execution on node |
| `src/node-host/config.ts` | Node-side config types |
| `src/infra/node-pairing.ts` | Pairing state machine |
| `src/agents/tools/nodes-tool.ts` | Agent-facing `nodes` tool |
| `src/agents/bash-tools.exec-host-node.ts` | Exec routing to nodes |
| **Plugins** | |
| `docs/plugins/manifest.md` | Manifest reference |
| `docs/plugins/agent-tools.md` | Tool plugin guide |
| `docs/plugins/community.md` | Community listing requirements |
| `src/plugins/types.ts` | All plugin types + hook definitions |
| `src/plugins/manifest.ts` | Manifest schema + validation |
| `src/plugins/discovery.ts` | Plugin discovery logic |
| `src/plugins/loader.ts` | Full loading pipeline |
| `src/plugins/hooks.ts` | Hook runner |
| `src/plugins/slots.ts` | Exclusive slot management |
| `src/plugins/config-state.ts` | Enable/disable state resolution |
| `src/plugins/install.ts` | `openclaw plugins install` logic |
| `src/plugin-sdk/index.ts` | SDK exports (466 lines) |
| `src/plugins/runtime/types.ts` | `PluginRuntime` interface |
