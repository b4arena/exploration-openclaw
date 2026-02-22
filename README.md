# OpenClaw Exploration

Structured exploration of the OpenClaw architecture and features.

## Prerequisites

This repo is part of a multi-repo workspace. The following repos should be checked out alongside it:

```
openclaw/                        ← you are here
├── exploration/                 ← research docs (this repo)
├── CLAUDE.md
├── openclaw/                    ← main OpenClaw app (separate repo)
├── infra/                       ← infrastructure / IaC (separate repo)
└── skills/                      ← third-party skills checked out for testing
```

**Note:** Path references in the exploration docs (e.g. `src/providers/...`, `docs/channels/...`) are relative to the `openclaw/` repo root. To resolve them from the workspace root, prepend `openclaw/`.

## Documents

### Reference

| File | Content |
|---|---|
| [external-resources.md](external-resources.md) | Curated external resources: guides, tutorials, community, ecosystem, security |

### Architecture — Core Platform

| File | Content |
|---|---|
| [architecture-overview.md](architecture/architecture-overview.md) | High-level architecture, Identity, Routing, Agents, Provider, Auth |
| [ecosystem.md](architecture/ecosystem.md) | Extensions, Skills, Skill creation, Orchestration, Plugin SDK |
| [security-and-sandbox.md](architecture/security-and-sandbox.md) | Security model, Sandbox, Container, Kubernetes question |
| [process-execution.md](architecture/process-execution.md) | tmux, PTY, Background Processes, Event-Driven Wake-Up |
| [nodes-and-plugins.md](architecture/nodes-and-plugins.md) | Nodes (companion devices: pairing, command surface, exec routing, APNS wake) and Plugin System |
| [cli-reference.md](architecture/cli-reference.md) | CLI command reference: agents, turns, skills, cron, messaging, config, sandbox, diagnostics |
| [session-transcripts.md](architecture/session-transcripts.md) | Session transcript storage (JSONL), built-in and community viewers, live transcript dashboard approaches |

### Agents — Patterns & Communication

| File | Content |
|---|---|
| [subagent-skills.md](agents/subagent-skills.md) | Skill inheritance for Subagents, B4Racing practical example |
| [subagent-deep-dive.md](agents/subagent-deep-dive.md) | Subagent inheritance: Skills, Bootstrap files, System Prompt, Depth control |
| [agent-to-agent.md](agents/agent-to-agent.md) | Agent-to-agent communication: subagent spawn vs sessions_send vs channel-based (Matrix) |
| [software-dev-use-cases.md](agents/software-dev-use-cases.md) | Software development use cases: GitHub integration, CI/CD, code review, coding agents |
| [coding-agents.md](agents/coding-agents.md) | External coding agent integration: Claude Code, Codex, OpenCode, Aider, PTY/exec architecture |
| [external-communication.md](agents/external-communication.md) | External agent communication: CLI, webhooks, MCP bridge projects, npm package, community skills |
| [beyond-file-skills.md](agents/beyond-file-skills.md) | Beyond file-based skills: OpenProse, Lobster, LLM Task, Command Dispatch |

### Deployment — Infrastructure & Operations

| File | Content |
|---|---|
| [best-practices.md](deployment/best-practices.md) | Config management, backup, monitoring/OTel, Docker hardening, Kubernetes/Helm, upgrades |
| [iac-setup.md](deployment/iac-setup.md) | IaC repo layout: Ansible for Linux + WSL2, config management, secrets, Tailscale networking |
| [devops-skills.md](deployment/devops-skills.md) | DevOps skills research: MCP servers (Ansible, Vault, Terraform), built-in skills, custom skill proposals |
| [matrix-server.md](deployment/matrix-server.md) | Matrix server hosting: Conduit vs Synapse vs Dendrite, Docker Compose, bot accounts, E2EE |
| [langfuse-audit-trail.md](deployment/langfuse-audit-trail.md) | Langfuse Cloud as audit trail: OTel integration, trace visualization, pricing, data sovereignty |
| [node-isolation.md](deployment/node-isolation.md) | Node.js isolation on Fedora without containers: Volta, fnm, mise, Nix — comparison table |
| [sandbox-container-lifecycle.md](deployment/sandbox-container-lifecycle.md) | Sandbox container lifecycle: creation timing, templates, reuse, scoping, auto-pruning |

### Beads — Protocol & Multi-Agent Coordination

| File | Content | Status |
|---|---|---|
| [architecture.md](beads/architecture.md) | Beads-based multi-agent architecture: Four-Tier token-saving, identity bridging, community ecosystem | Migrated → [`b4forge/docs/architecture.md`](../b4forge/docs/architecture.md) |
| [sync-patterns.md](beads/sync-patterns.md) | Beads agent synchronization: comment patterns, molecules, bd prime/hooks, bd mail, agent discovery |
| [multi-repo-topology.md](beads/multi-repo-topology.md) | Beads multi-repo topology: source_repo, hydration, auto-routing, Gas Town, routes.jsonl |

## Open Topics

### Not yet explored

- [ ] **Session Management** – How are sessions created, persisted, compacted? Compaction for long conversations?
- [ ] **End-to-End Message Flow** – From "User types in Telegram" to "Response comes back" – every step
- [ ] **Memory System** – Embeddings, Vector Search, Hybrid Search (BM25 + Vector)
- [ ] **Skill Injection** – How are Markdown skills integrated into the system prompt?
- [ ] **Auto-Reply & Group Gating** – When does the agent respond automatically in groups?
- [ ] **Tool System in Detail** – Custom Tools, Tool Execution Loop, Pi Agent Core Integration
- [ ] **Media Pipeline** – Images, Audio, Video, Transcription
- [x] **Config Hot-Reload** – Change config without Gateway restart (covered in [best-practices.md](deployment/best-practices.md))
- [ ] **Web UI Architecture** – Frontend structure, Gateway communication
- [ ] **Native Apps** – macOS Menu Bar, iOS, Android Integration
- [ ] **Cron & Heartbeat** – Scheduled tasks, autonomous Agent activity
- [x] **Beads Multi-Agent Communication** – Beads protocol integration, token-saving notification, agent identity bridging (covered in [architecture.md](beads/architecture.md))

### Ideas & Questions

- [ ] Can you write a custom Sandbox provider (e.g. Kubernetes)?
- [x] How would you run OpenClaw in production in a Kubernetes environment? (covered in [best-practices.md](deployment/best-practices.md))
- [ ] Which Skills are most relevant for our use case?
- [ ] How do you build a Multi-Agent setup with specialized agents?
