# Infrastructure as Code: Ansible Deployment for Linux + WSL2

Cross-platform deployment setup using Ansible, managed from Mac via Claude Code sessions.
Includes the local development workflow and feedback loops for skill/config iteration.

---

## 1. Architecture Overview

```
┌─────────────────────────────┐
│  Mac (Development)          │
│  ├── Claude Code session    │
│  ├── IaC repo (git)         │
│  └── ansible-playbook …     │──SSH──┬──▶  Linux Server (host-based)
│                             │       │      ├── Node.js 22 + OpenClaw
│                             │       │      ├── systemd service
│                             │       │      ├── Docker (sandbox only)
│                             │       │      └── Tailscale VPN
│                             │       │
│                             │       └──▶  Windows Server (WSL2)
│                             │              ├── WSL2 (Ubuntu 24.04)
│                             │              │  ├── Node.js 22 + OpenClaw
│                             │              │  ├── systemd service
│                             │              │  ├── Docker CE (sandbox)
│                             │              │  └── Tailscale
│                             │              └── Windows Task Scheduler
│                             │                  └── wsl -d Ubuntu (auto-start)
└─────────────────────────────┘
```

Both targets are Linux from Ansible's perspective. The official `openclaw-ansible` playbook
(`https://github.com/openclaw/openclaw-ansible`) runs unmodified on both.

---

## 2. Repo Layout

```
openclaw-infra/
├── README.md
├── ansible.cfg                     # Ansible configuration
├── requirements.yml                # Galaxy roles (openclaw-ansible, etc.)
│
├── inventory/
│   ├── hosts.yml                   # All hosts + groups
│   └── group_vars/
│       ├── all.yml                 # Shared defaults
│       ├── linux.yml               # Linux-specific overrides
│       └── windows_wsl.yml         # WSL2-specific overrides
│
├── playbooks/
│   ├── site.yml                    # Full deployment (provision + configure)
│   ├── provision.yml               # One-time setup (Node, Docker, Tailscale, systemd)
│   ├── configure.yml               # Push config + restart if needed
│   ├── upgrade.yml                 # Upgrade OpenClaw + run doctor
│   ├── backup.yml                  # Trigger remote backup
│   └── health-check.yml            # Run openclaw health --json on all hosts
│
├── roles/
│   └── wsl-bootstrap/              # Custom role: WSL2 auto-start + SSH setup
│       ├── tasks/main.yml
│       └── templates/
│           └── wsl-startup.xml.j2  # Windows Task Scheduler XML
│
├── templates/
│   ├── openclaw.json.j2            # Main config (Jinja2 template)
│   ├── agents.json5.j2             # Agent definitions ($include target)
│   ├── env.j2                      # .env file (secrets)
│   └── openclaw.service.j2         # systemd unit (if customized)
│
├── files/
│   └── seccomp-sandbox.json        # Seccomp profile for agent sandboxes
│
├── scripts/
│   ├── deploy.sh                   # Convenience wrapper: ansible-playbook + vault
│   └── rotate-secrets.sh           # Rotate API keys, update vault
│
├── Makefile                        # Developer shortcuts
│
└── .vault-password                 # (gitignored) Ansible Vault password
```

---

## 3. Key Files Explained

### inventory/hosts.yml

```yaml
all:
  children:
    linux:
      hosts:
        prod-linux:
          ansible_host: 10.0.0.10       # Tailscale IP preferred
          ansible_user: deploy
    windows_wsl:
      hosts:
        prod-windows:
          ansible_host: 10.0.0.20       # Tailscale IP of WSL2 instance
          ansible_user: deploy
          ansible_port: 22              # OpenSSH in WSL2
```

Both groups inherit from `all`. From Ansible's perspective, both are standard Linux hosts.

### inventory/group_vars/all.yml

```yaml
# Shared config for all hosts
openclaw_version: "latest"
openclaw_node_version: "22"
openclaw_gateway_port: 18789
openclaw_bind: "loopback"              # Access via Tailscale, not public

# Config structure
openclaw_config:
  gateway:
    port: "{{ openclaw_gateway_port }}"
    bind: "{{ openclaw_bind }}"
    auth:
      token: "{{ vault_openclaw_gateway_token }}"
  agents:
    "$include": "./agents.json5"
  sandbox:
    mode: "non-main"
    docker:
      readOnlyRoot: true
      network: "none"
      capDrop: ["ALL"]
      pidsLimit: 256
      memory: "1g"

# Backup
openclaw_backup_dir: "/var/backups/openclaw"
openclaw_backup_retain_days: 7

# Secrets (Ansible Vault encrypted)
vault_openclaw_gateway_token: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  ...
vault_anthropic_api_key: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  ...
```

### inventory/group_vars/windows_wsl.yml

```yaml
# WSL2-specific overrides
openclaw_wsl_distro: "Ubuntu-24.04"
openclaw_wsl_autostart: true           # Configure Windows Task Scheduler
```

### Makefile

```makefile
VAULT_OPTS = --vault-password-file .vault-password
INV        = -i inventory/hosts.yml

# Full deployment
deploy:
	ansible-playbook $(INV) $(VAULT_OPTS) playbooks/site.yml

deploy-linux:
	ansible-playbook $(INV) $(VAULT_OPTS) playbooks/site.yml --limit linux

deploy-windows:
	ansible-playbook $(INV) $(VAULT_OPTS) playbooks/site.yml --limit windows_wsl

# Config-only push (no provisioning)
configure:
	ansible-playbook $(INV) $(VAULT_OPTS) playbooks/configure.yml

# Upgrade OpenClaw on all hosts
upgrade:
	ansible-playbook $(INV) $(VAULT_OPTS) playbooks/upgrade.yml

# Health check
health:
	ansible-playbook $(INV) $(VAULT_OPTS) playbooks/health-check.yml

# Backup
backup:
	ansible-playbook $(INV) $(VAULT_OPTS) playbooks/backup.yml

# Secrets management
edit-secrets:
	ansible-vault edit $(VAULT_OPTS) inventory/group_vars/all.yml
```

---

## 4. Playbook Sketches

### playbooks/site.yml

```yaml
---
- name: Provision and configure OpenClaw
  hosts: all
  become: true
  roles:
    - role: wsl-bootstrap
      when: "'windows_wsl' in group_names"

  tasks:
    - name: Include official openclaw-ansible tasks
      ansible.builtin.include_role:
        name: openclaw.openclaw
      # The official playbook handles:
      # - Tailscale install + join
      # - UFW firewall
      # - Docker CE + Compose V2
      # - Node.js 22 + pnpm
      # - OpenClaw install
      # - systemd service with hardening

    - name: Deploy openclaw.json from template
      ansible.builtin.template:
        src: templates/openclaw.json.j2
        dest: "~/.openclaw/openclaw.json"
        owner: deploy
        mode: "0600"
      notify: restart openclaw

    - name: Deploy .env from template
      ansible.builtin.template:
        src: templates/env.j2
        dest: "~/.openclaw/.env"
        owner: deploy
        mode: "0600"
      notify: restart openclaw

    - name: Deploy agent configs
      ansible.builtin.template:
        src: templates/agents.json5.j2
        dest: "~/.openclaw/agents.json5"
        owner: deploy
        mode: "0600"
      # No restart needed: agents config hot-reloads

  handlers:
    - name: restart openclaw
      ansible.builtin.systemd:
        name: openclaw-gateway
        state: restarted
```

### playbooks/upgrade.yml

```yaml
---
- name: Upgrade OpenClaw
  hosts: all
  become: true
  serial: 1                            # Rolling upgrade, one host at a time
  tasks:
    - name: Backup before upgrade
      ansible.builtin.include_tasks: ../playbooks/backup.yml

    - name: Upgrade OpenClaw
      ansible.builtin.shell: |
        npm update -g openclaw@{{ openclaw_version }}
      args:
        executable: /bin/bash

    - name: Run doctor
      ansible.builtin.command: openclaw doctor --fix
      become_user: deploy

    - name: Restart gateway
      ansible.builtin.systemd:
        name: openclaw-gateway
        state: restarted

    - name: Health check
      ansible.builtin.command: openclaw health --json
      become_user: deploy
      register: health
      retries: 3
      delay: 10
      until: health.rc == 0
```

### playbooks/health-check.yml

```yaml
---
- name: Health check all hosts
  hosts: all
  tasks:
    - name: Run health check
      ansible.builtin.command: openclaw health --json
      register: health_result
      changed_when: false

    - name: Show health
      ansible.builtin.debug:
        msg: "{{ health_result.stdout | from_json }}"
```

---

## 5. Config Management Strategy

### Layered Configuration

OpenClaw's `$include` feature enables a layered config approach (`docs/gateway/configuration.md:307-328`):

```
openclaw.json (base: gateway, sandbox defaults)
└── $include: agents.json5 (agent definitions, per-host)
```

Ansible templates render host-specific values via Jinja2 variables. Secrets are injected
from Ansible Vault via `${ENV_VAR}` substitution in the config.

### Hot-Reload vs. Restart

Most config changes hot-reload automatically (`docs/gateway/configuration.md:333-369`):

| Change Type | Reload | Examples |
|---|---|---|
| Agents, channels, models, tools | Hot-reload | Adding an agent, changing model |
| Gateway server settings | Restart required | Port, bind address, auth token |

The `configure.yml` playbook uses a handler that only restarts when gateway-level changes
are detected. Agent and channel changes are applied automatically.

### Secrets Flow

```
Ansible Vault (encrypted in git)
    │
    ├──▶ .env file on host (~/.openclaw/.env)
    │       ANTHROPIC_API_KEY=sk-ant-...
    │       OPENCLAW_GATEWAY_TOKEN=...
    │
    └──▶ openclaw.json (env var substitution)
            "token": "${OPENCLAW_GATEWAY_TOKEN}"
```

Secrets never appear in plaintext in the git repo. `ansible-vault encrypt_string` encrypts
individual values in `group_vars/all.yml`.

---

## 6. Claude Code Workflow

A typical deployment session from the Mac:

```bash
# 1. Open Claude Code in the infra repo
cd ~/src/openclaw-infra
claude

# 2. Ask Claude to deploy
> "deploy the latest config to both servers"
# Claude runs: make deploy

# 3. Ask Claude to check health
> "are both instances healthy?"
# Claude runs: make health

# 4. Ask Claude to upgrade
> "upgrade openclaw on the linux server first"
# Claude runs: make deploy-linux && make upgrade -- --limit linux

# 5. Ask Claude to rotate secrets
> "rotate the gateway token on all hosts"
# Claude runs: scripts/rotate-secrets.sh + make configure
```

Claude Code has SSH access (via Tailscale) and can run Ansible commands directly.
The Makefile provides a clean interface that Claude can invoke autonomously.

---

## 7. WSL2-Specific Considerations

### Auto-Start

WSL2 does not start automatically on Windows boot. The `wsl-bootstrap` role handles this:

1. Creates a Windows Task Scheduler task via XML template
2. Task runs `wsl -d Ubuntu-24.04` at system startup
3. WSL2 starts with systemd enabled (Windows 11+)
4. OpenClaw systemd service starts automatically within WSL2

### SSH Access

The role installs OpenSSH Server inside WSL2 and configures port forwarding:

```powershell
# Windows firewall rule (created by the role via WinRM pre-step or manual)
netsh interface portproxy add v4tov4 listenport=22 listenaddress=0.0.0.0 \
  connectport=22 connectaddress=$(wsl hostname -I | awk '{print $1}')
```

Alternative: Use Tailscale inside WSL2 — then SSH connects directly to the Tailscale IP,
bypassing Windows networking entirely (recommended).

### Docker in WSL2

Two options:
- **Docker Desktop** (requires license for orgs > 250 employees)
- **Docker CE inside WSL2** (free, recommended) — install via official apt repo

The Ansible playbook installs Docker CE by default.

---

## 8. Network Access Pattern

```
Mac ──Tailscale──▶ Linux Server:22   (SSH / Ansible)
                 ▶ Linux Server:18789 (Gateway WS, via Tailscale only)

Mac ──Tailscale──▶ WSL2:22           (SSH / Ansible, Tailscale in WSL2)
                 ▶ WSL2:18789        (Gateway WS, via Tailscale only)
```

Both gateways bind to `loopback`. Tailscale provides the secure overlay network.
No ports are exposed to the public internet. UFW blocks everything except SSH + Tailscale
(`docs/install/ansible.md` — 4-layer security model).

---

## 9. Development Workflow & Feedback Loops

The "REPL" depends on what you're developing. OpenClaw offers several iteration modes
with different feedback speeds.

### Iteration Speed Overview

| What you're changing | REPL / Tool | Feedback speed | Restart needed? |
|---|---|---|---|
| Skill (SKILL.md) | `openclaw tui` or `openclaw agent --message "..."` | Seconds | No (loads on next session) |
| Agent personality (IDENTITY.md, SOUL.md) | `openclaw tui` | Seconds | No (loads on next session) |
| Config (openclaw.json) | Direct edit / `openclaw config set` / Control UI | Seconds | No (hot-reload) |
| Gateway settings (port, auth) | Direct edit | Seconds + restart | Yes (auto-restart in hybrid mode) |
| Plugin (TypeScript) | `openclaw plugins install -l` | 1-2 min | Yes (gateway restart) |
| Gateway core (src/) | `pnpm gateway:watch` | 30-60s | Yes (auto-rebuild) |

### The Primary REPLs

**`openclaw tui`** — Interactive terminal chat (the main REPL):
```bash
openclaw tui                          # Local gateway
openclaw tui --url ws://... --token …  # Remote gateway
openclaw tui --local                   # Embedded, no gateway needed
```

Features: multi-turn sessions, model picker (Ctrl+L), agent picker (Ctrl+G),
session switcher (Ctrl+P), tool output expansion (Ctrl+O), slash commands
(`docs/cli/tui.md`).

**`openclaw agent --message "..."`** — One-shot agent turn (scriptable):
```bash
openclaw agent --message "use my new deploy skill"
openclaw agent --message "summarize today" --agent work
openclaw agent --message "test" --local   # No gateway needed
```

Best for: CI pipelines, automated tests, quick one-off checks (`docs/cli/agent.md`).

**Control UI** — Web-based at `http://127.0.0.1:18789`:
- Config editor (form + raw JSON)
- Channel status + logs
- Session viewer
- Model picker

### Skill Development Loop (Fastest)

Skills live at `~/.openclaw/workspace/skills/<name>/SKILL.md` and are discovered at
**session start**, not gateway start (`docs/tools/skills.md:21-26`). Precedence on
name collision: workspace > local > bundled > extraDirs.

```
1. Create/edit skill:
   ~/.openclaw/workspace/skills/deploy/SKILL.md

2. Test immediately:
   openclaw agent --message "run the deploy skill"
   # or use openclaw tui (next message = new session = skill loaded)

3. Verify skill is visible:
   openclaw skills list
   openclaw skills check         # Verify requirements (binaries, env vars)
```

No gateway restart needed. Edit → next agent turn → done.

**Key constraint**: skills load at session start; changes mid-session require starting
a new session (not a gateway restart).

### Agent Behavior Tuning

Workspace files that shape agent personality (`docs/reference/templates/`):

| File | Purpose | Reload |
|---|---|---|
| `IDENTITY.md` | Agent personality and role definition | Next session |
| `SOUL.md` | Deep behavioral patterns | Next session |
| `AGENTS.md` | Multi-agent routing rules | Next session |
| `BOOT.md` | Startup checklist (runs every gateway start) | Gateway restart |
| `BOOTSTRAP.md` | First-run onboarding (one-time only) | N/A |

All except BOOT.md take effect on the next session without any restart.

### Config Iteration

Three ways to modify config, all triggering hot-reload (`docs/gateway/configuration.md:333-369`):

```bash
# CLI
openclaw config set agents.list[0].model "anthropic/claude-sonnet-4-6"
openclaw config get gateway.port

# Direct file edit (watched by gateway)
vim ~/.openclaw/openclaw.json

# Control UI (browser)
open http://127.0.0.1:18789
```

Hot-reload modes (`gateway.reload.mode` in config):

| Mode | Behavior |
|---|---|
| `hybrid` (default) | Hot-applies safe changes; auto-restarts for critical ones |
| `hot` | Hot-applies safe changes only; logs warning when restart needed |
| `restart` | Restarts on any change |
| `off` | Manual restart only |

Safe (hot-reload): channels, agents, models, tools, skills allowlist.
Critical (restart required): gateway port, bind address, auth token, TLS.

### Gateway Core Development

For working on OpenClaw itself (`docs/help/debugging.md:49-100`):

```bash
pnpm gateway:watch         # File watcher on src/ → auto-rebuild + restart
pnpm gateway:dev           # Isolated state (~/.openclaw-dev), port 19001
```

Dev mode features:
- Isolated state directory (`~/.openclaw-dev`) — safe experimentation
- Shifted ports (19001 default) — runs alongside production
- Skips BOOTSTRAP.md and channel providers by default
- Raw stream logging: `pnpm gateway:watch --raw-stream`

Runtime debug overrides (not persisted, requires `commands.debug: true`):
```
/debug show
/debug set messages.responsePrefix="[test]"
/debug reset
```

### Testing Infrastructure

Three test suites with different trade-offs (`docs/help/testing.md`):

| Suite | Command | Real API Keys | Speed |
|---|---|---|---|
| Unit/Integration | `pnpm test` | No | Fast (seconds) |
| E2E (Gateway smoke) | `pnpm test:e2e` | No | Medium (minutes) |
| Live | `pnpm test:live` | Yes | Slow (minutes, costs money) |

Quick pre-push check:
```bash
pnpm build && pnpm check && pnpm test
```

Live testing with specific models:
```bash
OPENCLAW_LIVE_MODELS="anthropic/claude-opus-4-6,openai/gpt-5.2" \
  pnpm test:live src/gateway/gateway-models.profiles.live.test.ts
```

### End-to-End Dev-to-Deploy Workflow

Typical daily workflow on the Mac:

```
┌─ Mac (Development) ─────────────────────────────────────────────┐
│                                                                  │
│  Terminal 1: Gateway                                             │
│  $ openclaw gateway                                              │
│                                                                  │
│  Terminal 2: Iterate (REPL)                                      │
│  $ openclaw tui                                                  │
│  > test my new skill                                             │
│  > /status                                                       │
│                                                                  │
│  Terminal 3: Logs                                                 │
│  $ openclaw logs --follow                                        │
│                                                                  │
│  Editor: Edit skills + config                                    │
│  $ vim ~/.openclaw/workspace/skills/deploy/SKILL.md              │
│  $ openclaw config set agents.list[0].model "..."                │
│                                                                  │
│  ── Ready to deploy ──────────────────────────────────────────── │
│                                                                  │
│  Terminal 4: Claude Code in infra repo                           │
│  $ cd ~/src/openclaw-infra && claude                             │
│  > "push the new skill config to both servers"                   │
│  # Claude runs: make configure                                   │
│  > "health check"                                                │
│  # Claude runs: make health                                      │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

The key insight: develop locally with `openclaw tui` as REPL, then promote config
changes to the IaC repo and deploy via Ansible. Skills and workspace files can be
synced via the `configure.yml` playbook or managed as Ansible templates.

---

## References

### In-Repo Documentation
- `docs/install/ansible.md` — Official Ansible deployment guide
- `docs/gateway/configuration.md:307-328` — `$include` for modular configs
- `docs/gateway/configuration.md:333-369` — Hot-reload modes
- `docs/gateway/authentication.md:31-37` — `.env` precedence for secrets
- `docs/install/hetzner.md:312-328` — Persistence model (what to back up)
- `docs/cli/tui.md` — TUI interactive chat (primary REPL)
- `docs/cli/agent.md` — One-shot agent CLI
- `docs/tools/skills.md:21-26` — Skill discovery and precedence
- `docs/tools/creating-skills.md` — Skill authoring guide
- `docs/help/debugging.md:49-100` — Dev mode, watch mode, debug overrides
- `docs/help/testing.md` — Test suites (unit, e2e, live)
- `docs/reference/templates/` — BOOT.md, BOOTSTRAP.md, IDENTITY.md templates

### External
- `https://github.com/openclaw/openclaw-ansible` — Official Ansible playbook
- `exploration/deployment/best-practices.md` — General deployment best practices
