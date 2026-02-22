# Subagent Skills & Practical Example B4Racing

## 1. Skill Inheritance for Subagents

### 1.1 Mechanism

Subagents do **not load skills themselves** – they inherit the **skill snapshot of the parent agent**.

```
Parent session starts
  → ensureSkillSnapshot() builds snapshot (with parent's skill filter)
  → Snapshot contains: resolved skills + prompt text + env overrides

Parent spawns subagent via sessions_spawn()
  → followupRun.run.skillsSnapshot = Parent's snapshot (passed through)

Subagent starts
  → Has skillsSnapshot? → Yes → use directly (no reloading)
  → resolveSkillsPromptForRun() returns snapshot prompt
```

Sources:
- `src/auto-reply/reply/get-reply-run.ts:415` – Snapshot is set in the run object passed to the agent runner
- `src/agents/pi-embedded-runner/run/attempt.ts:251-266` – Subagent checks for existing snapshot and resolves skills prompt from it
- `src/auto-reply/reply/session-updates.ts:166-174` – Snapshot creation on first turn via `buildWorkspaceSkillSnapshot()`
- `src/agents/skills/workspace.ts:536-545` – `resolveSkillsPromptForRun()` returns snapshot prompt directly if available
- `src/gateway/server-methods/agent.ts:392` – `skillsSnapshot` is carried over on session entry for agent delivery

### 1.2 Detailed Behavior

| Aspect | Behavior |
|---|---|
| **Inheritance** | Subagent receives parent's filtered snapshot as a one-time copy |
| **Own skill filter?** | No – the subagent's agent config is NOT used for skill filtering |
| **Timing** | Snapshot is created on the parent's first turn, then frozen |
| **Override on spawn?** | Not possible – `sessions_spawn()` has no parameter for skill filtering |
| **Refresh** | Subagent does not receive updates if skills change during the session |

### 1.3 Per-Agent Skill Filter (parent only)

A skill filter can be defined in the agent config:

```typescript
// src/agents/agent-scope.ts:128-133
export function resolveAgentSkillsFilter(
  cfg: OpenClawConfig,
  agentId: string,
): string[] | undefined {
  return normalizeSkillFilter(resolveAgentConfig(cfg, agentId)?.skills);
}
```

This filter is applied during snapshot construction of the **parent agent** (`ensureSkillSnapshot()`).
It is **not** re-evaluated for the subagent – the subagent inherits the already filtered snapshot.

The filter is called from:
- `src/auto-reply/reply/get-reply.ts:69` – gateway reply path passes it to `ensureSkillSnapshot()`
- `src/commands/agent.ts:317` – CLI agent command passes it to `buildWorkspaceSkillSnapshot()`
- `src/cron/isolated-agent/skills-snapshot.ts:21` – cron agent uses it for snapshot construction

### 1.4 Consequence

Subagents receive **all of the parent's skills** (after parent filtering). There is no mechanism to give a subagent a reduced skill set. This wastes prompt space, but is usually not a functional problem because the subagent is steered via its `task` description in the `sessions_spawn()` call.

---

## 2. Practical Example: B4Racing Setup

Path: `/Users/mhild/src/durandom/racing/b4racing/apps/openclaw/`

### 2.1 Agent Configuration

```json
{
  "agents": {
    "defaults": {
      "model": { "primary": "anthropic/claude-opus-4-6" },
      "subagents": {
        "maxConcurrent": 1,
        "maxSpawnDepth": 1,
        "maxChildrenPerAgent": 5
      }
    },
    "list": [
      {
        "id": "main",
        "default": true,
        "subagents": { "allowAgents": ["coder"] }
      },
      { "id": "coder" }
    ]
  }
}
```

- **main**: Receives Telegram messages, orchestrates work via skills
- **coder**: Only started via `sessions_spawn({agentId: "coder"})` (`src/agents/tools/sessions-spawn-tool.ts:8-18` – tool schema definition; `src/agents/subagent-spawn.ts:20-29` – spawn params type)

**Subagent Constraints:**
- `maxConcurrent: 1` – one coder at a time (queue) (`src/config/agent-limits.ts:14-19` – resolved via `resolveSubagentMaxConcurrent()`, default: 8)
- `maxSpawnDepth: 1` – no recursive spawning (main → coder, but not coder → coder) (`src/agents/subagent-spawn.ts:110-114` – depth check)
- `maxChildrenPerAgent: 5` – max active children per agent session (`src/agents/subagent-spawn.ts:118` – enforced before spawn)
- `allowAgents: ["coder"]` – main may only spawn "coder" (`src/agents/subagent-spawn.ts:132-136` – allowlist enforcement; `src/config/types.agents.ts:39` – type definition)

### 2.2 Skill Configuration

**Bundled Skills (Allowlist):**

```json
{
  "skills": {
    "allowBundled": [
      "github", "gh-issues", "healthcheck", "model-usage",
      "session-logs", "clawhub", "skill-creator", "summarize", "tmux"
    ],
    "load": { "extraDirs": ["/home/node/.openclaw/skills"] }
  }
}
```

9 out of 50+ bundled skills enabled – the rest are ignored.
(`src/config/types.skills.ts:40` – `allowBundled` type; `src/agents/skills/config.ts:38-50` – allowlist normalization and bundled source filtering)

**Custom Skills (via extraDirs):** (`src/config/types.skills.ts:13` – `extraDirs` type definition)

| Skill | Role | Used by |
|---|---|---|
| `b4racing-dev` | Orchestrator – spawns coder for code work | main |
| `b4racing-triage` | Issue triage, labeling, duplicate detection | main |
| `b4racing-monitor` | CI/PR monitoring, alerts, auto-fix | main |
| `coder` | 4-phase dispatch worker | coder (subagent) |

### 2.3 Skill Inheritance in the B4Racing Context

Since no per-subagent skill filter exists, the **coder subagent receives all skills of the main agent**:
- 9 bundled skills (github, gh-issues, healthcheck, ...)
- 4 custom skills (b4racing-dev, b4racing-triage, b4racing-monitor, coder)

The coder actually only needs the `coder` skill. The other skills are never triggered because the coder is steered via the `task` description in `sessions_spawn()`.

### 2.4 Coder: 4-Phase Protocol

```
1. Pre-Flight    → Clean up stale worktrees, clone repo, create worktree (/tmp/w-<ISSUE>)
2. Dispatch      → `uv run b4racing-dispatch worker run <ISSUE> --max-budget-usd 5` (blocking)
3. Post-Flight   → Read issue status via `gh issue view`, find PR, determine result
4. Cleanup       → Remove worktree, report result (always, even on errors)
```

Each issue gets an isolated git worktree in `/tmp/w-<ISSUE>`.

### 2.5 Infrastructure

**Docker Compose** with three services:
1. **seed-config** (init) – stages config files into named volume
2. **openclaw-gateway** – custom image with gh, claude, uv, just, gh-token
3. **signal-cli** (optional) – Signal integration

**GitHub App Auth:**
- `gh` wrapper at `/usr/local/bin/gh` automatically injects app tokens
- Token cache at `/tmp/gh-app-token.json` (refresh 5 min before expiry)
- Git credential helper delegates to `gh auth git-credential`

**Channels:**
- Telegram (dmPolicy: `allowlist`)
- Signal (optional, dmPolicy: `pairing`)

### 2.6 `allowAgents` vs. Skill Filter

Important distinction:

| Mechanism | Controls | Scope | Source |
|---|---|---|---|
| `allowAgents: ["coder"]` | Which agents may be spawned | Subagent spawning | `src/config/types.agents.ts:39`, `src/agents/subagent-spawn.ts:132` |
| `skills.allowBundled: [...]` | Which bundled skills are loaded | All agents (global) | `src/config/types.skills.ts:40` |
| Agent config `skills: [...]` | Which skills an agent has in its snapshot | Per agent (parent) | `src/agents/agent-scope.ts:128-133` |

There is **no** mechanism for "this subagent only gets these skills".

---

## References

### Source files

| File | What it covers |
|---|---|
| `src/agents/subagent-spawn.ts` | Core subagent spawn logic: depth check, children limit, allowAgents enforcement |
| `src/agents/tools/sessions-spawn-tool.ts` | `sessions_spawn` tool schema and execution |
| `src/agents/agent-scope.ts:128-133` | `resolveAgentSkillsFilter()` – per-agent skill filter resolution |
| `src/agents/skills/workspace.ts:536-545` | `resolveSkillsPromptForRun()` – uses snapshot prompt if available |
| `src/agents/pi-embedded-runner/run/attempt.ts:251-266` | Subagent runner: checks for existing snapshot, applies skill env, resolves prompt |
| `src/auto-reply/reply/session-updates.ts:114-232` | `ensureSkillSnapshot()` – builds and persists skills snapshot |
| `src/auto-reply/reply/get-reply-run.ts:268-281,415` | Calls `ensureSkillSnapshot()`, passes `skillsSnapshot` into run object |
| `src/auto-reply/reply/get-reply.ts:69` | Passes `resolveAgentSkillsFilter()` result as `skillFilter` |
| `src/gateway/server-methods/agent.ts:392` | Carries `skillsSnapshot` on agent delivery session entry |
| `src/config/types.agents.ts:37-42` | Agent config type: `subagents.allowAgents`, `subagents.model` |
| `src/config/types.agent-defaults.ts:245-247` | `maxSpawnDepth`, `maxChildrenPerAgent` type definitions |
| `src/config/types.skills.ts` | `SkillsConfig` type: `allowBundled`, `load.extraDirs`, `entries` |
| `src/config/agent-limits.ts` | `resolveSubagentMaxConcurrent()` – default 8 |
| `src/agents/skills/config.ts:38-50` | Bundled skill allowlist normalization |

### Official docs (cross-references)

| Doc | Relevant topics |
|---|---|
| `docs/tools/subagents.md` | Sub-agent tool (`sessions_spawn`), depth levels, allowlist, concurrency, tool policy, announce, auto-archive |
| `docs/tools/skills.md` | Skills system overview, locations/precedence, snapshot behavior, `allowBundled`, per-agent vs shared skills, token impact |
| `docs/tools/skills-config.md` | Skills config schema: `allowBundled`, `load.extraDirs`, per-skill entries, sandboxed env |
| `docs/concepts/multi-agent.md` | Multi-agent routing, per-agent workspaces/skills, bindings |
| `docs/cli/skills.md` | CLI commands for inspecting skill eligibility |
