# OpenClaw Security & Sandbox

> **Cross-references**: `docs/gateway/security/index.md` (security overview), `docs/gateway/sandboxing.md` (sandbox deep-dive), `docs/gateway/sandbox-vs-tool-policy-vs-elevated.md` (three-layer control model), `docs/security/THREAT-MODEL-ATLAS.md` (MITRE ATLAS threat model), `docs/security/formal-verification.md` (TLA+ formal models), `docs/install/docker.md` (Docker deployment + sandbox setup)

## 1. Security Model – Overview

```
┌─────────────────────────────────────────────────┐
│              Security Layers                      │
├─────────────────────────────────────────────────┤
│ 1. DM Pairing      → Who is allowed to talk?     │
│ 2. Tool Policies    → What can the agent do?      │
│ 3. Sandbox          → Where does it execute code? │
│ 4. Network          → Where can it connect to?    │
│ 5. Bind Validation  → What can be mounted?        │
└─────────────────────────────────────────────────┘
```

---

## 2. DM Pairing

> **Source**: `src/security/dm-policy-shared.ts` — `resolveDmAllowState()` resolves wildcard/allowlist logic (line 5–40)
> **Docs**: `docs/gateway/security/index.md` § "DM access model (pairing / allowlist / open / disabled)"

### 2.1 Modes

The actual DM policy values supported are `pairing`, `allowlist`, `open`, and `disabled` (`docs/gateway/security/index.md:256–261`). The table below reflects the most common modes:

| Mode | Behavior |
|---|---|
| `pairing` (default) | Unknown senders receive a pairing code, must be approved |
| `open` | Public DMs accepted (requires `allowFrom: ["*"]`) |
| `disabled` | DMs completely disabled |

### 2.2 AllowFrom Configuration

```json
{
  "channels": {
    "telegram": {
      "accounts": [{
        "allowFrom": ["+4917xxx", "username123"],
        "dmPolicy": "pairing"
      }]
    }
  }
}
```

- `"*"` = Wildcard, all allowed (`src/security/dm-policy-shared.ts:19` — `hasWildcard` check)
- Specific users/numbers (normalized + deduplicated via `normalizeStringEntries`, `src/security/dm-policy-shared.ts:16–33`)
- Configurable per channel and account

---

## 3. Tool Policies

> **Source**: `src/agents/sandbox/tool-policy.ts` — `isToolAllowed()` (line 16), `resolveSandboxToolPolicyForAgent()` (line 35)
> **Constants**: `src/agents/sandbox/constants.ts` — `DEFAULT_TOOL_ALLOW` (line 13–27), `DEFAULT_TOOL_DENY` (line 30–37)
> **Docs**: `docs/gateway/sandbox-vs-tool-policy-vs-elevated.md` (three-layer control explanation)

### 3.1 Default Allow List

```
exec, process, read, write, edit, apply_patch, image,
sessions_list, sessions_history, sessions_send, sessions_spawn,
subagents, session_status
```

### 3.2 Default Deny List

```
browser, canvas, nodes, cron, gateway,
+ all channel provider IDs (telegram, whatsapp, discord, slack, ...)
```

### 3.3 Configuration per Agent

```json
{
  "agents": {
    "list": [{
      "id": "main",
      "tools": {
        "policy": "deny-by-default",
        "allowed": ["exec", "read", "write"],
        "denied": ["browser", "gateway"]
      }
    }]
  }
}
```

- Glob patterns allowed: `"sessions_*"` matches all session tools (`src/agents/sandbox/tool-policy.ts:18` uses `compileGlobPatterns`)
- Resolution hierarchy: Agent-specific > Global > Default (`src/agents/sandbox/tool-policy.ts:45–84`)
- Note: the config keys shown above (`policy`, `allowed`, `denied`) are simplified; the actual config keys are `tools.sandbox.tools.allow` / `tools.sandbox.tools.deny` and `agents.list[].tools.sandbox.tools.allow` / `agents.list[].tools.sandbox.tools.deny` (`docs/gateway/sandbox-vs-tool-policy-vs-elevated.md:60`)

---

## 4. Sandbox System

> **Source**: `src/agents/sandbox/` (main directory), `Dockerfile.sandbox`, `Dockerfile.sandbox-common`, `Dockerfile.sandbox-browser`
> **Config resolution**: `src/agents/sandbox/config.ts` — `resolveSandboxConfigForAgent()` (line 145)
> **Docker types**: `src/agents/sandbox/types.docker.ts` (full `SandboxDockerConfig` shape)
> **Docs**: `docs/gateway/sandboxing.md` (comprehensive sandbox docs), `docs/install/docker.md` § "Agent Sandbox"

### 4.1 Architecture

```
Gateway
  │
  │  Agent wants to execute code
  │
  ▼
┌──────────────────────────────────┐
│  Sandbox Manager                  │
│                                   │
│  1. Create/find container         │
│  2. Command via docker exec       │
│  3. Read back result              │
│  4. Manage container pool         │
└──────────────────────────────────┘
  │
  ▼
┌──────────────────────────────────┐
│  Docker Container                 │
│  debian:bookworm-slim             │
│  User: sandbox (non-root)         │
│  Network: none (default)          │
│  Capabilities: ALL dropped        │
│  Read-only root FS                │
└──────────────────────────────────┘
```

### 4.2 Container Runtime

**Only Docker is supported** (`src/agents/sandbox/docker.ts:33` — `spawn("docker", args, ...)`).

Podman is supported for **hosting OpenClaw itself** (`setup-podman.sh`), but not as a sandbox runtime. See also `docs/install/podman.md`.

### 4.3 Sandbox Modes

| Mode (`sandbox.mode`) | Behavior |
|---|---|
| `off` (default) | No sandbox |
| `non-main` | Only sandbox non-main sessions |
| `all` | Sandbox all sessions |

> Default: `"off"` (`src/agents/sandbox/config.ts:166`). Docs: `docs/gateway/sandboxing.md:37–43`.

### 4.4 Sandbox Scope

| Scope (`sandbox.scope`) | Behavior |
|---|---|
| `session` | Dedicated container per session |
| `agent` (default) | Shared container per agent |
| `shared` | One container for all agents |

> Default: `"agent"` (`src/agents/sandbox/config.ts:51`). Docs: `docs/gateway/sandboxing.md:47–51`. Type: `SandboxScope` in `src/agents/sandbox/types.ts:51`.

### 4.5 Docker Configuration

```json
{
  "agents": {
    "list": [{
      "id": "main",
      "sandbox": {
        "mode": "all",
        "scope": "session",
        "workspaceAccess": "ro",
        "docker": {
          "image": "openclaw-sandbox:bookworm-slim",
          "containerPrefix": "openclaw-sbx-",
          "workdir": "/workspace",
          "readOnlyRoot": true,
          "tmpfs": ["/tmp", "/var/tmp", "/run"],
          "network": "none",
          "capDrop": ["ALL"],
          "pidsLimit": 256,
          "memory": "512m",
          "cpus": "1.0",
          "env": { "MY_VAR": "value" },
          "setupCommand": "apt-get update && apt-get install -y python3"
        },
        "prune": {
          "idleHours": 24,
          "maxAgeDays": 7
        }
      }
    }]
  }
}
```

### 4.6 Sandbox Images

| Dockerfile | Contents | Use Case |
|---|---|---|
| `Dockerfile.sandbox` | Minimal: bash, curl, git, jq, python3, rg | Standard sandbox |
| `Dockerfile.sandbox-common` | + Node.js, Go, Rust, Bun, pnpm, Brew | Developer sandbox |
| `Dockerfile.sandbox-browser` | + Chromium, Xvfb, VNC, noVNC | Browser sandbox |

### 4.7 Hot Window & Container Reuse

- Recently used containers (< 5 min) are reused (`src/agents/sandbox/docker.ts:117` — `HOT_CONTAINER_WINDOW_MS = 5 * 60 * 1000`)
- Container registry path: `src/agents/sandbox/constants.ts:51` — `SANDBOX_REGISTRY_PATH` (resolves to `<STATE_DIR>/sandbox/containers.json`)
- Registry I/O: `src/agents/sandbox/registry.ts` — `readRegistry()` (line 130), `updateRegistry()` (line 164)
- Config hash comparison: `src/agents/sandbox/docker.ts:398–439` — if hash mismatches on a hot container, log warning instead of recreating
- Auto-prune after `idleHours` / `maxAgeDays`: `src/agents/sandbox/prune.ts:22–34` — `shouldPruneSandboxEntry()`
- Default prune values: `idleHours: 24`, `maxAgeDays: 7` (`src/agents/sandbox/constants.ts:10–11`)

---

## 5. Security Validation

> **Source**: `src/agents/sandbox/validate-sandbox-security.ts` — `BLOCKED_HOST_PATHS` (line 13–28), `validateSandboxSecurity()` (line 185–195)
> **Called from**: `src/agents/sandbox/docker.ts:246` — `buildSandboxCreateArgs()` calls `validateSandboxSecurity()` at runtime

### 5.1 Blocked Host Paths

The following paths must **NOT** be mounted into containers:

```
/etc, /private/etc, /proc, /sys, /dev, /root, /boot,
/run, /var/run, /private/var/run,
/var/run/docker.sock, /private/var/run/docker.sock, /run/docker.sock
```

### 5.2 This Means:

- **No Docker-in-Docker** (Docker socket is blocked — `src/agents/sandbox/validate-sandbox-security.ts:27`)
- **No Podman-in-Docker**
- **No access to host system directories**
- Symlinks are resolved (no bypassing via symlinks — `tryRealpathAbsolute()` at `src/agents/sandbox/validate-sandbox-security.ts:91–104`)
- Only absolute paths allowed (`getBlockedBindReason()` at `src/agents/sandbox/validate-sandbox-security.ts:68–76`)
- Additionally validates: network mode (blocks `host`, line 155), seccomp profile (blocks `unconfined`, line 165), apparmor profile (blocks `unconfined`, line 175)

---

## 6. Container-in-Container: Status

### 6.1 Current Situation

```
Kubernetes Cluster / Docker Host
  │
  ▼
┌──────────────────────────────┐
│ OpenClaw Container            │  ← OpenClaw runs here
│ (Gateway + Agents)            │
│                               │
│  sandbox.mode = "all"         │
│  → Tries docker exec ...      │
│  → FAILS: no Docker           │
│    socket in container!        │
└──────────────────────────────┘
```

**Problem**: When OpenClaw itself runs in a container, it has no access to the host's Docker daemon. Docker-in-Docker is explicitly blocked.

### 6.2 Possible Workarounds

| Approach | Risk | Feasibility |
|---|---|---|
| Mount Docker socket | HIGH (host access) | Blocked by OpenClaw |
| DinD sidecar | Medium | Technically possible, but not officially supported |
| Disable sandbox (`mode: off`) | Medium (no code sandboxing) | Simple, but insecure |
| Write custom sandbox provider | Low | Complex (see 7.) |

---

## 7. Kubernetes – Current Status

### 7.1 Is There Kubernetes Support?

**No. None.** No Kubernetes API client, no pod manifests, no kubectl, no Helm chart, no CRDs. Searching for "kubernetes", "k8s", "pod", "kubectl" in the entire codebase yields nothing.

### 7.2 What Would Be Needed for Kubernetes Pods as Sandbox?

A new sandbox provider would need to be implemented:

```
Current:                          Target:
src/agents/sandbox/               src/agents/sandbox/
├── docker.ts      ← only        ├── docker.ts
├── constants.ts     runtime      ├── kubernetes.ts   ← NEW
├── container-*                   ├── runtime.ts      ← Abstraction
└── validate-*                    └── validate-*
```

**Required Components**:

1. `kubernetes.ts` – Kubernetes API client (`@kubernetes/client-node`)
2. Pod manifest generation (analogous to Docker `create` args)
3. Exec via Kubernetes API (analogous to `docker exec`)
4. Mapping of security options:
   - `capDrop: ["ALL"]` → `securityContext.capabilities.drop: ["ALL"]`
   - `readOnlyRoot: true` → `securityContext.readOnlyRootFilesystem: true`
   - `network: "none"` → NetworkPolicy
   - `memory/cpus` → Resource Limits
5. Volume handling (PVC or emptyDir instead of Docker binds)
6. RBAC for the OpenClaw ServiceAccount
7. Extend config schema: `sandbox.kubernetes.*`

### 7.3 Challenges

| Topic | Docker | Kubernetes |
|---|---|---|
| **Startup Latency** | ~100ms | ~5-30s (pod scheduling) |
| **Exec** | `docker exec` (instant) | `kubectl exec` via API (slower) |
| **Networking** | `--network none` | NetworkPolicy (more complex) |
| **Volume Mounts** | Bind mounts | PVCs / emptyDir |
| **Cleanup** | `docker rm` | Pod deletion + GC |
| **Cost** | Low (local containers) | Higher (pod = schedulable unit) |

### 7.4 Alternative Approach: OpenClaw as Kubernetes Deployment

Instead of spawning pods as sandbox, one could run OpenClaw as a regular Kubernetes deployment and disable the sandbox:

```yaml
# openclaw-deployment.yaml (conceptual)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: openclaw-gateway
spec:
  replicas: 1
  template:
    spec:
      containers:
        - name: openclaw
          image: openclaw:latest
          env:
            - name: OPENCLAW_GATEWAY_TOKEN
              valueFrom:
                secretKeyRef: ...
          volumeMounts:
            - name: data
              mountPath: /home/node/.openclaw
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: openclaw-data
```

With `sandbox.mode: "off"` – the agent executes code directly in the container. Security then via:
- Tool policies (what the agent is allowed to do)
- Kubernetes NetworkPolicies (where it can connect to)
- Read-only filesystem + SecurityContext
- Resource limits at pod level

---

## 8. Deployment Options

| Method | Script/Config | Description | Docs |
|---|---|---|---|
| **Docker Compose** | `docker-compose.yml`, `docker-setup.sh` | Standard deployment | `docs/install/docker.md` |
| **Podman (rootless)** | `setup-podman.sh` | Rootless with systemd Quadlet | `docs/install/podman.md` |
| **Fly.io** | `fly.toml` | Cloud deployment | `docs/install/fly.md` |
| **Bare Metal** | `openclaw gateway` | Directly on the host | `docs/install/node.md` |

### 8.1 Docker Compose Reference

```yaml
# docker-compose.yml (simplified)
openclaw-gateway:
  image: openclaw:local
  volumes:
    - ${OPENCLAW_CONFIG_DIR}:/home/node/.openclaw
    - ${OPENCLAW_WORKSPACE_DIR}:/home/node/.openclaw/workspace
  ports:
    - "18789:18789"   # Gateway
    - "18790:18790"   # Bridge
  environment:
    - OPENCLAW_GATEWAY_TOKEN=...

openclaw-cli:
  image: openclaw:local
  # Interactive for onboarding
```

---

## 9. Best Practices

### 9.1 Security Checklist

- [ ] DM pairing enabled (default) – no `allowFrom: ["*"]` in production
- [ ] Tool policies defined per agent – only allow required tools
- [ ] Sandbox enabled for code execution (`sandbox.mode: "all"` or `"non-main"`)
- [ ] Network disabled in sandbox (`network: "none"`) if possible
- [ ] Capabilities dropped (`capDrop: ["ALL"]`)
- [ ] Read-only root filesystem (`readOnlyRoot: true`)
- [ ] Resource limits set (`memory`, `cpus`, `pidsLimit`)
- [ ] No sensitive env vars in sandbox (`env` is sanitized — `src/agents/sandbox/sanitize-env-vars.ts:1–19` blocks API keys, tokens, passwords)
- [ ] Gateway token strong and secret (`OPENCLAW_GATEWAY_TOKEN`) — see `docs/gateway/security/index.md` § "Lock down the Gateway WebSocket"
- [ ] Auth profiles stored securely (not in Git!) — credential paths listed in `docs/gateway/security/index.md` § "Credential storage map"

### 9.2 For Container Deployment

- [ ] Non-root user in container
- [ ] Only expose required ports (18789)
- [ ] Persistent volume for `/home/node/.openclaw`
- [ ] Sandbox `mode: off` when no Docker socket is available
- [ ] Instead: Configure tool policies restrictively

---

## References

### Source Files

| File | Purpose |
|---|---|
| `src/security/dm-policy-shared.ts` | DM allowlist/wildcard resolution |
| `src/agents/sandbox/tool-policy.ts` | Sandbox tool allow/deny resolution with glob support |
| `src/agents/sandbox/constants.ts` | Default allow/deny lists, image names, prune defaults |
| `src/agents/sandbox/types.ts` | `SandboxConfig`, `SandboxScope`, `SandboxToolPolicy` types |
| `src/agents/sandbox/types.docker.ts` | `SandboxDockerConfig` type (full Docker options shape) |
| `src/agents/sandbox/config.ts` | `resolveSandboxConfigForAgent()` — merges global + per-agent config |
| `src/agents/sandbox/docker.ts` | Docker runtime: `spawn("docker", ...)`, container create/start, hot window logic |
| `src/agents/sandbox/validate-sandbox-security.ts` | Blocked host paths, bind mount validation, network/seccomp/apparmor checks |
| `src/agents/sandbox/sanitize-env-vars.ts` | Blocks sensitive env vars (API keys, tokens, passwords) from sandbox containers |
| `src/agents/sandbox/registry.ts` | Container registry read/write (`containers.json`) |
| `src/agents/sandbox/prune.ts` | Auto-prune idle/old sandbox containers |
| `src/agents/sandbox/manage.ts` | Container listing + removal (used by CLI) |
| `Dockerfile.sandbox` | Minimal sandbox image |
| `Dockerfile.sandbox-common` | Developer sandbox image (Node, Go, Rust, Bun) |
| `Dockerfile.sandbox-browser` | Browser sandbox image (Chromium, Xvfb, noVNC) |
| `docker-compose.yml` | Docker Compose deployment config |
| `docker-setup.sh` | Docker setup script |
| `setup-podman.sh` | Podman rootless setup script |
| `fly.toml` | Fly.io deployment config |

### Official Documentation

| Doc | Topic |
|---|---|
| `docs/gateway/security/index.md` | Security overview, audit, DM access model, credential map, hardening |
| `docs/gateway/sandboxing.md` | Sandbox modes, scopes, workspace access, images, bind mounts, setup |
| `docs/gateway/sandbox-vs-tool-policy-vs-elevated.md` | Three-layer control model (sandbox vs tool policy vs elevated) |
| `docs/security/README.md` | Security landing page, vulnerability reporting |
| `docs/security/THREAT-MODEL-ATLAS.md` | MITRE ATLAS threat model, attack chains, risk matrix |
| `docs/security/formal-verification.md` | TLA+ formal verification models |
| `docs/install/docker.md` | Docker deployment + agent sandbox setup |
| `docs/install/podman.md` | Podman deployment |
| `docs/install/fly.md` | Fly.io deployment |
| `docs/tools/exec-approvals.md` | Exec approval logic |
| `docs/tools/multi-agent-sandbox-tools.md` | Per-agent sandbox + tool overrides |
| `docs/tools/elevated.md` | Elevated exec escape hatch |
