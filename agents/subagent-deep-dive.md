# 06 — Subagent Deep Dive: Skills, Context & Bootstrapping

## Question

When a subagent is spawned via `sessions_spawn`, what exactly does it inherit from the main agent? Specifically:

1. Does it get all skills, or can we control which skills it sees?
2. Does it get SOUL.md, USER.md, IDENTITY.md and the other bootstrap files?
3. Does it have its own system prompt, or just the task message?

## TL;DR

| Aspect | Main Agent | Subagent |
|---|---|---|
| System prompt mode | `"full"` | `"minimal"` |
| Skills section | Yes (all eligible skills listed) | **No** (stripped) |
| SOUL.md | Yes | **No** |
| USER.md | Yes | **No** |
| IDENTITY.md | Yes | **No** |
| HEARTBEAT.md | Yes | **No** |
| BOOTSTRAP.md | Yes | **No** |
| AGENTS.md | Yes | **Yes** |
| TOOLS.md | Yes | **Yes** |
| Memory section | Yes | **No** |
| User Identity section | Yes | **No** |
| Messaging section | Yes | **No** |
| Reply Tags | Yes | **No** |
| Docs section | Yes | **No** |
| Model Aliases | Yes | **No** |
| Silent Replies | Yes | **No** |
| Heartbeats | Yes | **No** |
| Safety section | Yes | Yes |
| Tooling section | Yes | Yes |
| Workspace section | Yes | Yes |
| Runtime section | Yes | Yes |
| Subagent Context (extra) | No | **Yes** (injected) |

**In short: Subagents get a stripped-down system prompt, only AGENTS.md + TOOLS.md as bootstrap context, no Skills section, and a dedicated "Subagent Context" block that defines their role and constraints.** (See also: `docs/concepts/system-prompt.md` — "Prompt modes" and "Workspace bootstrap injection"; `docs/tools/subagents.md` — "Limitations")

---

## 1. System Prompt: `promptMode = "minimal"`

When the embedded Pi runner builds the system prompt for a session, it checks the session key:

```typescript
// src/agents/pi-embedded-runner/run/attempt.ts:425-428
const promptMode =
  isSubagentSessionKey(params.sessionKey) || isCronSessionKey(params.sessionKey)
    ? "minimal"
    : "full";
```

The session key for a subagent follows the pattern `agent:<agentId>:subagent:<uuid>` (see `src/agents/subagent-spawn.ts:148`), which `isSubagentSessionKey()` recognizes.

In `"minimal"` mode, `buildAgentSystemPrompt()` (in `src/agents/system-prompt.ts`) skips many sections. The `isMinimal` flag gates each section:

```typescript
// src/agents/system-prompt.ts:358
const isMinimal = promptMode === "minimal" || promptMode === "none";
```

### Sections included in "minimal" mode

- **Tooling** — Full tool list with summaries (same tools as main agent, subject to sandbox policy)
- **Safety** — Always included
- **Workspace** — Working directory and guidance
- **Runtime** — Host, OS, model, channel info
- **Subagent Context** — Injected via `extraSystemPrompt` parameter

### Sections EXCLUDED in "minimal" mode

- **Skills (mandatory)** — `buildSkillsSection()` returns `[]` when `isMinimal`
- **Memory Recall** — `buildMemorySection()` returns `[]` when `isMinimal`
- **User Identity** — `buildUserIdentitySection()` returns `[]` when `isMinimal`
- **Reply Tags** — `buildReplyTagsSection()` returns `[]` when `isMinimal`
- **Messaging** — `buildMessagingSection()` returns `[]` when `isMinimal`
- **Voice (TTS)** — `buildVoiceSection()` returns `[]` when `isMinimal`
- **Documentation** — `buildDocsSection()` returns `[]` when `isMinimal`
- **Model Aliases** — Gated by `!isMinimal`
- **OpenClaw Self-Update** — Gated by `!isMinimal`
- **Silent Replies** — Gated by `!isMinimal`
- **Heartbeats** — Gated by `!isMinimal`

Source: `src/agents/system-prompt.ts:17-638`

## 2. Bootstrap Files: Only AGENTS.md and TOOLS.md

Bootstrap files are the workspace markdown files injected into the system prompt as "Project Context". The full set loaded from the workspace is:

1. `AGENTS.md`
2. `SOUL.md`
3. `TOOLS.md`
4. `IDENTITY.md`
5. `USER.md`
6. `HEARTBEAT.md`
7. `BOOTSTRAP.md`
8. `MEMORY.md` (and `memory/*.md`)

Source: `src/agents/workspace.ts:412-466`

However, before injection, they are filtered by `filterBootstrapFilesForSession()`:

```typescript
// src/agents/workspace.ts:468-478
const MINIMAL_BOOTSTRAP_ALLOWLIST = new Set([
  DEFAULT_AGENTS_FILENAME,  // "AGENTS.md"
  DEFAULT_TOOLS_FILENAME,   // "TOOLS.md"
]);

export function filterBootstrapFilesForSession(
  files: WorkspaceBootstrapFile[],
  sessionKey?: string,
): WorkspaceBootstrapFile[] {
  if (!sessionKey || (!isSubagentSessionKey(sessionKey) && !isCronSessionKey(sessionKey))) {
    return files;
  }
  return files.filter((file) => MINIMAL_BOOTSTRAP_ALLOWLIST.has(file.name));
}
```

**Subagents only see `AGENTS.md` and `TOOLS.md`** — no SOUL.md, no USER.md, no IDENTITY.md, no MEMORY.md, no BOOTSTRAP.md, no HEARTBEAT.md.

This is called from `resolveBootstrapFilesForRun()` in `src/agents/bootstrap-files.ts:25-46` (which in turn is called by `resolveBootstrapContextForRun()` at `src/agents/bootstrap-files.ts:48-66`).

## 3. Skills: Not in System Prompt, But Still Loaded from Workspace

This is a subtle point. There are two separate mechanisms:

### 3a. Skills Prompt in System Prompt → EXCLUDED

The skills prompt (the `<available_skills>` block that tells the model which skills exist) is **not injected** for subagents because `buildSkillsSection()` returns `[]` when `isMinimal` is true.

The subagent's system prompt contains no "Skills (mandatory)" section. The model has no knowledge of available skills.

### 3b. Skills on Disk → STILL PRESENT

The subagent uses the same workspace directory (or the agent's configured workspace). Skills are physically present on disk in the `skills/` directory. If a sandbox is involved, `syncSkillsToWorkspace()` (`src/agents/skills/workspace.ts:606`) copies all skills into the sandbox workspace.

However, since the model doesn't know about them (no skills prompt), it won't proactively read any SKILL.md files.

### 3c. Per-Agent Skill Filtering (for non-subagent use)

Agents defined in `openclaw.json` can have a `skills` array that acts as an allowlist:

```typescript
// src/config/types.agents.ts:28-29
/** Optional allowlist of skills for this agent (omit = all skills; empty = none). */
skills?: string[];
```

This is resolved via `resolveAgentSkillsFilter()` in `src/agents/agent-scope.ts:128-133`:

```typescript
export function resolveAgentSkillsFilter(
  cfg: OpenClawConfig,
  agentId: string,
): string[] | undefined {
  return normalizeSkillFilter(resolveAgentConfig(cfg, agentId)?.skills);
}
```

When the `agent` command builds a skills snapshot, it passes this filter:

```typescript
// src/commands/agent.ts:317-324
const skillFilter = resolveAgentSkillsFilter(cfg, sessionAgentId);
const skillsSnapshot = needsSkillsSnapshot
  ? buildWorkspaceSkillSnapshot(workspaceDir, {
      config: cfg,
      eligibility: { remote: getRemoteSkillEligibility() },
      snapshotVersion: skillsSnapshotVersion,
      skillFilter,
    })
  : sessionEntry?.skillsSnapshot;
```

The filter works like this:
- `undefined` (not set) = all eligible skills included
- `[]` (empty array) = no skills at all
- `["skill-a", "skill-b"]` = only these skills

**But this filtering applies to the agent's own session, not specifically to its subagents.** Since subagents get `promptMode: "minimal"` which strips the skills section entirely, the skill filter is effectively irrelevant for subagents.

## 4. The Subagent System Prompt

When `spawnSubagentDirect()` creates a subagent, it builds a dedicated system prompt via `buildSubagentSystemPrompt()`:

```typescript
// src/agents/subagent-spawn.ts:230-238
const childSystemPrompt = buildSubagentSystemPrompt({
  requesterSessionKey,
  requesterOrigin,
  childSessionKey,
  label: label || undefined,
  task,
  childDepth,
  maxSpawnDepth,
});
```

This prompt is injected as `extraSystemPrompt` in the gateway call (line 259), which gets appended under `## Subagent Context` in the system prompt.

The subagent system prompt (built in `src/agents/subagent-announce.ts:595-682`) contains:

1. **Role definition** — "You are a subagent spawned by the main agent for a specific task"
2. **Task description** — The exact task string from the requester
3. **Rules** — Stay focused, complete the task, don't initiate, be ephemeral, trust push-based completion
4. **Output format** — What to include in the final response
5. **Prohibitions** — No user conversations, no external messages, no cron, no pretending to be the main agent
6. **Sub-Agent Spawning** (conditional) — If depth allows further nesting, instructions for using `sessions_spawn`
7. **Session Context** — Label, requester session key, requester channel, child session key

### The Task Message

In addition to the system prompt, the subagent receives its first user message:

```typescript
// src/agents/subagent-spawn.ts:239-242
const childTaskMessage = [
  `[Subagent Context] You are running as a subagent (depth ${childDepth}/${maxSpawnDepth}).
   Results auto-announce to your requester; do not busy-poll for status.`,
  `[Subagent Task]: ${task}`,
].join("\n\n");
```

## 5. Depth Control and Nesting

<!-- See also: docs/tools/subagents.md § "Nested Sub-Agents" -->

Subagent spawning is depth-limited:

```typescript
// src/agents/subagent-spawn.ts:109-116
const callerDepth = getSubagentDepthFromSessionStore(requesterInternalKey, { cfg });
const maxSpawnDepth = cfg.agents?.defaults?.subagents?.maxSpawnDepth ?? 1;
if (callerDepth >= maxSpawnDepth) {
  return {
    status: "forbidden",
    error: `sessions_spawn is not allowed at this depth (current depth: ${callerDepth}, max: ${maxSpawnDepth})`,
  };
}
```

Default `maxSpawnDepth` is **1**, meaning only the main agent can spawn subagents (depth 0 → 1). To allow sub-sub-agents, you'd set `maxSpawnDepth: 2` or higher.

There's also a per-session children limit:

```typescript
// src/agents/subagent-spawn.ts:118-125
const maxChildren = cfg.agents?.defaults?.subagents?.maxChildrenPerAgent ?? 5;
const activeChildren = countActiveRunsForSession(requesterInternalKey);
if (activeChildren >= maxChildren) {
  return { status: "forbidden", ... };
}
```

## 6. Agent Allowlist for Cross-Agent Spawning

<!-- See also: docs/tools/subagents.md § "Allowlist" -->

When a subagent requests a different `agentId` than its own, the system checks `subagents.allowAgents`:

```typescript
// src/agents/subagent-spawn.ts:131-147
if (targetAgentId !== requesterAgentId) {
  const allowAgents = resolveAgentConfig(cfg, requesterAgentId)
    ?.subagents?.allowAgents ?? [];
  const allowAny = allowAgents.some((value) => value.trim() === "*");
  // ...
  if (!allowAny && !allowSet.has(normalizedTargetId)) {
    return { status: "forbidden", ... };
  }
}
```

To allow an agent to spawn subagents as a different agent, you must configure `subagents.allowAgents` in the agent's config entry. `"*"` allows spawning any agent.

## 7. Model Override for Subagents

<!-- See also: docs/tools/subagents.md § "Tool" (model/thinking params) -->

The spawner can request a specific model for the subagent:

```typescript
// src/agents/subagent-spawn.ts:152-156
const resolvedModel = resolveSubagentSpawnModelSelection({
  cfg,
  agentId: targetAgentId,
  modelOverride,
});
```

If provided, the model is applied via `sessions.patch` before the run starts. There's also a thinking-level override (`thinking` parameter), which can be set per-spawn or configured as a default in the agent or global config. (Config: `agents.defaults.subagents.model`, `agents.defaults.subagents.thinking`, or per-agent `agents.list[].subagents.model`/`.thinking` — see `docs/tools/subagents.md` § "Tool" and `docs/gateway/configuration-reference.md`)

## Summary Diagram

```
┌──────────────────────────────────────────────────────────────┐
│                     MAIN AGENT                                │
│  promptMode: "full"                                           │
│                                                               │
│  System Prompt:                                               │
│  ├── Tooling (all tools)                                      │
│  ├── Safety                                                   │
│  ├── Skills (mandatory) ◄── full skills list                  │
│  ├── Memory Recall                                            │
│  ├── User Identity                                            │
│  ├── Messaging + Reply Tags                                   │
│  ├── Model Aliases                                            │
│  ├── Workspace                                                │
│  ├── Documentation                                            │
│  ├── Silent Replies                                           │
│  ├── Heartbeats                                               │
│  ├── Runtime                                                  │
│  └── Project Context:                                         │
│      ├── AGENTS.md                                            │
│      ├── SOUL.md                                              │
│      ├── TOOLS.md                                             │
│      ├── IDENTITY.md                                          │
│      ├── USER.md                                              │
│      ├── HEARTBEAT.md                                         │
│      ├── BOOTSTRAP.md                                         │
│      └── MEMORY.md + memory/*.md                              │
│                                                               │
│  sessions_spawn({ task: "...", agentId: "coder" })            │
│     │                                                         │
└─────┼────────────────────────────────────────────────────────┘
      ▼
┌──────────────────────────────────────────────────────────────┐
│                     SUBAGENT                                  │
│  promptMode: "minimal"                                        │
│  sessionKey: agent:coder:subagent:<uuid>                      │
│                                                               │
│  System Prompt:                                               │
│  ├── Tooling (same tools, sandbox-filtered)                   │
│  ├── Safety                                                   │
│  ├── Workspace                                                │
│  ├── Runtime                                                  │
│  ├── Subagent Context: ◄── injected via extraSystemPrompt     │
│  │   ├── Role: "handle: <task>"                               │
│  │   ├── Rules: stay focused, complete task, be ephemeral     │
│  │   ├── Output format guidance                               │
│  │   ├── Prohibitions (no user chat, no cron, no external)    │
│  │   ├── Sub-agent spawning (if depth allows)                 │
│  │   └── Session context (label, requester, child key)        │
│  └── Project Context:                                         │
│      ├── AGENTS.md                                            │
│      └── TOOLS.md                                             │
│                                                               │
│  First User Message:                                          │
│  "[Subagent Context] depth 1/1 ..."                           │
│  "[Subagent Task]: <task text>"                               │
│                                                               │
│  ✗ No Skills section                                          │
│  ✗ No SOUL.md, USER.md, IDENTITY.md                           │
│  ✗ No Memory, Messaging, Heartbeat                            │
└──────────────────────────────────────────────────────────────┘
```

## Key Takeaways for B4Racing

> Cross-ref: `docs/tools/subagents.md` covers the user-facing subagent docs end-to-end (tool params, nesting, announce, tool policy, concurrency, limitations). `docs/concepts/system-prompt.md` covers prompt modes and bootstrap injection.

1. **Skills are invisible to subagents** — The coder subagent won't know about skills unless we explicitly mention them in the task string or TOOLS.md.

2. **SOUL.md persona is lost** — If SOUL.md defines a personality, subagents won't have it. This is by design (subagents are workers, not personalities).

3. **AGENTS.md is inherited** — This is important for us because AGENTS.md typically contains project-level instructions (coding standards, git workflow, etc.).

4. **TOOLS.md is inherited** — Custom tool guidance survives into subagents.

5. **Per-agent skill filtering exists** but is irrelevant for subagents since the skills section is stripped entirely in minimal mode.

6. **To give a subagent specific instructions**, the only levers are:
   - The `task` string in `sessions_spawn`
   - Content in `AGENTS.md` (inherited)
   - Content in `TOOLS.md` (inherited)
   - The agent's workspace directory (can be different per agentId)

7. **Depth defaults to 1** — Our coder subagent cannot spawn further subagents unless we raise `maxSpawnDepth`.

---

## References

### Source Files

| File | What it covers |
|---|---|
| `src/agents/pi-embedded-runner/run/attempt.ts:425-428` | `promptMode` selection (`"minimal"` for subagents) |
| `src/agents/system-prompt.ts:17-638` | `buildAgentSystemPrompt()` and all `isMinimal`-gated section builders |
| `src/agents/system-prompt.ts:357-358` | `isMinimal` flag derivation |
| `src/agents/workspace.ts:412-466` | `loadWorkspaceBootstrapFiles()` — full bootstrap file set |
| `src/agents/workspace.ts:468-478` | `MINIMAL_BOOTSTRAP_ALLOWLIST` and `filterBootstrapFilesForSession()` |
| `src/agents/bootstrap-files.ts:25-46` | `resolveBootstrapFilesForRun()` — calls `filterBootstrapFilesForSession` |
| `src/agents/bootstrap-files.ts:48-66` | `resolveBootstrapContextForRun()` — entry point used by the runner |
| `src/agents/subagent-spawn.ts:109-116` | Depth limit enforcement |
| `src/agents/subagent-spawn.ts:118-125` | Per-session `maxChildrenPerAgent` limit |
| `src/agents/subagent-spawn.ts:131-147` | Cross-agent `allowAgents` check |
| `src/agents/subagent-spawn.ts:148` | Child session key format (`agent:<id>:subagent:<uuid>`) |
| `src/agents/subagent-spawn.ts:152-156` | Model override resolution |
| `src/agents/subagent-spawn.ts:230-242` | `buildSubagentSystemPrompt()` call and `childTaskMessage` construction |
| `src/agents/subagent-spawn.ts:259` | `extraSystemPrompt` injection into gateway call |
| `src/agents/subagent-announce.ts:595-682` | `buildSubagentSystemPrompt()` — full subagent system prompt builder |
| `src/agents/agent-scope.ts:128-132` | `resolveAgentSkillsFilter()` |
| `src/config/types.agents.ts:29` | `skills?: string[]` type definition on agent config |
| `src/commands/agent.ts:317-325` | Skill filter wiring into `buildWorkspaceSkillSnapshot()` |

### Official Documentation (cross-references)

| Doc | Relevance |
|---|---|
| `docs/tools/subagents.md` | Canonical sub-agent reference: slash commands, tool params, nesting, announce, tool policy, concurrency, limitations |
| `docs/concepts/system-prompt.md` | System prompt structure, prompt modes (`full`/`minimal`/`none`), workspace bootstrap injection, skills injection |
| `docs/concepts/multi-agent.md` | Multi-agent routing, agent isolation (workspace, agentDir, sessions), per-agent sandbox and tool config |
| `docs/concepts/session-tool.md` | Session tool names (`sessions_list`, `sessions_history`, `sessions_send`, `sessions_spawn`) and key model |
| `docs/gateway/configuration-reference.md` | Full config reference including `agents.defaults.subagents.*` fields |
