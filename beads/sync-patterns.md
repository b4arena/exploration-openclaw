# 21 — Beads Agent Synchronization & Communication Patterns

## Scope

Deep research into how agents coordinate within the Beads ecosystem:
communication patterns, molecules, `bd prime` / session hooks, agent discovery,
`bd mail`, community tools (BeadHub, beads-village, Gas Town, The Claude
Protocol), and implications for an OpenClaw integration.

For the foundational architecture (Four-Tier token saving, identity bridging,
notification flow, Phase 1–3 implementation plan) see
[architecture.md](architecture.md).

---

## 1. Multi-Agent Coordination Best Practices

### 1.1 The Core Principle: DAG = The Coordination Contract

The Beads dependency graph is the single source of truth for agent work
allocation. Every coordination pattern flows from one rule:

> "Agents follow dependency edges, not instructions." — Beads best practices

Practical consequences:
- Agents never negotiate what to do next; they call `bd ready --json` and work
  whatever the DAG unblocks.
- Sequence is expressed via `blocks` edges, not prose. "Phase 1 before Phase 2"
  becomes `bd dep add phase2 phase1` (requirement language, not sequence
  language).
- Parallel work is the default; dependencies are added only when ordering is
  genuinely required.

### 1.2 Task Lifecycle Protocol

Every agent session follows this cycle:

```
bd ready --json           → select next unblocked task
bd update <id> --claim    → atomic claim (prevents double-claiming)
[execute work]
bd comment <id> "..."     → log progress / blockers inline
bd update <id> --status done
bd sync                   → commit + push (mandatory before session ends)
```

The push is non-negotiable. From `AGENT_INSTRUCTIONS.md`:
> "The plane has NOT landed until `git push` completes successfully. NEVER say
> 'ready to push when you are' — YOU must push, not the user."

### 1.3 Landing the Plane Checklist

When a session ends, agents must complete all four steps:

1. File remaining work as new beads (use `bd create`)
2. Run quality gates (lint, tests)
3. Update all touched issue statuses
4. Push to remote (`bd sync` handles both commit and push)

Unpushed work breaks the coordination workflow for all other agents because the
shared DAG state only propagates via git.

---

## 2. Comment Patterns: Progress, Blockers, Handoffs

Comments on beads are the primary mechanism for inter-agent context
propagation. The same bead comment is read by the next session (or next agent)
that claims the task.

### 2.1 Progress Comments

Written inline as work progresses; not batched at the end:

```
bd comment bd-a1b2 "Implemented auth middleware. Tests passing (12/12).
Next: wire up the session expiry callback (bd-a1b2.3)."
```

Fields agents typically include:
- What was done
- Test/lint status
- Which subtask is next
- Any discoveries (link with `bd dep relate` to new beads)

### 2.2 Blocker Comments

When work cannot proceed:

```
bd comment bd-c3d4 "BLOCKED: requires API key from infra team.
Filed bd-e5f6 to track the credential rotation. Will unblock on merge."
bd dep add bd-c3d4 bd-e5f6   # makes blocking explicit in DAG
bd update bd-c3d4 --status blocked
```

The status change to `blocked` removes the task from `bd ready` output,
preventing other agents from wasting effort claiming it.

### 2.3 Handoff Comments

The handoff comment is the primary mechanism for cross-session continuity:

```
bd comment bd-a1b2 "HANDOFF: auth middleware complete.
Context: using JWT with 15m expiry + refresh token rotation.
Known issue: RS256 key size hardcoded in config/auth.ts:42 — should
become an env var.
Next step: implement /refresh endpoint (see bd-a1b2.4).
Run: npm test auth to validate environment."
```

The pattern from community research: "a ready-to-paste summary that the next
agent (or tomorrow's session) can use to immediately orient itself."

Dispatch prompts themselves are auto-logged as bead comments in the Claude
Protocol framework (via a PostToolUse hook), so supervisors reading a bead
inherit the full dispatch context automatically.

### 2.4 Discovery Comments

When work uncovers unexpected scope:

```
bd create "Fix RS256 key hardcoding" --parent bd-a1b2 \
  --dep-type discovered-from --dep bd-a1b2
bd comment bd-a1b2 "Discovered: RS256 hardcoding filed as bd-g7h8
(not blocking this task, but should be done before v2 release)."
```

The `discovered-from` link type keeps the discovery trace without blocking the
original work.

---

## 3. Molecules: Compound Workflow Execution

### 3.1 What a Molecule Is

A molecule is an epic (parent bead with children) that has been given
**execution semantics** — i.e., an agent (or orchestrator) that will traverse
it to completion.

Source: `docs/MOLECULES.md`

> "A molecule is just an epic (parent + children) with workflow semantics."

The chemistry metaphor extends to three phases:

| Phase | Term | State | Storage |
|---|---|---|---|
| Template | **Proto** | Frozen template | `.beads/formulas/` (TOML or JSON) |
| Active instance | **Mol** | Persistent work | Synced to repo via JSONL |
| Ephemeral | **Wisp** | Temporary | Local only, not in JSONL |

Operations:
- `bd mol pour <formula>` — instantiate a proto as a mol (creates the full
  child-issue graph)
- `bd wisp <formula>` — create ephemeral instance
- `bd mol squash <id>` — compress to digest (preserve outcome, discard detail)
- `bd mol burn <id>` — discard entirely

### 3.2 Dependency-Driven Execution

Children are parallel by default. Sequence is imposed by `blocks`/`waits-for`:

```toml
# .beads/formulas/deploy.toml
[molecule]
title = "Deploy to production"

[[children]]
id = "test"
title = "Run full test suite"

[[children]]
id = "build"
title = "Build release artifact"
deps = ["test"]          # blocks: test must close before build starts

[[children]]
id = "deploy"
title = "Push to production"
deps = ["build"]

[[children]]
id = "smoke"
title = "Smoke test production"
deps = ["deploy"]
```

An agent picks up the molecule by calling `bd ready --parent <mol-id>` and
works through whatever children are unblocked.

### 3.3 Bonding: Cross-Molecule Traversal

When molecule A **bonds** to molecule B (A blocks B), an agent completing all
of A's work can seamlessly continue into B without a new session start. This
enables multi-day workflows without explicit handoff messages.

Use case: code → review → deploy pipeline expressed as three bonded molecules.
The orchestrator runs `bd ready` and follows the dependency chain across all
three without needing to re-brief.

### 3.4 Gates: External Synchronization Points

Gates are a molecule-native feature for pausing execution until external
conditions are met. Implemented as issues with `issue_type: "gate"`:

| Gate type | await_type | Resolves when |
|---|---|---|
| Timer | `timer` | ISO 8601 timestamp reached |
| CI pipeline | `gh:run` | GitHub Actions workflow succeeds |
| PR merge | `gh:pr` | Pull request `merged_at` set |
| Human approval | `human` | Operator calls approval endpoint with token |
| Mail response | `mail` | Specified email thread receives a reply |

`bd gate check` polls the condition. When the gate clears, blocked child tasks
become ready. The `waiters` field sends notifications to listed addresses.

### 3.5 WaitsFor: Dynamic Fanout

`waits-for` dependencies (distinct from `blocks`) enable dynamic patterns:

- `all-children`: parent unblocks only after all children close (parallel fan-in)
- `any-children`: parent unblocks when first child closes (race / fastest-wins)

These are declared on the parent issue:
```
bd dep add <parent-id> <child-id> --type waits-for --meta '{"mode":"all-children"}'
```

### 3.6 Common Molecule Pitfalls

| Pitfall | Cause | Fix |
|---|---|---|
| Issue never unblocks | `blocks` dep added in wrong direction | Check: "A blocks B" → B depends on A → `bd dep add B A --type blocks` |
| Children run in wrong order | Numbered names assumed to imply sequence | Add explicit `blocks` edges |
| Wisps accumulate | No cleanup routine | Add `bd mol squash` / `bd mol burn` to session-end checklist |
| Gates never clear | External condition not met | Check `bd gate check <id>` output and `waiters` notifications |

---

## 4. `bd prime`: Context Injection for Agent Sessions

### 4.1 What It Does

`bd prime` generates a compact (~1–2k token) workflow briefing injected into
the agent's system prompt at session start. It contains:

- P0/P1/P2 priority breakdown of open issues
- Blocking vs. unblocked issue summary
- Ready work (issues with no active blockers — equivalent to `bd ready`)
- Active dependency relationships

Source: `cmd/bd/prime.go:88` (referenced in DeepWiki).

Contrast with MCP tool schemas (which consume 10–50k tokens for tool
definitions alone): `bd prime` is intentionally lean.

### 4.2 Token Efficiency Architecture

Two optimized data models are used internally:

| Model | Description | Savings vs full object |
|---|---|---|
| `BriefIssue` | Stripped issue representation | ~97% token reduction |
| `BriefDep` | Minimal dependency info | ~97% token reduction |

This allows 50+ issues to fit in the same budget that would otherwise fit 2–3
full issue objects.

### 4.3 PRIME.md Workspace Override

A `.beads/PRIME.md` file, if present, overrides the auto-generated `bd prime`
output. Teams use this to inject stable context (architectural decisions, team
conventions, current sprint focus) that persists across sessions without
repeating it in every bead comment.

Example use in OpenClaw context: put agent-specific context (e.g., "Alice
handles frontend; Bob handles infra") in PRIME.md so every session starts
with that routing context.

### 4.4 Session Hooks: Installation

```bash
bd setup claude             # installs globally (all projects)
bd setup claude --project   # installs for current project only
bd setup claude --stealth   # stealth mode: flush only, no prime injection
bd setup claude --check     # verify installation
bd setup claude --remove    # uninstall
```

Two hooks are installed:

| Hook | Trigger | Action |
|---|---|---|
| **SessionStart** | Conversation begins | Runs `bd prime` → injects ~1-2k tokens of workflow context |
| **PreCompact** | Before context window compaction | Runs `bd sync` + `bd ready` → refreshes available work, prevents context loss |

The PreCompact hook is critical for long coding sessions: without it, the agent
loses awareness of the Beads state as context compresses.

### 4.5 bd prime vs. TodoWrite

From GitHub issue #499, Steve Yegge resolved a conflict between `bd prime`'s
strict "no TodoWrite" stance and the Claude Code skill's hybrid recommendation:

**Resolved position:**
- Use Beads for all multi-session work and complex dependency chains
- TodoWrite acceptable for simple single-session linear tasks
- Default to Beads when uncertain

The hybrid approach is now encoded in `bd prime` output.

### 4.6 Staleness Detection (Issue #1637)

`bd prime` historically had no freshness check — a stale daemon could go
undetected for weeks. The proposed fix (now superseded by the Dolt migration):

- In git-portable mode: compare JSONL metadata hashes or git status
- In dolt-native mode: verify Dolt remote sync and uncommitted changes

Must complete in under 1 second (executes at session start). The `bd sync
--status` infrastructure supports this.

---

## 5. `bd mail`: Native Inter-Agent Messaging

### 5.1 Data Model

Messages in Beads are first-class issues with `type: message`. This means:

- Same atomic claim and dependency mechanisms as regular issues
- Threading via `replies_to` dependency chains
- Ephemeral lifecycle: `bd mail --older-than N` for cleanup
- Same identity resolution (BEADS_AGENT / `--agent` flag)

Commands:

```bash
bd mail send --to alice "PR #42 review request: bd-a1b2"
bd mail --unread --json --agent alice    # poll for unread mail
bd show msg-123 --thread                 # read full thread
bd mail --older-than 7 --delete          # cleanup
```

### 5.2 Mail as Async Queue

The Tier 1 watcher (from [architecture.md](architecture.md))
polls mail alongside tasks:

```bash
MAIL=$(bd mail --unread --json --agent="$AGENT" 2>/dev/null)
```

Zero tokens consumed by the poll itself. The agent only wakes (and spends
tokens) when unread mail exists.

### 5.3 Communication Pattern: Mail-First, Chat When Blocked

From community research:

| Channel | Pattern | Use case |
|---|---|---|
| `bd mail` | Async, fire-and-forget | Status updates, review requests, handoff notes |
| `bd comment` | Inline, bead-scoped | Progress notes, discoveries, context for next session |
| OpenClaw `sessions_send` | Sync RPC | When blocked and need immediate answer to proceed |
| Gate (`type: human`) | Blocking checkpoint | Human approval required before proceeding |

### 5.4 Mail in beads-village Context

beads-village extends `bd mail` semantics with broadcast support:

```python
msg(to="agent-name", text="PR ready for review: bd-a1b2")
msg(global=True, to="all", text="Deployment blocked: staging env down")
inbox()   # retrieve received messages
```

Messages persist in `.mail/` directory (git-tracked), survives session
restarts.

---

## 6. Agent Discovery

### 6.1 What "Agent Discovery" Means in Beads

Beads does not have a dynamic service registry. Agent discovery is handled by
three mechanisms:

**1. Issue Status Inspection**

```bash
bd list --status in_progress --json   # shows who has claimed what
```

The `assignee` field on `in_progress` issues reveals which agent is doing what.

**2. Actor Attribution on Comments/History**

Every `bd comment` and `bd update` is attributed to an actor (BEADS_ACTOR or
git user.name). Reading a bead's history reveals which agents have touched it.

**3. Explicit AGENTS.md / SOUL.md Files**

The OpenClaw pattern (from [architecture.md](architecture.md)):
each agent's `AGENTS.md` file describes known agents and their capabilities.
The main orchestrator's `AGENTS.md` is the capability registry.

Beads does not automatically maintain this — it is a convention the team must
enforce.

### 6.2 BeadHub: Dynamic Discovery

BeadHub adds presence awareness on top of Beads:

```
bdh status --include-agents   # shows all online agents, their claimed tasks, focus bead
```

Agents are assigned roles automatically (`alice`, `bob`, `charlie`) and can be
given specialization roles (`fe`, `be`, `reviewer`). The web dashboard
(`app.beadhub.ai`) shows live agent activity.

BeadHub stores presence in Redis (TTL-based, auto-expires on crash) and
coordination state in PostgreSQL.

### 6.3 beads-village: Role-Based Routing

The beads-village `status(include_agents=True)` call discovers active agents
and their roles. Leaders assign tasks using `assign(task_id, agent_name)`.
Workers filter their task queue by role: `ls(status="ready", role="fe")`.

### 6.4 Gas Town: The Mayor Pattern

Gas Town (`steveyegge/gastown`) implements a theatrical hierarchy:

| Role | Description |
|---|---|
| **Mayor** | Primary AI orchestrator with full workspace context |
| **Polecats** | Worker agents with persistent identity but ephemeral sessions |
| **Convoys** | Work-tracking units bundling multiple beads per agent |
| **Rigs** | Project containers wrapping git repositories |

The Mayor knows all agents and their current convoys. A `gt convoy list`
gives an instant overview of who is doing what. Discovery is manual (the Mayor
assigns work) rather than dynamic.

Gas Town claims to scale "comfortably to 20–30 agents" vs the 4–10 typically
manageable without orchestration.

---

## 7. Community Coordination Tools

### 7.1 Gas Town (`steveyegge/gastown`)

First-party (Steve Yegge) multi-agent workspace manager. Key patterns:

- Each agent gets a **git worktree** as its isolated workspace
- State survives crashes via git-backed hooks
- Formula-based workflows: TOML templates in `.beads/formulas/`, instantiated
  with `bd mol pour`
- Supports multiple agent runtimes (Claude, Codex, Gemini, Cursor)
- Runtime override per task: `gt sling <bead-id> <rig> --agent cursor`

Installation: `npm install -g gastown` (or `gt install ~/gt --git`).

### 7.2 The Claude Protocol (`AvivK5498/Claude-Code-Beads-Orchestration`)

Opinionated orchestration framework with **13 enforcement hooks that physically
block** bad actions rather than warn. Phases:

1. **Investigation** — orchestrator reads actual source files via Grep/Read/Glob
2. **Planning** — discussion in Plan mode; no code written yet
3. **User Approval** — explicit approval for standard tasks; quick fixes (<10
   lines on feature branches) require approval prompt
4. **Bead Creation** — each approved task becomes a bead; epics for
   cross-domain work with enforced child dependencies
5. **Supervisor Dispatch** — orchestrator delegates via `Task()` to
   tech-stack-specific supervisors
6. **Isolated Worktree Execution** — each bead gets `.worktrees/bd-BD-XXX`;
   supervisor reads bead comments for accumulated context
7. **PR & Merge** — closed beads are immutable; bugs become new linked beads
   via `bd dep relate`

**PostToolUse hook**: dispatch prompts are auto-logged as bead comments, so
supervisors reading a bead inherit full dispatch context without re-investigation.

**Knowledge base**: agents log learned conventions to `.beads/memory/
knowledge.jsonl`; recalled on SessionStart to prevent re-solving known
problems.

### 7.3 BeadHub (`beadhub.ai`)

Commercial coordination server wrapping Beads with real-time features.
Replaces `bd` with `bdh` CLI transparently. Key additions:

| Feature | Mechanism | Storage |
|---|---|---|
| Presence | Redis TTL (auto-expire on crash) | Redis |
| File locks | Redis TTL with release API | Redis |
| Persistent claims | PostgreSQL | PostgreSQL |
| Web dashboard | SSE (Server-Sent Events) | Live |
| Jump-in override | Forcible claim takeover | PostgreSQL |

Communication patterns:
- **Async mail** (`bdh :aweb mail`) — fire-and-forget; delivered on next `bdh`
  command execution
- **Sync chat** (`bdh :aweb chat`) — blocks sender waiting 60s–5min; used only
  when blocked on dependency requiring immediate answer
- **Escalation** — when agents cannot resolve peer-to-peer, human notified via
  dashboard + CLI with full context

Pricing: free tier (1 project, 15 workspaces), Pro $49/mo/project.

### 7.4 beads-village (`lns2905/mcp-beads-village`)

MCP server wrapping Beads + built-in mail system. No external dependencies
(mail stored in `.mail/` directory, git-tracked). 21 MCP tools:

**Core lifecycle:**
```
init() → claim() → reserve(files=[...]) → [work] → done() → RESTART
```

**Identity:**
- Name: `BEADS_AGENT` env var (defaults to `agent-{pid}`)
- Team: `BEADS_TEAM` env var
- Role: `fe` / `be` / `mobile` for task routing

**Communication tools:**
```
msg(to="agent-name", text="...")           # direct message
msg(global=True, to="all", text="...")     # broadcast
inbox()                                     # receive messages
status(include_agents=True)                 # discover active agents
```

**File locking** (`reserve`/`release`/`reservations`) prevents simultaneous
edits with TTL-based deadlock prevention.

Git-native: all state in `.beads/`, `.mail/`, `.reservations/` — no external
services required.

### 7.5 beads-orchestration / beads-compound

From `COMMUNITY_TOOLS.md`:

| Tool | Description |
|---|---|
| `beads-orchestration` | Multi-agent orchestration with tech-specific supervisors (Node.js/Python) |
| `beads-compound` | Claude Code plugin with persistent memory and 28+ agent configurations (Bash/TypeScript) |

These are community implementations of patterns similar to The Claude Protocol
but less opinionated about enforcement hooks.

---

## 8. OpenClaw-Specific Patterns & Gaps

### 8.1 No Existing Integration

As of 2026-02-20, there is no OpenClaw ↔ Beads integration in the community.
Doc 16 confirmed this; additional research found no counter-evidence. This
remains a first-mover opportunity.

For comparison, `opencode-beads` (Joshua Thomas) implements the equivalent for
OpenCode with:
- `bd prime` on SessionStart and post-compaction hooks
- `/bd-*` slash command interface
- `beads-task-agent` subagent for autonomous issue completion

OpenClaw would need analogous integration via a plugin with background services
(see [16-beads-agent-architecture.md §7](16-beads-agent-architecture.md)).

### 8.2 OpenClaw Feature Mapping to Beads Patterns

| OpenClaw mechanism | Beads equivalent | Integration point |
|---|---|---|
| `SOUL.md` / `IDENTITY.md` | `beads.role` git config / `--agent` flag | Identity bridge (see doc 16 §3) |
| `AGENTS.md` | Static capability registry (no dynamic equivalent in Beads) | Populate manually or via PRIME.md |
| `sessions_spawn` / `sessions_send` | Molecule dispatch via `bd claim` + wake-up | Tier 4 smart dispatch (see doc 16 §2.3) |
| OpenClaw cron | Tier 1 beads-watcher shell script | `*/2 * * * *` system cron |
| Heartbeat | Tier 3: Beads check as one item in HEARTBEAT.md | Share heartbeat session with other checks |
| Plugin background service | Tier 1 alternative with tighter lifecycle coupling | `src/plugins/services.ts` |

### 8.3 PRIME.md for OpenClaw Agent Routing

An OpenClaw-specific `.beads/PRIME.md` could encode the routing table:

```markdown
## Agent Routing

- Alice (agentId: alice): Frontend — React, TypeScript, UI components
- Bob (agentId: bob): Infrastructure — Ansible, Terraform, Docker
- Carol (agentId: carol): Backend — Node.js, PostgreSQL, APIs

Label convention:
- `fe` → assign to alice
- `infra` → assign to bob
- `be` → assign to carol
- `review` → any agent

Current sprint focus: Authentication system (bd-a3f8)
```

This gives every agent session immediate routing awareness at ~200 tokens.

### 8.4 Knowledge Base Pattern (from Claude Protocol)

Adapt the Claude Protocol's `.beads/memory/knowledge.jsonl` for OpenClaw:

```bash
# Agent appends learning after encountering a pattern:
bd comment bd-a1b2 "KNOWLEDGE: Tailscale ACL changes require 2min propagation
before testing. Factor into smoke test timing."

# Knowledge base skill reads and surfaces these at session start:
bd list --label knowledge --status open --json | head -20
```

This prevents re-solving known problems across agents and sessions.

### 8.5 Recommended OpenClaw Integration Stack (Priority Order)

Based on all research:

| Layer | Tool | Rationale |
|---|---|---|
| Task tracking | Native `bd` CLI | Zero deps, universal, proven |
| Context injection | `bd prime` via SessionStart hook in plugin | 1-2k token overhead, workspace-specific via PRIME.md |
| Polling | Tier 1 system cron watcher (see doc 16 §4.2) | Zero tokens, decoupled |
| File conflict prevention | `bd update --claim` (native) | Already in core Beads |
| Broadcast messaging | `bd mail` (native) | Git-backed, no external services |
| Real-time dashboards | `beads-dashboard` or `bdui` (community) | Observability without extra infrastructure |
| Multi-agent orchestration | Gas Town or The Claude Protocol patterns | Adopt workflow patterns, not necessarily the tools |
| Enterprise coordination | BeadHub | Only if presence + file locking become pain points |

---

## 9. Summary: Key Patterns Reference

### Coordination Decision Tree

```
Need to coordinate with another agent?
│
├─ Async status update / review request / handoff note
│   └─ bd mail send --to <agent> "..."
│
├─ Progress/blocker note on current task
│   └─ bd comment <bead-id> "PROGRESS/BLOCKED/HANDOFF: ..."
│
├─ Blocking another agent's work explicitly
│   └─ bd dep add <blocked-bead> <blocker-bead> --type blocks
│       + bd update <blocked-bead> --status blocked
│
├─ Need human approval to proceed
│   └─ bd create --type gate --await-type human "Approval needed: ..."
│
├─ Wait for CI/PR before proceeding
│   └─ bd create --type gate --await-type gh:run --await-id <workflow>
│
└─ Need immediate answer (sync, blocking your session)
    └─ OpenClaw sessions_send to other agent
        (not Beads-native; only when truly sync-required)
```

### Agent Session Checklist

```
START:
  bd prime                     # get workflow context (~1-2k tokens)
  bd ready --json              # select unblocked work

DURING:
  bd update <id> --claim       # claim before starting
  bd comment <id> "PROGRESS:"  # log meaningful milestones
  bd create <new-bead>         # file discoveries immediately
  bd dep add ...               # add blocking edges when discovered

END:
  bd comment <id> "HANDOFF: [full context for next session]"
  bd update <id> --status done (or blocked)
  bd sync                      # commits + pushes (mandatory)
```

---

## References

### Beads Repository

- [steveyegge/beads](https://github.com/steveyegge/beads) — main repo
- `docs/MOLECULES.md` — molecules, protos, mols, wisps, bonding, gates
- `docs/CLAUDE_INTEGRATION.md` — bd prime, SessionStart, PreCompact, PRIME.md
- `docs/COMMUNITY_TOOLS.md` — full tool ecosystem listing
- `AGENT_INSTRUCTIONS.md` — agent workflow rules, landing-the-plane protocol
- `docs/ADVANCED.md` — database redirects, multi-agent dolt server mode
- GitHub Issue #499 — bd prime vs TodoWrite conflict resolution
- GitHub Issue #1637 — bd prime staleness detection proposal

### Community Tools

- [steveyegge/gastown](https://github.com/steveyegge/gastown) — Gas Town multi-agent workspace manager
- [AvivK5498/Claude-Code-Beads-Orchestration](https://github.com/AvivK5498/Claude-Code-Beads-Orchestration) — The Claude Protocol (13 enforcement hooks)
- [lns2905/mcp-beads-village](https://github.com/lns2905/mcp-beads-village) — beads-village MCP server with mail + file locking
- [BeadHub](https://beadhub.ai/) — commercial coordination server (Redis + PostgreSQL)
- [joshuadavidthomas/opencode-beads](https://github.com/joshuadavidthomas/opencode-beads) — OpenCode integration reference

### DeepWiki Analysis

- [AI Agent Integration](https://deepwiki.com/steveyegge/beads/8-ai-agent-integration) — MCP, hooks, gates, actor attribution
- [Claude Plugin and Editor Integration](https://deepwiki.com/steveyegge/beads/9.2-claude-plugin) — bd prime token models, gate patterns

### Related Exploration Docs

- [architecture.md](architecture.md) — Four-Tier token saving, identity bridging, notification options, implementation phases
- [agent-to-agent.md](../agents/agent-to-agent.md) — OpenClaw native A2A mechanisms
- [process-execution.md](../architecture/process-execution.md) — zero-token event-driven wake-up
- [architecture-overview.md](../architecture/architecture-overview.md) — SOUL.md, IDENTITY.md, AGENTS.md identity system
