# Sandbox Container Lifecycle

How OpenClaw manages Docker containers for agent isolation: creation timing, templates, reuse, and configuration.

## Container Creation Timing

**Containers are created eagerly at agent run start — not lazily on first shell exec.**

The call chain is:

```
Agent run starts (embedded runner)
  → attempt.ts:233 / compact.ts:313
    → resolveSandboxContext()           # sandbox/context.ts:85
      → resolveSandboxRuntimeStatus()   # checks sandbox.mode
        → shouldSandboxSession()        # "off" | "non-main" | "all"
      → maybePruneSandboxes()           # clean up stale containers
      → ensureSandboxWorkspaceLayout()  # prepare host directories
      → ensureSandboxContainer()        # sandbox/docker.ts:388
        → dockerContainerState()        # exists? running?
          → No:  createSandboxContainer() → docker create + docker start
          → Yes, stopped: docker start
          → Yes, running: reuse immediately
```

This means: once the system determines that an agent session should be sandboxed (based on `sandbox.mode`), the container is created **before the LLM even generates its first response** — not on the first `exec` tool call.

Source: `src/agents/pi-embedded-runner/run/attempt.ts:233`, `src/agents/sandbox/context.ts:85-110`

## Sandbox Mode — When Is a Session Sandboxed?

The `sandbox.mode` setting (`src/agents/sandbox/runtime-status.ts:10-18`) controls which sessions get sandboxed:

| Mode | Behavior |
|------|----------|
| `"off"` | No sessions are sandboxed |
| `"non-main"` | Only subagent/non-main sessions are sandboxed; the main agent runs on the host |
| `"all"` | All sessions (including the main agent) run in containers |

The decision is made by `shouldSandboxSession()` which compares the session key against the agent's main session key.

Source: `src/agents/sandbox/runtime-status.ts:10-18`, `src/agents/sandbox/types.ts:54`

## Container Templates

Containers are highly configurable with a **3-level hierarchy** (highest priority first):

| Level | Config path | Description |
|-------|-------------|-------------|
| Agent-specific | `agents.list[N].sandbox.docker.*` | Override for one agent |
| Global default | `agents.defaults.sandbox.docker.*` | Default for all agents |
| Built-in | `sandbox/constants.ts` | Hardcoded fallbacks |

Source: `src/agents/sandbox/config.ts:54-97`

### Available Docker Properties

From `src/agents/sandbox/types.docker.ts`:

| Property | Default | Description |
|----------|---------|-------------|
| `image` | `openclaw-sandbox:bookworm-slim` | Docker image |
| `containerPrefix` | `openclaw-sbx-` | Container name prefix |
| `workdir` | `/workspace` | Working directory inside container |
| `readOnlyRoot` | — | Read-only root filesystem |
| `tmpfs` | `/tmp, /var/tmp, /run` | Temporary filesystems |
| `network` | `"none"` | Network mode |
| `user` | — | Run as specific user |
| `capDrop` | `["ALL"]` | Drop Linux capabilities |
| `env` | — | Environment variables |
| `setupCommand` | — | Run once on container creation |
| `pidsLimit` | — | Process count limit |
| `memory` | — | Memory limit |
| `cpus` | — | CPU limit |
| `seccompProfile` | — | Seccomp security profile |
| `apparmorProfile` | — | AppArmor profile |
| `dns` | — | DNS servers |
| `binds` | — | Volume mounts |

### Pre-built Sandbox Images

| Dockerfile | Contents |
|------------|----------|
| `Dockerfile.sandbox` | Minimal: bash, curl, git, jq, python3, rg |
| `Dockerfile.sandbox-common` | Dev tools: + Node.js, Go, Rust, Bun, pnpm |
| `Dockerfile.sandbox-browser` | Browser: + Chromium, Xvfb, VNC, noVNC |

### Example: Per-Agent Configuration

```json
{
  "agents": {
    "defaults": {
      "sandbox": {
        "mode": "all",
        "scope": "agent",
        "docker": {
          "image": "openclaw-sandbox:bookworm-slim",
          "memory": "512m",
          "cpus": 1.0
        }
      }
    },
    "list": [
      {
        "id": "main",
        "sandbox": {
          "docker": {
            "image": "openclaw-sandbox-common:bookworm-slim",
            "memory": "2g",
            "cpus": 4.0,
            "network": "bridge"
          }
        }
      },
      {
        "id": "dev",
        "sandbox": {
          "docker": {
            "image": "openclaw-sandbox-common:bookworm-slim",
            "setupCommand": "apt-get install -y build-essential",
            "memory": "1g"
          }
        }
      }
    ]
  }
}
```

## Container Reuse

Containers are aggressively reused based on **scope** and a **hot-window** mechanism.

### Scope Modes

The `sandbox.scope` setting determines the reuse boundary (`src/agents/sandbox/shared.ts`):

| Scope | Container-per | Cross-session? | Use case |
|-------|---------------|----------------|----------|
| `"shared"` | Entire gateway | Yes | Single shared workspace |
| `"agent"` (default) | Agent ID | Yes | Persistent agent workspace |
| `"session"` | Session | No | Isolated per-conversation |

Container naming follows the pattern: `{containerPrefix}{slugify(scopeKey)}`

### Hot-Window Logic

Source: `src/agents/sandbox/docker.ts:117, 388-462`

```
HOT_CONTAINER_WINDOW_MS = 5 * 60 * 1000  // 5 minutes
```

When `ensureSandboxContainer()` is called:

1. **Container doesn't exist** → create and start new container
2. **Container exists, used < 5 min ago** → reuse immediately (hot window)
3. **Container exists, used > 5 min ago, config hash matches** → reuse (update lastUsedAtMs)
4. **Container exists, used > 5 min ago, config hash mismatch** → delete and recreate

The config hash (`src/agents/sandbox/config-hash.test.ts`) is a fingerprint of the Docker configuration. If you change the image or resource limits, cold containers get recreated automatically.

### Container Registry

The system tracks all containers in a persistent registry at `$OPENCLAW_STATE_DIR/sandbox/containers.json`.

Registry entry fields (`src/agents/sandbox/registry.ts:11-37`):

| Field | Description |
|-------|-------------|
| `containerName` | Docker container name |
| `sessionKey` | Session that created it |
| `createdAtMs` | Creation timestamp |
| `lastUsedAtMs` | Last exec timestamp (updated on every use) |
| `image` | Docker image used |
| `configHash` | Config fingerprint for staleness detection |

### Auto-Pruning

Source: `src/agents/sandbox/prune.ts:22-33`

Containers are automatically pruned when:
- **Idle** for more than `idleHours` (default: 24h)
- **Total age** exceeds `maxAgeDays` (default: 7 days)

Pruning is triggered on `resolveSandboxContext()` but throttled to run at most every 5 minutes.

## Lifecycle Example

```
Agent "dev" starts session A:
  ├─ resolveSandboxContext() → no container → CREATE openclaw-sbx-agent-dev
  ├─ LLM calls exec tool → docker exec openclaw-sbx-agent-dev bash -c "..."
  ├─ LLM calls exec again → same container (hot, scope=agent)
  └─ Session ends → container stays running

Agent "dev" starts session B (10 min later):
  ├─ resolveSandboxContext() → container exists, config hash ok → REUSE
  └─ All exec calls use the same container

Config change (image updated):
  ├─ Agent "dev" starts session C
  ├─ resolveSandboxContext() → container cold, config hash MISMATCH
  └─ DELETE old container → CREATE new one with new image

25 hours of no activity:
  └─ Next resolveSandboxContext() call triggers prune → container removed
```

## What Goes Into a Container — Mounts & Workspace Seeding

### Mount Architecture

When a container is created (`src/agents/sandbox/docker.ts:334-371`), exactly **two bind mounts** are set up automatically, plus any user-configured extra `binds`:

```
Host filesystem                          Container filesystem
─────────────────────────────────────    ─────────────────────────────────────
$STATE_DIR/sandboxes/{scopeKey}/    →    /workspace        (workdir, main mount)
  ├── SOUL.md                                ├── SOUL.md
  ├── AGENTS.md                              ├── AGENTS.md
  ├── TOOLS.md                               ├── TOOLS.md
  ├── IDENTITY.md                            ├── IDENTITY.md
  ├── USER.md                                ├── USER.md
  ├── BOOTSTRAP.md                           ├── BOOTSTRAP.md
  ├── HEARTBEAT.md                           ├── HEARTBEAT.md
  ├── skills/                                ├── skills/
  └── (agent work files)                     └── (agent work files)

~/.openclaw/agents/{agentId}/       →    /agent            (agent workspace, secondary mount)
  ├── SOUL.md (original)                     ├── SOUL.md (original)
  ├── AGENTS.md (original)                   ├── AGENTS.md (original)
  └── ...                                    └── ...
```

Source: `src/agents/sandbox/docker.ts:352-362`, `src/agents/sandbox/constants.ts:48`

#### Mount #1: Sandbox Workspace → `/workspace`

```typescript
// docker.ts:355
args.push("-v", `${workspaceDir}:${cfg.workdir}${mainMountSuffix}`);
```

This is the **primary working directory** for the agent inside the container. It is a **copy** of the agent workspace (not the original), prepared on the host under `$STATE_DIR/sandboxes/{scopeKey}/`.

#### Mount #2: Agent Workspace → `/agent`

```typescript
// docker.ts:356-362
if (params.workspaceAccess !== "none" && workspaceDir !== params.agentWorkspaceDir) {
  args.push("-v", `${params.agentWorkspaceDir}:${SANDBOX_AGENT_WORKSPACE_MOUNT}${agentMountSuffix}`);
}
```

The original agent workspace is mounted at `/agent` as a **secondary mount**. This gives the container read access (or read-write, depending on `workspaceAccess`) to the original agent configuration files.

This mount is **only added** when:
- `workspaceAccess` is not `"none"`
- The sandbox workspace dir differs from the agent workspace dir (i.e., we're not in `rw` mode pointing at the same directory)

#### Mount #3+: Custom Binds

```typescript
// docker.ts:326-330
if (params.cfg.binds?.length) {
  for (const bind of params.cfg.binds) {
    args.push("-v", bind);
  }
}
```

Additional mounts from the `docker.binds` config array. These follow standard Docker bind-mount syntax (`/host/path:/container/path:ro`).

### Workspace Access Modes

The `workspaceAccess` setting (`src/agents/sandbox/types.ts:29`) controls mount permissions:

| Mode | `/workspace` mount | `/agent` mount | Effect |
|------|-------------------|----------------|--------|
| `"rw"` | Read-write | Read-write (or same dir) | Agent can modify host files directly |
| `"ro"` | Read-only | Read-only | Agent sees files but cannot change host |
| `"none"` | Read-write (isolated copy) | Not mounted | Agent is fully isolated from host workspace |

With `workspaceAccess: "rw"`, the sandbox workspace points directly at the agent workspace — both mounts collapse into one, and changes inside the container are immediately visible on the host.

### Workspace Seeding — How Agent Files Get Into the Container

When the sandbox workspace is first prepared (`src/agents/sandbox/workspace.ts:15-51`), these files are **copied** from the agent workspace:

| File | Purpose |
|------|---------|
| `AGENTS.md` | Agent registry / multi-agent definitions |
| `SOUL.md` | Agent personality and behavior instructions |
| `TOOLS.md` | Tool configuration and restrictions |
| `IDENTITY.md` | Agent identity (name, avatar, etc.) |
| `USER.md` | User-specific context |
| `BOOTSTRAP.md` | Bootstrap/setup instructions |
| `HEARTBEAT.md` | Heartbeat/cron behavior |

Source: `src/agents/sandbox/workspace.ts:23-31`, `src/agents/workspace.ts:23-29`

**Important behavior:**
- Files are copied with `flag: "wx"` (write-exclusive) — they are **only written if they don't already exist** in the sandbox workspace (`workspace.ts:40`)
- This means: on first creation, the sandbox gets seeded with the agent's personality files. On subsequent reuses (with `scope: "agent"`), the existing copies are preserved — even if the originals changed on the host.
- To force a refresh, delete the sandbox workspace or recreate the container (`openclaw sandbox recreate`).

### Skill Synchronization

After workspace seeding, skills are synced from the agent workspace to the sandbox workspace (`src/agents/sandbox/context.ts:50-58`):

```typescript
await syncSkillsToWorkspace({
  sourceWorkspaceDir: agentWorkspaceDir,
  targetWorkspaceDir: sandboxWorkspaceDir,
  config: params.config,
});
```

This only happens when `workspaceAccess !== "rw"` (in `rw` mode, the sandbox already points at the agent workspace, so skills are already available).

### FS Bridge — How the Gateway Reads/Writes Container Files

The gateway cannot directly access files inside the container's filesystem. Instead, it uses a **FS bridge** (`src/agents/sandbox/fs-bridge.ts:55-248`) that translates host paths to container paths and executes file operations via `docker exec`:

```
Gateway wants to read /workspace/output.txt
  → fsBridge.readFile({ filePath: "/workspace/output.txt" })
    → resolves host path via mount table
    → docker exec <container> sh -c 'cat -- "$1"' /workspace/output.txt
    → returns content
```

The bridge maintains a mount table (`src/agents/sandbox/fs-paths.ts:92-128`) built from:
1. The workspace mount (host sandbox dir → `/workspace`)
2. The agent mount (host agent dir → `/agent`)
3. Any custom `docker.binds` entries

Path resolution is bidirectional — the bridge can translate both host→container and container→host paths.

### Security Defaults

Containers are created with restrictive defaults (`src/agents/sandbox/docker.ts:237-331`):

| Setting | Value | Effect |
|---------|-------|--------|
| `--cap-drop ALL` | Drop all Linux capabilities | No privileged operations |
| `--security-opt no-new-privileges` | Prevent privilege escalation | Cannot gain new capabilities |
| `--network none` | No network access | Container is fully offline by default |
| `--tmpfs /tmp,/var/tmp,/run` | Temporary filesystems | Ephemeral scratch space |
| Env sanitization | Blocks sensitive vars | API keys, tokens filtered out |
| `sleep infinity` | Container entrypoint | Container stays alive, commands run via `docker exec` |

The container runs `sleep infinity` as its main process — all actual work happens via `docker exec` commands issued by the gateway.

## Agent Evolution vs. Sandbox Isolation

The OpenClaw workspace serves a dual purpose: it holds the agent's **personality files** (SOUL.md, IDENTITY.md, MEMORY.md) *and* acts as the **working directory** for code and file operations. In a sandbox, these two concerns create a tension between isolation and persistence.

### The Core Question: Should Agent Files Survive Sessions?

For evolving agents (agents that learn, update their own SOUL.md, accumulate memory), the answer is **yes**. But the default sandbox mode (`workspaceAccess: "none"`) creates an isolated copy — changes in the copy do not flow back to the host.

### Persistence Behavior by Configuration

```
                      ┌─────────────────────────────────────────────────┐
                      │           Agent Workspace (Host)                │
                      │     ~/.openclaw/agents/{agentId}/               │
                      │     SOUL.md, IDENTITY.md, MEMORY.md, ...       │
                      │         (the "source of truth")                 │
                      └──────────────┬──────────────────────────────────┘
                                     │
              ┌──────────────────────┼──────────────────────┐
              │                      │                      │
        workspaceAccess:        workspaceAccess:       workspaceAccess:
           "rw"                    "ro"                   "none" (default)
              │                      │                      │
    ┌─────────▼──────────┐ ┌────────▼──────────┐ ┌────────▼──────────┐
    │ /workspace = Host  │ │ /workspace = Copy  │ │ /workspace = Copy │
    │ (bind mount, rw)   │ │ (bind mount, ro)   │ │ (bind mount, rw)  │
    │                    │ │                    │ │                    │
    │ Changes to SOUL.md │ │ Cannot change any  │ │ Changes stay in   │
    │ persist on host ✓  │ │ files at all ✗     │ │ copy only ⚠       │
    │                    │ │                    │ │                    │
    │ /agent: not mounted│ │ /agent = Host (ro) │ │ /agent: not       │
    │ (same directory)   │ │                    │ │ mounted            │
    └────────────────────┘ └────────────────────┘ └────────────────────┘
```

Source: `src/agents/sandbox/context.ts:40`, `src/agents/sandbox/docker.ts:352-362`

### Semi-Persistence Trap with `workspaceAccess: "none"` + `scope: "agent"`

The default configuration creates a **semi-persistent** workspace that can be misleading:

| Event | What happens to agent files in sandbox copy |
|-------|---------------------------------------------|
| Session 1: Agent modifies SOUL.md | Change stored in sandbox copy on host (`$STATE_DIR/sandboxes/{scope}/`) |
| Session 2: Same agent starts | Same copy reused → change is still there |
| Container idle > 24h → auto-pruned | Container removed, but host copy directory **may** remain |
| `openclaw sandbox recreate` | Copy directory wiped, fresh copy from agent workspace |
| Host SOUL.md updated by admin | Sandbox copy is **not** updated (seeded with `flag: "wx"`, write-exclusive) |

This means evolution data in the sandbox copy can be **silently lost** on container recreation and is **invisible** to the host system.

### Recommended Configurations for Evolving Agents

| Scenario | `workspaceAccess` | `scope` | Why |
|----------|-------------------|---------|-----|
| Agent evolves (writes SOUL.md, MEMORY.md) | `"rw"` | `"agent"` | Changes persist on host; survive container recreation |
| Agent is read-only (personality is static) | `"none"` | `"agent"` or `"session"` | Full isolation; sandbox copy sufficient |
| Agent reads config but only writes code | `"ro"` | `"session"` | Can read SOUL.md etc. but cannot modify anything |
| Agent works on a shared codebase | `"rw"` | `"shared"` | All agents see each other's file changes |

**For b4forge agents that should evolve:** Use `workspaceAccess: "rw"`. This gives the agent direct read-write access to its host workspace directory. The container still provides **execution isolation** (all shell commands run inside the container with `--cap-drop ALL`, `--network none`, etc.) — the agent just happens to be working on the real files instead of a copy.

Source: `src/agents/sandbox/types.ts:29`, `src/agents/sandbox/config.ts:168`

## Building Sandbox Images

### Three Pre-built Images

OpenClaw ships three Dockerfiles and corresponding build scripts. They layer on top of each other:

```
debian:bookworm-slim (pinned hash)
  │
  ├─► Dockerfile.sandbox               → openclaw-sandbox:bookworm-slim
  │     bash, ca-certificates, curl, git, jq, python3, ripgrep
  │     User: sandbox (non-root)
  │     CMD: sleep infinity
  │     Build: scripts/sandbox-setup.sh
  │
  ├─► Dockerfile.sandbox-common        → openclaw-sandbox-common:bookworm-slim
  │     (extends sandbox base)
  │     + Node.js, npm, pnpm, Go, Rust, cargo, Bun, Homebrew, build-essential
  │     Build: scripts/sandbox-common-setup.sh
  │
  └─► Dockerfile.sandbox-browser       → openclaw-sandbox-browser:bookworm-slim
        (standalone, not based on sandbox)
        + Chromium, Xvfb, x11vnc, noVNC, websockify
        Ports: 9222 (CDP), 5900 (VNC), 6080 (noVNC)
        CMD: openclaw-sandbox-browser (entrypoint script)
        Build: scripts/sandbox-browser-setup.sh
```

Source: `Dockerfile.sandbox`, `Dockerfile.sandbox-common`, `Dockerfile.sandbox-browser`

### Build Commands

```bash
# 1. Base image (required first)
scripts/sandbox-setup.sh

# 2. Common dev tools (optional, extends base)
scripts/sandbox-common-setup.sh

# 3. Browser image (optional, standalone)
scripts/sandbox-browser-setup.sh
```

### Customizing the Common Image

`sandbox-common-setup.sh` is fully parameterized via environment variables:

| Variable | Default | Purpose |
|----------|---------|---------|
| `BASE_IMAGE` | `openclaw-sandbox:bookworm-slim` | Base image to extend |
| `TARGET_IMAGE` | `openclaw-sandbox-common:bookworm-slim` | Output image name |
| `PACKAGES` | `curl wget jq nodejs npm python3 git golang-go rustc cargo build-essential ...` | apt packages to install |
| `INSTALL_PNPM` | `1` | Install pnpm globally |
| `INSTALL_BUN` | `1` | Install Bun runtime |
| `INSTALL_BREW` | `1` | Install Homebrew |
| `FINAL_USER` | `sandbox` | User the container runs as |

Example: Build a minimal common image without Rust, Go, Bun, and Brew:

```bash
PACKAGES="curl jq nodejs npm python3 git build-essential" \
INSTALL_BUN=0 \
INSTALL_BREW=0 \
TARGET_IMAGE="my-sandbox:slim-dev" \
scripts/sandbox-common-setup.sh
```

Source: `scripts/sandbox-common-setup.sh`, `Dockerfile.sandbox-common`

### Building Custom Images

For agents with specific tool requirements, you can create your own image:

```dockerfile
FROM openclaw-sandbox:bookworm-slim

USER root
RUN apt-get update && apt-get install -y --no-install-recommends \
    your-custom-tool \
  && rm -rf /var/lib/apt/lists/*
USER sandbox
```

Key constraints for custom images:
- **Must support `sleep infinity`** — the container runs this as its main process; all work happens via `docker exec`
- **Non-root user recommended** — the base image creates a `sandbox` user; custom images should follow this pattern
- **No entrypoint needed** — the gateway passes the command directly (`sleep infinity`)
- **Pin the base image hash** — the shipped Dockerfiles pin `debian:bookworm-slim` by SHA for reproducibility

### Image Fallback Behavior

If the configured image doesn't exist (`src/agents/sandbox/docker.ts:178-189`):

| Image | Fallback |
|-------|----------|
| `openclaw-sandbox:bookworm-slim` (default) | Auto-pulls `debian:bookworm-slim` and tags it — works but has no tools installed |
| Any other image | **Error**: `Sandbox image not found: <name>. Build or pull it first.` |

This means the default image has a graceful fallback (bare Debian), but custom images must be pre-built.

### Alternative: `setupCommand` Instead of Custom Image

For lightweight customizations, you can skip building a custom image and use `setupCommand` instead:

```json
{
  "agents": {
    "list": [{
      "id": "dev",
      "sandbox": {
        "docker": {
          "image": "openclaw-sandbox:bookworm-slim",
          "network": "bridge",
          "setupCommand": "apt-get update && apt-get install -y --no-install-recommends your-tool"
        }
      }
    }]
  }
}
```

Pitfalls documented in `docs/gateway/sandboxing.md`:
- `network` must be set (default is `"none"` — no egress, so `apt-get` fails)
- `readOnlyRoot` must be `false` (default) for writes to succeed
- `user` must be root (omit `user` or set `"0:0"`)
- Runs **once** after container creation, not on every session start

Trade-off: `setupCommand` is simpler but slower (runs on first container creation) and less reproducible than baking a custom image. For production, prefer a custom Dockerfile.

## Key Takeaway for Multi-Agent Setup

For running all agents (including main) in containers:

1. Set `sandbox.mode: "all"` — sandboxes every session, including the main agent
2. Use `sandbox.scope: "agent"` for persistent workspaces (or `"session"` for isolation)
3. Give each agent its own template via `agents.list[N].sandbox.docker`
4. Containers are created at **run start**, not on demand — there is no lazy option
5. For **evolving agents**: use `workspaceAccess: "rw"` so personality file changes persist on the host
6. For **static agents**: use `workspaceAccess: "none"` (default) for maximum isolation

The container is ready before the LLM generates its first token. This avoids latency on the first `exec` call but means every agent run pays the container startup cost upfront.
