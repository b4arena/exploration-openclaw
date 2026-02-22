# 09 — Agent-to-Agent Communication

## Question

Should agents with different "souls" (personas/system prompts) communicate via external channels (like Matrix rooms) rather than internal subagent spawning? What patterns does OpenClaw support, and when should you use which?

## TL;DR

OpenClaw provides **three** distinct mechanisms for multi-agent interaction:

| Mechanism | Communication Path | Isolation | Persona Preservation | Use Case |
|---|---|---|---|---|
| **Subagent spawn** (`sessions_spawn`) | Internal, ephemeral | Partial (shared tools, stripped persona) | No (minimal prompt mode) | Task delegation to workers |
| **Agent-to-agent send** (`sessions_send`) | Internal, persistent sessions | Full (separate workspace, sessions, auth) | Yes (full prompt mode, own SOUL.md) | Cross-persona collaboration |
| **Channel-based** (Matrix/Telegram rooms) | External, via messaging protocol | Full + network isolation | Yes (full prompt mode, own SOUL.md) | Observable multi-persona interaction |

**Recommendation:** Use `sessions_send` with `tools.agentToAgent` for programmatic inter-agent communication. Use channel-based (Matrix rooms) when you want humans to observe and participate in multi-agent conversations. Use `sessions_spawn` only for fire-and-forget worker tasks where persona does not matter.

---

## 1. Internal Communication: Subagent Spawn

### How It Works

`sessions_spawn` creates an ephemeral child session with `promptMode: "minimal"`. The subagent gets a stripped-down system prompt: no SOUL.md, no USER.md, no Skills section, no Memory. It inherits only AGENTS.md and TOOLS.md.

```typescript
// src/agents/pi-embedded-runner/run/attempt.ts:425-428
const promptMode =
  isSubagentSessionKey(params.sessionKey) || isCronSessionKey(params.sessionKey)
    ? "minimal"
    : "full";
```

The child session key follows the pattern `agent:<agentId>:subagent:<uuid>` (`src/agents/subagent-spawn.ts:148`).

### Key Properties

- **Non-blocking:** Returns `{ status: "accepted", runId, childSessionKey }` immediately (`docs/concepts/session-tool.md`)
- **Depth-limited:** Default `maxSpawnDepth: 1` — only the main agent can spawn (`src/agents/subagent-spawn.ts:109-116`)
- **Children-limited:** Default `maxChildrenPerAgent: 5` (`src/agents/subagent-spawn.ts:118-125`)
- **Cross-agent spawning:** Requires `subagents.allowAgents` allowlist (`src/agents/subagent-spawn.ts:131-147`)
- **Auto-archived:** After `agents.defaults.subagents.archiveAfterMinutes` (default: 60) (`docs/concepts/session-tool.md`)
- **Announce:** After completion, result is posted to the requester's chat channel (`src/agents/subagent-announce.ts:595-682`)

### Limitations for Multi-Persona Use

Subagents are **workers, not personas**. They lose:
- SOUL.md (personality)
- USER.md (user context)
- IDENTITY.md (identity)
- Memory section
- Skills section
- Messaging section

Source: `src/agents/workspace.ts:468-478` — `MINIMAL_BOOTSTRAP_ALLOWLIST` only includes AGENTS.md and TOOLS.md.

**Verdict:** Not suitable for "agents with different souls talking to each other." The persona is stripped by design.

---

## 2. Internal Communication: Agent-to-Agent Send

### How It Works

`sessions_send` sends a message into another agent's session. Unlike `sessions_spawn`, the target agent runs with its **full system prompt** (`promptMode: "full"`), retaining SOUL.md, skills, memory, and personality.

```typescript
// src/config/types.tools.ts:483-488
agentToAgent?: {
  /** Enable agent-to-agent messaging tools. Default: false. */
  enabled?: boolean;
  /** Allowlist of agent ids or patterns (implementation-defined). */
  allow?: string[];
};
```

### Configuration

Agent-to-agent messaging is **off by default**. Enable it explicitly:

```json5
{
  tools: {
    agentToAgent: {
      enabled: true,
      allow: ["agent-a", "agent-b"],
    },
  },
}
```

Source: `docs/concepts/multi-agent.md` — the WhatsApp multi-agent example shows this config pattern.

### Ping-Pong Turns

After the initial send, OpenClaw runs a **reply-back loop** alternating between requester and target agents:

- Max turns: `session.agentToAgent.maxPingPongTurns` (0–5, default 5)
- Agent replies `REPLY_SKIP` to stop the loop
- After the loop, an **announce step** delivers the result to the target's channel

Source: `src/agents/tools/sessions-send-helpers.ts:158-166` — `resolvePingPongTurns()`.

The full A2A flow is implemented in `src/agents/tools/sessions-send-tool.a2a.ts:18-149`:

1. Round 1: requester sends message to target, target replies
2. Rounds 2–N: alternating replies between requester and target (ping-pong)
3. Announce step: target agent decides whether to post result to its channel

### Message Provenance

Inter-session messages are persisted with `message.provenance.kind = "inter_session"` so transcript readers can distinguish routed agent instructions from external user input (`docs/concepts/session-tool.md`).

### Context Injection

The target agent receives an `extraSystemPrompt` with agent-to-agent context:

```typescript
// src/agents/tools/sessions-send-helpers.ts:73-89
"Agent-to-agent message context:"
"Agent 1 (requester) session: <key>."
"Agent 1 (requester) channel: <channel>."
"Agent 2 (target) session: <key>."
```

### Session Visibility

Session tools can be scoped to control cross-session access:

- `self`: only the current session
- `tree`: current + spawned sessions (default)
- `agent`: any session belonging to the current agent
- `all`: any session (cross-agent still requires `tools.agentToAgent`)

Source: `src/config/types.tools.ts:495-503`.

**Verdict:** This is the right mechanism for agents with different personas to communicate programmatically. Each agent retains its full identity.

---

## 3. Channel-Based Communication

### Concept

Instead of using internal tools, agents communicate via external messaging channels (Matrix rooms, Telegram groups, etc.). Each agent is a separate Matrix user that posts and reads messages in shared rooms.

### Architecture

Each agent is bound to a separate channel account via `bindings`:

```json5
{
  agents: {
    list: [
      { id: "researcher", workspace: "~/.openclaw/workspace-researcher" },
      { id: "critic", workspace: "~/.openclaw/workspace-critic" },
    ],
  },
  bindings: [
    { agentId: "researcher", match: { channel: "matrix", accountId: "researcher" } },
    { agentId: "critic", match: { channel: "matrix", accountId: "critic" } },
  ],
  channels: {
    matrix: {
      enabled: true,
      accounts: {
        researcher: {
          homeserver: "https://matrix.example.org",
          accessToken: "syt_researcher_***",
          encryption: true,
        },
        critic: {
          homeserver: "https://matrix.example.org",
          accessToken: "syt_critic_***",
          encryption: true,
        },
      },
    },
  },
}
```

Source: `docs/channels/matrix.md` — multi-account section; `docs/concepts/multi-agent.md` — bindings and agent list.

### How Agents Would "Talk"

When both agents are invited to the same Matrix room:

1. A human (or cron job) posts a message in the room
2. Agent A (researcher) processes it and replies (if mention-gated, requires @mention)
3. Agent B (critic) sees A's reply as a new room message and responds
4. This creates a natural conversation loop visible to all room participants

**Important limitation:** Each agent sees the room as an independent group session (`agent:<agentId>:matrix:group:<roomId>`). They do NOT share conversation context — each maintains its own history (`docs/channels/broadcast-groups.md` — "Session Isolation" section).

### Broadcast Groups (Experimental)

For WhatsApp (only, currently), OpenClaw supports broadcast groups where multiple agents process the same message simultaneously:

```json5
{
  broadcast: {
    strategy: "parallel",
    "120363403215116621@g.us": ["code-reviewer", "security-auditor"],
  },
}
```

Source: `docs/channels/broadcast-groups.md`.

Key properties:
- Currently **WhatsApp only** (Telegram, Discord, Slack planned)
- Agents process in parallel or sequential order
- **No shared context** — agents don't see each other's responses (by design)
- Group context buffer (recent messages) is shared per peer

**Not yet available for Matrix.** When it lands, it would enable true multi-agent broadcast in Matrix rooms.

### Matrix-Specific Setup

Matrix is a **plugin** (not bundled):

```bash
openclaw plugins install @openclaw/matrix
```

Key capabilities relevant to multi-agent:
- Multi-account: each account runs as a separate Matrix user on any homeserver (`docs/channels/matrix.md` — multi-account section)
- E2EE supported (Rust crypto SDK)
- Threads supported (`threadReplies: off | inbound | always`)
- Room allowlists per account
- Per-account DM policy, group policy, mention gating

Source: `extensions/matrix/` — full plugin implementation; `docs/channels/matrix.md` — configuration reference.

---

## 4. Comparison: Internal vs Channel-Based

| Dimension | `sessions_spawn` | `sessions_send` (A2A) | Channel-based (Matrix) |
|---|---|---|---|
| **Persona preserved** | No (minimal mode) | Yes (full mode) | Yes (full mode) |
| **SOUL.md active** | No | Yes | Yes |
| **Human observability** | Announce only | Announce only | Full (room history) |
| **Human participation** | No | No | Yes (same room) |
| **Latency** | Low (internal) | Low (internal) | Higher (network + sync) |
| **Session isolation** | Partial | Full | Full + network |
| **Conversation history** | Ephemeral (auto-archived) | Persistent | Persistent (Matrix server) |
| **Max agents** | 5 children default | No hard limit | No hard limit |
| **Turn control** | Single task | Ping-pong (0–5 turns) | Unbounded (room messages) |
| **Configuration complexity** | Low | Medium | High (Matrix accounts) |
| **Requires external service** | No | No | Yes (Matrix homeserver) |
| **Cross-gateway possible** | No | No | Yes (federated Matrix) |

---

## 5. Architecture Patterns

### 5a. Hub-and-Spoke (Internal)

One orchestrator agent delegates to specialist workers via `sessions_spawn`.

```
        ┌─────────┐
        │  Main   │  ← receives all user messages
        │ (hub)   │
        └────┬────┘
      ┌──────┼──────┐
      ▼      ▼      ▼
  ┌──────┐┌──────┐┌──────┐
  │coder ││triage││monitor│  ← subagents (minimal prompt)
  └──────┘└──────┘└──────┘
```

- Used in the B4Racing setup (`exploration/agents/subagent-skills.md`)
- Workers are stateless, task-oriented, no persona
- Main agent orchestrates and reports results

**Config example** (from B4Racing):
```json5
{
  agents: {
    list: [
      { id: "main", default: true, subagents: { allowAgents: ["coder"] } },
      { id: "coder" },
    ],
  },
}
```

Source: `src/agents/subagent-spawn.ts:131-147` — `allowAgents` enforcement.

### 5b. Peer-to-Peer (Internal A2A)

Multiple agents with distinct personas communicate via `sessions_send`. No single hub — any agent can message any other (if allowed).

```
  ┌──────────┐    sessions_send    ┌──────────┐
  │researcher│ ◄────────────────► │  critic   │
  │ SOUL.md  │                    │ SOUL.md   │
  └──────────┘                    └──────────┘
       │                               │
       ▼                               ▼
   Telegram                        Telegram
   (announce)                      (announce)
```

- Both agents run with full system prompts
- Ping-pong loop allows multi-turn conversation (up to 5 turns)
- Results are announced to their respective channels

**Config:**
```json5
{
  tools: {
    agentToAgent: {
      enabled: true,
      allow: ["researcher", "critic"],
    },
  },
  session: {
    agentToAgent: {
      maxPingPongTurns: 3,
    },
  },
}
```

Source: `src/agents/tools/sessions-send-helpers.ts:158-166`; `docs/concepts/session-tool.md`.

### 5c. Broadcast (Channel-Based)

Multiple agents respond to the same message in a shared channel. Currently WhatsApp only; planned for other channels.

```
  Human posts in WhatsApp group
       │
       ├──► Agent A (code-reviewer)  → reply
       ├──► Agent B (security-audit) → reply
       └──► Agent C (docs-checker)   → reply
```

- Agents don't see each other's responses
- Parallel or sequential processing
- Each agent has its own session/context

Source: `docs/channels/broadcast-groups.md`.

### 5d. Room-Based (Channel-Based, Matrix)

Multiple agents as separate Matrix users in a shared room. Messages flow through the Matrix protocol. Agents see each other's messages as regular room messages.

```
  ┌────────────────────────────────────────┐
  │         Matrix Room                     │
  │                                         │
  │  @researcher:example.org  (agent)       │
  │  @critic:example.org      (agent)       │
  │  @human:example.org       (user)        │
  │                                         │
  │  Human: "Analyze this paper"            │
  │  Researcher: "Here are my findings..." │
  │  Critic: "I disagree because..."       │
  │  Researcher: "Good point, however..."  │
  └────────────────────────────────────────┘
```

- Each agent is a separate Matrix account bound to an agent ID via bindings
- Full persona preservation (each agent has its own SOUL.md, workspace)
- Conversation is visible and joinable by humans
- Can work across federated Matrix servers
- Mention gating controls when agents respond

**Config:**
```json5
{
  agents: {
    list: [
      { id: "researcher", workspace: "~/.openclaw/workspace-researcher" },
      { id: "critic", workspace: "~/.openclaw/workspace-critic" },
    ],
  },
  bindings: [
    { agentId: "researcher", match: { channel: "matrix", accountId: "researcher" } },
    { agentId: "critic", match: { channel: "matrix", accountId: "critic" } },
  ],
  channels: {
    matrix: {
      enabled: true,
      groupPolicy: "allowlist",
      groups: {
        "!debateRoom:example.org": {
          allow: true,
          requireMention: false,  // agents respond to all messages
        },
      },
      accounts: {
        researcher: {
          homeserver: "https://matrix.example.org",
          accessToken: "syt_researcher_***",
          encryption: true,
        },
        critic: {
          homeserver: "https://matrix.example.org",
          accessToken: "syt_critic_***",
          encryption: true,
        },
      },
    },
  },
}
```

Source: `docs/channels/matrix.md` — multi-account, rooms, mention gating; `docs/concepts/multi-agent.md` — bindings.

---

## 6. Practical Recommendations

### Use `sessions_spawn` (subagents) when:

- You need **task delegation** to a worker (coding, data processing)
- Persona/personality does **not** matter
- You want **ephemeral** sessions that auto-archive
- You need **depth control** and concurrency limits
- Example: B4Racing's coder subagent

### Use `sessions_send` (A2A) when:

- Agents need to **retain their full persona** (SOUL.md)
- Communication is **programmatic** (no human observation needed)
- You want **bounded conversation** (up to 5 ping-pong turns)
- You need **low latency** (no network round-trip)
- Example: a researcher agent asking a fact-checker agent to verify claims

### Use channel-based (Matrix rooms) when:

- Humans should **observe and participate** in agent conversations
- You want **unbounded conversation** (not limited to 5 turns)
- You need **auditability** (Matrix server stores full history)
- You want agents to communicate **across separate gateways** (Matrix federation)
- You want **natural language interaction** visible in a chat client
- Example: a debate between a researcher and a critic in a Matrix room, with a human moderator

### Do NOT use channel-based when:

- You need **fast, high-volume** agent-to-agent communication (network latency)
- You want to avoid **external dependencies** (Matrix homeserver)
- You need **tight coordination** (channel-based is async and loosely coupled)
- You want **structured data exchange** (channel messages are plain text)

---

## 7. Open Issues and Caveats

1. **Broadcast groups are WhatsApp-only.** Matrix broadcast is planned but not implemented (`docs/channels/broadcast-groups.md` — Compatibility section).

2. **Matrix room loop risk.** If two agents are in the same room with `requireMention: false`, they could enter an infinite reply loop. Mitigation: use mention gating (`requireMention: true`) and explicit @mention patterns per agent (`docs/channels/matrix.md` — rooms section; `docs/concepts/multi-agent.md` — `groupChat.mentionPatterns`).

3. **No shared context in channel-based.** Each agent has its own session and cannot read the other agent's internal state. They only see what is posted in the room (`docs/channels/broadcast-groups.md` — "Session Isolation").

4. **A2A ping-pong is limited to 5 turns.** For longer conversations, use channel-based or chain multiple `sessions_send` calls (`src/agents/tools/sessions-send-helpers.ts:10-11`).

5. **Matrix is a plugin.** It requires separate installation (`openclaw plugins install @openclaw/matrix`) and a Matrix homeserver. Not suitable for minimal deployments (`docs/channels/matrix.md`).

6. **Self-message filtering.** When agents share a Matrix room, the Matrix plugin filters out the bot's own messages. Each agent only processes messages from others, not its own replies. This prevents self-loops but also means agent A does not "process" its own messages through the LLM pipeline — it only sees them in the room context buffer.

---

## References

### Source Files

| File | What it covers |
|---|---|
| `src/agents/subagent-spawn.ts:109-147` | Depth limit, children limit, allowAgents enforcement |
| `src/agents/subagent-spawn.ts:148` | Child session key format |
| `src/agents/subagent-spawn.ts:230-242` | Subagent system prompt and task message construction |
| `src/agents/subagent-announce.ts:595-682` | `buildSubagentSystemPrompt()` |
| `src/agents/pi-embedded-runner/run/attempt.ts:425-428` | `promptMode` selection |
| `src/agents/system-prompt.ts:17-638` | System prompt builder with `isMinimal` gates |
| `src/agents/workspace.ts:468-478` | `MINIMAL_BOOTSTRAP_ALLOWLIST` (AGENTS.md + TOOLS.md only) |
| `src/agents/tools/sessions-send-tool.ts` | `sessions_send` tool implementation |
| `src/agents/tools/sessions-send-tool.a2a.ts:18-149` | A2A ping-pong and announce flow |
| `src/agents/tools/sessions-send-helpers.ts:73-166` | A2A context builders, `REPLY_SKIP`, `ANNOUNCE_SKIP`, ping-pong turns |
| `src/config/types.tools.ts:483-503` | `agentToAgent` and `sessions.visibility` type definitions |
| `extensions/matrix/` | Full Matrix plugin implementation |

### Official Documentation

| Doc | Relevance |
|---|---|
| `docs/concepts/multi-agent.md` | Multi-agent routing, bindings, per-agent isolation, platform examples |
| `docs/concepts/session-tool.md` | `sessions_list`, `sessions_history`, `sessions_send`, `sessions_spawn` — full tool reference |
| `docs/concepts/session.md` | Session management, `dmScope`, session keys, send policy |
| `docs/channels/matrix.md` | Matrix plugin setup, multi-account, E2EE, rooms, threads, mention gating |
| `docs/channels/channel-routing.md` | Routing rules, session key shapes, broadcast groups overview |
| `docs/channels/broadcast-groups.md` | Broadcast groups: multi-agent per message (WhatsApp only, experimental) |
| `docs/gateway/multiple-gateways.md` | Running multiple gateway instances for stronger isolation |
| `docs/tools/subagents.md` | Subagent tool params, nesting, announce, tool policy, limitations |
| `docs/concepts/system-prompt.md` | Prompt modes (full/minimal/none), bootstrap injection |

### Prior Exploration

| Doc | Relevance |
|---|---|
| `exploration/agents/subagent-skills.md` | Skill inheritance for subagents, B4Racing practical example |
| `exploration/agents/subagent-deep-dive.md` | Full subagent inheritance analysis: what is kept, what is stripped |
