# Deployment Best Practices

Production deployment guide for OpenClaw. Covers configuration management, backup, monitoring, Docker hardening, Kubernetes, and upgrade strategies.

---

## 1. Configuration Management

### Config File and Format

OpenClaw reads a single **JSON5** config from `~/.openclaw/openclaw.json`. JSON5 allows comments and trailing commas. All fields are optional; safe defaults are used when omitted (`docs/gateway/configuration.md:10-11`).

Strict validation: unknown keys, malformed types, or invalid values cause the Gateway to **refuse to start** (`docs/gateway/configuration.md:64-66`). Run `openclaw doctor` to diagnose.

### Environment Variables

Env vars are loaded from three sources with this precedence (`docs/gateway/configuration.md:425-435`):

1. Parent process environment
2. `.env` in the current working directory
3. `~/.openclaw/.env` (global fallback)

Neither `.env` file overrides existing vars. For daemon-managed processes (systemd/launchd), prefer `~/.openclaw/.env` (`docs/gateway/authentication.md:31-37`).

Inline env vars can also be set in config:

```json5
{
  env: {
    OPENROUTER_API_KEY: "sk-or-...",
    vars: { GROQ_API_KEY: "gsk-..." },
  },
}
```

### Env Var Substitution in Config

Reference env vars in any config string value with `${VAR_NAME}`. Only uppercase names are matched. Missing or empty vars throw an error at load time (`docs/gateway/configuration-reference.md:2236-2249`).

```json5
{
  gateway: { auth: { token: "${OPENCLAW_GATEWAY_TOKEN}" } },
}
```

### Splitting Config with `$include`

Large configs can be split into multiple files using `$include` (`docs/gateway/configuration.md:307-328`):

```json5
{
  gateway: { port: 18789 },
  agents: { $include: "./agents.json5" },
  broadcast: { $include: ["./clients/a.json5", "./clients/b.json5"] },
}
```

- Single file: replaces the containing object.
- Array of files: deep-merged in order (later wins).
- Nested includes supported up to 10 levels deep.
- Paths resolved relative to the including file.

### Config Hot Reload

The Gateway watches `~/.openclaw/openclaw.json` and applies changes automatically (`docs/gateway/configuration.md:333-369`). Four reload modes:

| Mode | Behavior |
|---|---|
| `hybrid` (default) | Hot-applies safe changes; auto-restarts for critical ones |
| `hot` | Hot-applies safe changes only; logs warning when restart needed |
| `restart` | Restarts on any change |
| `off` | Manual restart only |

Most fields hot-apply (channels, agents, models, tools, sessions). **Gateway server changes** (`gateway.*`: port, bind, auth, TLS) require a restart.

### Per-Agent Configuration

Multi-agent routing allows separate workspaces, models, sandbox profiles, and tool policies per agent (`docs/gateway/configuration-reference.md:972-1012`):

```json5
{
  agents: {
    list: [
      { id: "home", default: true, workspace: "~/.openclaw/workspace-home" },
      { id: "work", workspace: "~/.openclaw/workspace-work" },
    ],
  },
  bindings: [
    { agentId: "home", match: { channel: "whatsapp", accountId: "personal" } },
    { agentId: "work", match: { channel: "whatsapp", accountId: "biz" } },
  ],
}
```

### Secrets Management

- API keys: store in `~/.openclaw/.env` or use env var substitution (`${VAR_NAME}`) in config (`docs/gateway/authentication.md:31-37`).
- Gateway token: `OPENCLAW_GATEWAY_TOKEN` or `gateway.auth.token` in config.
- API key rotation: OpenClaw supports multiple keys per provider via `<PROVIDER>_API_KEYS` and retries on rate-limit errors (`docs/gateway/authentication.md:109-121`).
- Auth profiles: per-agent profiles stored at `<agentDir>/auth-profiles.json` (`docs/gateway/configuration-reference.md:2256-2271`).
- For Kubernetes: use HashiCorp Vault or Kubernetes Secrets with encryption at rest (community Helm chart pattern).

### Multi-Instance Isolation

Run multiple gateways on one host with unique ports and state dirs (`docs/gateway/configuration-reference.md:2050-2058`):

```bash
OPENCLAW_CONFIG_PATH=~/.openclaw/a.json \
OPENCLAW_STATE_DIR=~/.openclaw-a \
openclaw gateway --port 19001
```

---

## 2. Backup Strategies

### What to Back Up

All long-lived state lives under `~/.openclaw/` (or `$OPENCLAW_STATE_DIR`). The Hetzner guide documents the persistence model (`docs/install/hetzner.md:312-328`):

| Component | Location | Notes |
|---|---|---|
| Gateway config | `~/.openclaw/openclaw.json` | Core settings, model config, channel bindings |
| Credentials | `~/.openclaw/credentials/` | API keys, OAuth tokens, WhatsApp session |
| Session transcripts | `~/.openclaw/agents/<agentId>/sessions/` | JSONL message history |
| Auth profiles | `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` | Per-agent model auth |
| Agent workspace | `~/.openclaw/workspace/` | Code, HEARTBEAT.md, IDENTITY.md, skills |
| Memory index | `~/.openclaw/` area | Vector and full-text search database |
| Skills config | `~/.openclaw/skills/` | Skill-level state |

### Backup Approaches

**Manual backup**: copy the entire `~/.openclaw/` directory.

```bash
tar czf openclaw-backup-$(date +%Y%m%d-%H%M%S).tar.gz ~/.openclaw/
```

**Automated backup** via cron:

```bash
# Daily backup, keep 7 days
0 3 * * * tar czf /backups/openclaw-$(date +\%F).tar.gz ~/.openclaw/ && find /backups/ -name 'openclaw-*.tar.gz' -mtime +7 -delete
```

**Docker volume backup**: for the containerized gateway, back up the host bind mounts (`docs/install/docker.md:63-64`):

- `~/.openclaw/` (config + state)
- `~/.openclaw/workspace` (agent workspace)

**VPS snapshot**: on Hetzner/DigitalOcean, use provider-level VPS snapshots as an additional safety net.

**Infrastructure-as-Code**: community Terraform modules provide automated backup/restore scripts (`docs/install/hetzner.md:331-348`).

### Restoration

1. Copy backed-up state directory to `~/.openclaw/`.
2. Run `openclaw doctor` to detect and apply any needed migrations.
3. Restart the Gateway.

The `doctor` command creates `.old` backups before moving files and is idempotent (safe to run multiple times).

### Security Considerations for Backups

Backups contain sensitive data (API keys, OAuth tokens, gateway tokens, WhatsApp sessions). Encrypt backups when storing off-machine. Enforce file permissions: `chmod 600` for `openclaw.json` and auth files; `chmod 700` for the credentials directory.

---

## 3. Monitoring and Observability

### Health Checks

**CLI health** (`docs/cli/health.md`):

```bash
openclaw health           # Quick health snapshot
openclaw health --json    # JSON output for scripting
openclaw health --verbose # Per-account live probes
```

**Status commands** (`docs/gateway/health.md`):

```bash
openclaw status           # Local summary
openclaw status --all     # Full local diagnosis
openclaw status --deep    # Probes running Gateway
```

**Docker health check** (`docs/install/docker.md:307-308`):

```bash
docker compose exec openclaw-gateway node dist/index.js health --token "$OPENCLAW_GATEWAY_TOKEN"
```

**In-chat health**: send `/status` as a message in WhatsApp/WebChat to get a status reply without invoking the agent (`docs/gateway/health.md:18`).

The health state is managed in `src/gateway/server/health-state.ts`, which provides a cached health snapshot that is refreshed and broadcast to connected clients.

### Logging

Logs are written in two places (`docs/logging.md:14-17`):

1. **File logs** (JSONL): `/tmp/openclaw/openclaw-YYYY-MM-DD.log` (configurable via `logging.file`)
2. **Console output**: TTY-aware, colorized

Configuration (`docs/logging.md:99-113`):

```json5
{
  logging: {
    level: "info",
    file: "/var/log/openclaw/openclaw.log", // stable path for production
    consoleLevel: "info",
    consoleStyle: "json", // json for log processors
    redactSensitive: "tools",
    redactPatterns: ["sk-.*"],
  },
}
```

Tail logs via CLI: `openclaw logs --follow` (works remotely over Gateway WS).

### Diagnostics Flags

Targeted debug logs without raising global log levels (`docs/diagnostics/flags.md`):

```json5
{
  diagnostics: {
    flags: ["telegram.http", "gateway.*"],
  },
}
```

Env override: `OPENCLAW_DIAGNOSTICS=telegram.http,telegram.payload`

### OpenTelemetry Integration

Full OTel export via the `diagnostics-otel` plugin (`docs/logging.md:222-264`):

```json5
{
  plugins: {
    allow: ["diagnostics-otel"],
    entries: { "diagnostics-otel": { enabled: true } },
  },
  diagnostics: {
    enabled: true,
    otel: {
      enabled: true,
      endpoint: "http://otel-collector:4318",
      protocol: "http/protobuf",
      serviceName: "openclaw-gateway",
      traces: true,
      metrics: true,
      logs: true,
      sampleRate: 0.2,
      flushIntervalMs: 60000,
    },
  },
}
```

**Exported metrics** (`docs/logging.md:267-304`):

| Metric | Type | Description |
|---|---|---|
| `openclaw.tokens` | counter | Token usage per provider/model/channel |
| `openclaw.cost.usd` | counter | Cost per provider/model |
| `openclaw.run.duration_ms` | histogram | Agent run duration |
| `openclaw.context.tokens` | histogram | Context window usage |
| `openclaw.webhook.received` | counter | Webhook ingress per channel |
| `openclaw.webhook.duration_ms` | histogram | Webhook processing time |
| `openclaw.message.queued` | counter | Messages enqueued |
| `openclaw.message.duration_ms` | histogram | Message processing time |
| `openclaw.queue.depth` | histogram | Queue depth per lane |
| `openclaw.session.state` | counter | Session state transitions |
| `openclaw.session.stuck` | counter | Stuck session warnings |

**Exported spans** (`docs/logging.md:306-324`):

- `openclaw.model.usage` (tokens, provider, model, session)
- `openclaw.webhook.processed` / `openclaw.webhook.error`
- `openclaw.message.processed`
- `openclaw.session.stuck`

**Diagnostic events** (`docs/logging.md:163-183`):

- `model.usage`, `webhook.received`, `webhook.processed`, `webhook.error`
- `message.queued`, `message.processed`
- `queue.lane.enqueue`, `queue.lane.dequeue`
- `session.state`, `session.stuck`, `run.attempt`, `diagnostic.heartbeat`

### Auth Monitoring

Automation-friendly auth check (`docs/gateway/authentication.md:88-89`):

```bash
openclaw models status --check  # exit 1 = expired/missing, exit 2 = expiring
```

---

## 4. Docker / Container Best Practices

### Security Hardening

The default Dockerfile runs as non-root `node` user (uid 1000) (`Dockerfile:49-52`):

```dockerfile
USER node
```

Key hardening points from the official Docker guide (`docs/install/docker.md:187-238`):

- Default image is **security-first**: no system package installs at runtime, no root access.
- Bind mount ownership must match uid 1000 on the host:
  ```bash
  sudo chown -R 1000:1000 /path/to/openclaw-config /path/to/openclaw-workspace
  ```

### Sandbox Container Hardening

Agent sandbox containers support extensive hardening (`docs/install/docker.md:383-446`):

```json5
{
  agents: {
    defaults: {
      sandbox: {
        docker: {
          readOnlyRoot: true,
          tmpfs: ["/tmp", "/var/tmp", "/run"],
          network: "none",            // no egress by default
          user: "1000:1000",
          capDrop: ["ALL"],
          pidsLimit: 256,
          memory: "1g",
          memorySwap: "2g",
          cpus: 1,
          ulimits: {
            nofile: { soft: 1024, hard: 2048 },
            nproc: 256,
          },
          seccompProfile: "/path/to/seccomp.json",
          apparmorProfile: "openclaw-sandbox",
        },
      },
    },
  },
}
```

### Persistence

The `docker-compose.yml` mounts two host directories (`docker-compose.yml:12-13`):

```yaml
volumes:
  - ${OPENCLAW_CONFIG_DIR}:/home/node/.openclaw
  - ${OPENCLAW_WORKSPACE_DIR}:/home/node/.openclaw/workspace
```

Optional: persist the entire `/home/node` with a named Docker volume via `OPENCLAW_HOME_VOLUME` (`docs/install/docker.md:137-163`).

### Restart Policy

Use `restart: unless-stopped` in docker-compose (`docker-compose.yml:17`). The `--allow-unconfigured` flag is for bootstrap only; always set proper auth for production.

### Bake Dependencies at Build Time

All binaries needed by skills must be installed during image build, not at runtime (`docs/install/hetzner.md:197-218`). Runtime-installed packages are lost on container restart.

```dockerfile
ARG OPENCLAW_DOCKER_APT_PACKAGES="ffmpeg build-essential"
```

### Network Binding

- Default bind is `loopback` for security (`Dockerfile:56-57`).
- Docker Compose overrides to `lan` for container use (`docker-compose.yml:25`).
- For VPS: bind to `127.0.0.1` on the host and access via SSH tunnel (`docs/install/hetzner.md:176-179`):
  ```yaml
  ports:
    - "127.0.0.1:${OPENCLAW_GATEWAY_PORT}:18789"
  ```

### Init Process

Always use `init: true` in Docker Compose to ensure proper signal handling and zombie process reaping (`docker-compose.yml:17`).

---

## 5. Kubernetes / Helm Production Tips

### Community Helm Chart

The primary community Helm chart is [serhanekicii/openclaw-helm](https://github.com/serhanekicii/openclaw-helm) (referenced in `exploration/external-resources.md:38`).

### Architecture Constraints

OpenClaw is a **single-instance** application. It does not support horizontal scaling. One Gateway owns channel connections (especially WhatsApp Web sessions) and state. Run exactly one replica (`docs/gateway/network-model.md:13`).

### Resource Limits

Recommended starting points (from the community Helm chart):

| Container | CPU Request | CPU Limit | Memory Request | Memory Limit |
|---|---|---|---|---|
| Gateway | 200m | 2000m | 512Mi | 2Gi |
| Chromium sidecar | 100m | 1000m | 256Mi | 1Gi |

### Persistent Volumes

- Use a `PersistentVolumeClaim` with `ReadWriteOnce` (5-10Gi) for `~/.openclaw/`.
- This stores workspace, sessions, runtime state, and configuration.
- Use `emptyDir` for temporary files.

### Security Context

Enforce non-root execution:

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  fsGroup: 1000
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
```

### Config Management Modes

Two approaches (`docs/gateway/configuration.md` and community Helm patterns):

1. **Merge mode** (default): Helm values deep-merge with existing PVC config, preserving runtime changes from the Control UI. Good for hands-on operators.
2. **Overwrite mode**: strict GitOps; Helm values are the sole source of truth.

For ArgoCD with merge mode, configure `ignoreDifferences` on the ConfigMap to prevent sync conflicts.

### Network Policies

When enabled (requires Calico/Cilium CNI):

- Ingress: allow from ingress controller namespace on port 18789.
- Egress: allow kube-dns (DNS resolution) and public internet IPs.
- Block RFC1918, link-local, and CGN ranges to contain blast radius.

### Secrets in Kubernetes

- Preferred: HashiCorp Vault / OpenBao with Vault Secrets Operator syncing to Kubernetes Secrets.
- Alternative: native Kubernetes Secrets with encryption at rest.
- Required secrets: `ANTHROPIC_API_KEY`, `OPENCLAW_GATEWAY_TOKEN`, channel tokens.

### Device Pairing in K8s

After pod startup:

```bash
kubectl exec <pod> -- node dist/index.js devices approve <REQUEST_ID>
```

### Kubernetes Operator

A community Kubernetes operator is available at [openclaw.rocks](https://openclaw.rocks/blog/openclaw-kubernetes-operator) (referenced in `exploration/external-resources.md`), providing CRD-based management with storage persistence and sidecar support.

---

## 6. Upgrade Strategies

### Standard Upgrade (npm)

```bash
npm update -g openclaw
openclaw doctor          # Detect and apply migrations
openclaw gateway         # Restart
```

### Docker Upgrade

1. Pull or build the new image.
2. Run `docker compose down`.
3. Run `docker compose up -d`.
4. The mounted `~/.openclaw/` state persists across container recreation (`docs/install/hetzner.md:312-328`).

### Pre-Upgrade Safety

- **Always back up** `~/.openclaw/` before upgrading.
- `openclaw doctor --fix` creates a backup at `~/.openclaw/backups/doctor-<timestamp>/` before making changes.
- Feature request exists for auto-backup before any config modification ([GitHub #6477](https://github.com/openclaw/openclaw/issues/6477)).

### Doctor Command

`openclaw doctor` handles (`docs/gateway/configuration.md:66-71`):

- Legacy path migration (old session directories to agent-scoped locations).
- Config key remapping (deprecated keys to current schema).
- Workspace detection and consolidation.
- Schema validation with repair suggestions.

The command is idempotent and safe to run multiple times.

### Kubernetes Upgrade

With Helm:

```bash
helm upgrade openclaw openclaw-helm/openclaw -f values.yaml
```

The PVC persists across pod restarts. In merge mode, runtime config changes survive upgrades. In overwrite mode, Helm values are reapplied.

### Rollback

- Docker: revert to previous image tag + restore `~/.openclaw/` from backup.
- Kubernetes: `helm rollback openclaw <revision>`.
- VPS: restore from provider snapshot.

---

## References

### Official Documentation (in-repo)

- `docs/gateway/configuration.md` - Configuration overview, hot reload, env vars
- `docs/gateway/configuration-reference.md` - Complete field-by-field reference
- `docs/gateway/authentication.md` - Model auth, API keys, OAuth, setup-token
- `docs/gateway/health.md` - Health check CLI and diagnostics
- `docs/gateway/heartbeat.md` - Heartbeat polling and notification rules
- `docs/gateway/network-model.md` - Gateway networking model
- `docs/gateway/remote.md` - Remote access via SSH/Tailscale
- `docs/install/docker.md` - Docker setup, sandboxing, hardening
- `docs/install/hetzner.md` - Production VPS guide with persistence model
- `docs/vps.md` - VPS hosting overview
- `docs/logging.md` - Logging, diagnostics, OpenTelemetry integration
- `docs/diagnostics/flags.md` - Targeted diagnostic flags
- `docs/ci.md` - CI pipeline overview
- `docs/security/README.md` - Security and trust
- `docs/security/THREAT-MODEL-ATLAS.md` - MITRE ATLAS threat model

### Source Files

- `src/gateway/server/health-state.ts` - Health snapshot cache and broadcast
- `src/commands/health.ts` - Health snapshot generation
- `Dockerfile` - Official container image (non-root, bookworm base)
- `docker-compose.yml` - Default Compose configuration

### External Resources

- [openclaw-helm (Helm chart)](https://github.com/serhanekicii/openclaw-helm) - Community Helm chart
- [openclaw-terraform-hetzner](https://github.com/andreesg/openclaw-terraform-hetzner) - Terraform IaC for Hetzner
- [OpenClaw.rocks Kubernetes Operator](https://openclaw.rocks/blog/openclaw-kubernetes-operator) - CRD-based operator
- [DeepWiki: Migration and Backup](https://deepwiki.com/openclaw/openclaw/14.4-migration-and-backup) - Backup and restore reference
- [SitePoint: Production Guide (4 Weeks)](https://www.sitepoint.com/openclaw-production-lessons-4-weeks-self-hosted-ai/) - Real-world production lessons
- [GitHub Issue #6477: Auto-backup before doctor](https://github.com/openclaw/openclaw/issues/6477) - Feature request for safer upgrades
- [Microsoft Security Blog: Running OpenClaw Safely](https://www.microsoft.com/en-us/security/blog/2026/02/19/running-openclaw-safely-identity-isolation-runtime-risk/) - Identity, isolation, runtime risk
