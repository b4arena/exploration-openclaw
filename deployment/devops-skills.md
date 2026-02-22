# DevOps Skills Research: Claude Code & OpenClaw Skills for Ansible Deployment

Research into existing and custom skills that support the Ansible-based deployment workflow
described in `exploration/deployment/iac-setup.md`.

---

## 1. Relevant MCP Servers

### Available Today

| MCP Server | Provider | Purpose | Relevance |
|---|---|---|---|
| **Ansible Automation Platform MCP** | Red Hat | Execute playbooks, manage inventory, validate syntax | Direct fit — wraps `ansible-playbook` |
| **Vault MCP Server** | HashiCorp | Manage secrets, SSH certificates, mounts | Replaces/augments Ansible Vault for secrets |
| **Terraform MCP Server** | HashiCorp | Query registry, inspect state, trigger runs | Future — if migrating IaC to Terraform |
| **Git/GitHub MCP** | Community | Clone repos, validate diffs, trigger CI | Already covered by `gh` CLI + `github` skill |

### Ansible MCP Server Details

Red Hat's official MCP integration (Technology Preview in AAP 2.6.4):
- Execute playbooks with customizable parameters
- View/manage Ansible inventory
- Validate playbook syntax without execution
- Preview tasks before execution

**For our setup:** This targets the full Ansible Automation Platform, not standalone Ansible.
For local `ansible-playbook` usage, a custom skill wrapping the CLI is simpler and more appropriate.

### Vault MCP Server Details

HashiCorp's MCP server for Vault:
- Manage secrets and mounts
- SSH secrets engine for short-lived credentials
- Supports stdio and StreamableHTTP transports

**For our setup:** Could replace Ansible Vault for secrets management. Overkill for two hosts,
but valuable if scaling to more servers. Worth evaluating in Phase 3.

### Security Note

Per 2025 research (Astrix Security), 53% of MCP servers rely on insecure long-lived secrets.
Best practice: use wrapped MCP with AWS Secrets Manager or similar for credential injection
instead of hardcoding API keys in MCP config.

---

## 2. Relevant Built-in OpenClaw Skills

OpenClaw ships 52 bundled skills (`exploration/architecture/ecosystem.md`). These are directly useful
for the deployment workflow:

### `healthcheck` — Security Auditing

- Firewall and SSH hardening recommendations
- OpenClaw security audit integration (`openclaw security audit --deep`)
- Per-risk-profile remediation plans
- Source: `skills/healthcheck/SKILL.md`

### `github` — Git Operations via `gh` CLI

- PR/issue management, CI status checks
- Can monitor deployment pipelines
- Source: `skills/github/SKILL.md`

### `tmux` — Remote Session Control

- Send keystrokes, scrape output from running sessions
- Useful for monitoring long-running Ansible playbooks
- Source: `skills/tmux/SKILL.md`

### `coding-agent` — Delegate to Claude Code / Codex

- Requires `pty: true` for interactive terminal apps
- Background execution with `background: true`
- Can spawn parallel sessions
- Perfect for orchestrating multi-host deployments via Claude Code
- Source: `skills/coding-agent/SKILL.md`

### `skill-creator` — Bootstrap New Skills

- Programmatically creates SKILL.md files
- Useful for scaffolding the custom DevOps skills below
- Source: `skills/skill-creator/SKILL.md`

---

## 3. Custom Skills to Build

### Priority 1: Core Deployment Skills

#### `ansible-deploy`

Wraps `ansible-playbook` for use in OpenClaw agent conversations and Claude Code sessions.

```yaml
---
name: ansible-deploy
description: Run Ansible playbooks from the openclaw-infra repo
user-invocable: true
tools:
  - exec
  - read_file
---
```

Key capabilities:
- Accept playbook name + optional `--limit` target
- Auto-detect vault password file
- Parse output for changed/failed/ok counts
- Return structured result with per-host status
- Support `--check` mode for dry runs

#### `remote-health`

SSH to remote OpenClaw instances and aggregate health status.

```yaml
---
name: remote-health
description: Check health of all deployed OpenClaw instances
user-invocable: true
tools:
  - exec
---
```

Key capabilities:
- Run `openclaw health --json` on each host via SSH
- Check systemd service status
- Report disk space, memory usage
- Aggregate into single status overview
- Alert on unhealthy hosts

#### `vault-secrets`

Manage Ansible Vault secrets without exposing them in logs.

```yaml
---
name: vault-secrets
description: Manage encrypted secrets in the infrastructure repo
user-invocable: true
tools:
  - exec
  - read_file
  - write_file
---
```

Key capabilities:
- List vault-protected variables (names only, not values)
- Rotate individual secrets
- Add new secrets
- Validate vault password is available
- Never display decrypted values in agent output

### Priority 2: Orchestration Skills

#### `rolling-upgrade`

Orchestrates upgrades across multiple hosts with safety checks.

```yaml
---
name: rolling-upgrade
description: Upgrade OpenClaw across all hosts with rollback support
user-invocable: true
tools:
  - exec
---
```

Key capabilities:
- Pre-upgrade backup validation on each host
- Serial upgrade (one host at a time)
- Health check between upgrades
- Automatic rollback on failure
- Summary report with before/after versions

#### `config-validate`

Validate OpenClaw config before deploying.

```yaml
---
name: config-validate
description: Validate OpenClaw configuration before deployment
user-invocable: true
tools:
  - exec
  - read_file
---
```

Key capabilities:
- Validate JSON5 syntax of `openclaw.json.j2` templates
- Check that `$include` targets exist
- Verify env var references (`${VAR_NAME}`) have corresponding vault entries
- Flag security issues (e.g., bind set to `lan` without firewall)
- Diff current vs. new config per host

---

## 4. Claude Code Skills / Slash Commands

Claude Code supports custom slash commands via skills marked `user-invocable: true`
(`docs/tools/slash-commands.md:130-135`). These auto-expose as `/skill-name` in chat.

For the infra repo, the CLAUDE.md already provides context. Additional Claude Code
integration options:

### Custom Slash Commands for the Infra Repo

These would live in `~/.openclaw/workspace/skills/` on the Mac:

| Command | Maps to | Purpose |
|---|---|---|
| `/deploy` | `make deploy` | Full deployment |
| `/configure` | `make configure` | Config-only push |
| `/upgrade` | `make upgrade` | Rolling upgrade |
| `/health` | `make health` | Health check all hosts |
| `/backup` | `make backup` | Trigger backups |

### Alternative: Direct Makefile Integration

Since Claude Code can run `make` commands directly, creating OpenClaw skills for this
may be over-engineering. The Makefile already provides a clean interface.

**Recommendation:** Start with the Makefile approach. Only create dedicated skills if
the workflow becomes more complex (e.g., multi-step orchestration, conditional logic,
parallel deployments).

---

## 5. Plugin Opportunity

OpenClaw's plugin system (`docs/plugins/manifest.md`, `docs/plugins/agent-tools.md`) supports:

- `api.registerTool()` — Register agent tools
- `api.registerHook()` — Event handlers (e.g., `gateway:startup`)
- `api.registerHttpHandler()` — Webhooks
- `api.registerCli()` — CLI subcommands
- `api.registerService()` — Background services

A `devops-toolkit` plugin could bundle:
1. All custom skills from Section 3
2. A `gateway:startup` hook for automatic health checks
3. A background service for periodic inventory sync from Tailscale
4. A `/infra` CLI subcommand for quick operations

**Recommendation:** Defer plugin creation to Phase 3. Skills + Makefile are sufficient
for the initial setup.

---

## 6. Recommended Phased Approach

### Phase 1 — Immediate (Week 1-2)

1. Use **Makefile** as the primary interface for Claude Code sessions
2. Create `ansible-deploy` custom skill (wraps `make deploy/configure/upgrade`)
3. Create `remote-health` custom skill (wraps `make health` with structured output)
4. Test with the playbooks from `openclaw-infra/`

### Phase 2 — Short-term (Week 3-4)

5. Create `vault-secrets` skill for safe secret rotation
6. Create `config-validate` skill for pre-deploy checks
7. Evaluate Vault MCP Server for secrets management

### Phase 3 — Medium-term (Month 2-3)

8. Bundle skills into a `devops-toolkit` plugin
9. Add `gateway:startup` hook for automatic health reporting
10. Integrate Ansible MCP Server if scaling to more hosts
11. CI/CD pipeline: auto-deploy on config repo push

### Phase 4 — Future

12. Terraform MCP Server for hybrid IaC
13. Multi-region failover orchestration
14. Inventory auto-sync from Tailscale admin API

---

## References

### In-Repo Documentation
- `docs/tools/skills.md` — Skill system overview, discovery, precedence
- `docs/tools/creating-skills.md` — SKILL.md format, frontmatter, requirements
- `docs/tools/slash-commands.md:130-135` — User-invocable skills as slash commands
- `docs/plugins/manifest.md` — Plugin manifest format
- `docs/plugins/agent-tools.md` — Plugin tool registration API
- `skills/healthcheck/SKILL.md` — Built-in security audit skill
- `skills/coding-agent/SKILL.md` — Coding agent delegation skill
- `skills/tmux/SKILL.md` — tmux remote session skill

### Exploration Cross-References
- `exploration/architecture/ecosystem.md` — Full list of 52 bundled skills
- `exploration/agents/subagent-skills.md` — Skill inheritance for subagents
- `exploration/deployment/iac-setup.md` — Ansible playbook structure and dev workflow

### External
- [Ansible MCP Server — Red Hat](https://www.redhat.com/en/blog/it-automation-agentic-ai-introducing-mcp-server-red-hat-ansible-automation-platform)
- [Ansible Development Tools MCP Server](https://docs.ansible.com/projects/vscode-ansible/mcp/)
- [Vault MCP Server — HashiCorp](https://developer.hashicorp.com/vault/docs/mcp-server/overview)
- [Terraform MCP Server — HashiCorp](https://github.com/hashicorp/terraform-mcp-server)
- [MCP Server Security 2025 — Astrix](https://astrix.security/learn/blog/state-of-mcp-server-security-2025/)
