# Session Transcripts — Storage, Rendering, and Live Viewing

## Question

How does OpenClaw store, render, and expose session transcripts? What built-in and community tools exist to view agent conversations as readable, live-updating chat logs — and what gaps remain for a "mission control" style multi-agent transcript dashboard?

## TL;DR

OpenClaw stores every session as a JSONL file with a tree-structured entry format (`id`/`parentId` DAG). The built-in **Control UI**, **TUI**, and **macOS WebChat** all render live transcripts via the Gateway WebSocket API. A `/export-session` slash command generates self-contained HTML snapshots. After evaluating community projects, **PinchChat** is the recommended viewer for Mimas deployment: it renders tool calls, thinking blocks, and supports split-view — all via Docker with zero backend. **OpenClaw Deck** (7-column layout) looks appealing but lacks tool call and thinking block rendering, making it unsuitable for b4arena agents. A purely passive, read-only multi-agent transcript feed (scrolling blog/timeline) remains a gap that could be filled with a JSONL file watcher approach.

---

## 1. JSONL Session Storage Format

### File Location

```
~/.openclaw/agents/<agentId>/sessions/<sessionId>.jsonl
```

Each session is one file. A session store index at `sessions.json` maps `sessionKey → SessionEntry` with metadata (token counts, model overrides, compaction count, labels).

### Entry Structure

Each line is a JSON object. Entries form a tree via `id` + `parentId`:

```jsonl
{ "type": "session", "version": 3, "id": "<sessionId>", "cwd": "...", "timestamp": "..." }
{ "id": "msg-1", "parentId": null, "type": "message", "message": { "role": "user", "content": [...], "timestamp": 1234 } }
{ "id": "msg-2", "parentId": "msg-1", "type": "message", "message": { "role": "assistant", "content": [...], "usage": { "input": 500, "output": 200 } } }
{ "id": "cmp-1", "parentId": "msg-2", "type": "compaction", "firstKeptEntryId": "msg-2", "tokensBefore": 12345 }
{ "id": "lbl-1", "parentId": "msg-2", "type": "label", "targetId": "msg-2", "label": "checkpoint" }
```

### Entry Types

| Type | Purpose |
|---|---|
| `session` | Header — version, session ID, working directory |
| `message` | User or assistant message with content blocks |
| `compaction` | Marks where context was trimmed (summarization boundary) |
| `label` | Named bookmark on a specific entry |
| `branch_summary` | Summary of a pruned branch |
| `custom` / `custom_message` | Extension-defined entries |

### Message Content Blocks

The `message.content` array contains typed blocks:

| Block Type | Content |
|---|---|
| `text` | Plain text or Markdown |
| `tool_use` | Tool invocation (name, input params) |
| `tool_result` | Tool execution result |
| `thinking` | Model reasoning (extended thinking) |
| `image` | Inline image data |

### Key Fields on Messages

| Field | Description |
|---|---|
| `role` | `"user"` or `"assistant"` |
| `provider` | e.g., `"anthropic"`, `"openai"`, `"google"` |
| `model` | Model ID used for this turn |
| `usage` | Token counts: `input`, `output`, `cacheRead`, `cacheWrite`, `totalTokens` |
| `usage.cost` | Cost breakdown |
| `stopReason` | `"stop"`, `"tool_use"`, `"max_tokens"` |
| `provenance` | Origin metadata, e.g., `{ kind: "inter_session" }` for A2A messages |
| `timestamp` | Epoch milliseconds |

### Tree Linearization

The DAG structure (branches, compactions) means you cannot simply read the file top-to-bottom. The HTML export code in `src/auto-reply/reply/commands-export-session.ts` demonstrates correct linearization: it calls `SessionManager.open(file).getEntries()` and reconstructs the active branch by following `parentId` chains from the leaf entry.

Source: `docs/reference/session-management-compaction.md:124-141`, `src/auto-reply/reply/commands-export-session.ts:19-25`

---

## 2. Built-in Transcript Viewers

### Control UI (Browser)

- **URL**: `http://<host>:18789/`
- **Tech**: Vite + Lit SPA
- **Live**: Yes — connects to Gateway WebSocket, streams via `chat` and `agent` events
- **Features**: Chat view with Markdown rendering, tool call cards, session switcher, model/thinking toggles
- **Limitation**: One session at a time, interactive (not read-only)

Source: `docs/web/control-ui.md`

### TUI (Terminal)

- **Command**: `openclaw tui` (local) or `openclaw tui --url ws://<host>:<port> --token <token>` (remote)
- **Tech**: `@mariozechner/pi-tui` terminal UI framework
- **Live**: Yes — fetches history via `chat.history` (default 200 messages), streams new messages
- **Features**: Chat log rendering (`src/tui/components/chat-log.ts`), streaming assistant updates, tool call display
- **Limitation**: One session at a time, terminal-only

Source: `docs/web/tui.md`, `src/tui/gateway-chat.ts`

### macOS WebChat

- **Access**: Menu bar app → chat window
- **Tech**: SwiftUI
- **Live**: Yes — same Gateway WS API
- **Features**: Session switcher for non-main sessions
- **Limitation**: macOS only, one session at a time

Source: `docs/web/webchat.md`, `docs/platforms/mac/webchat.md`

### HTML Export (`/export-session`)

- **Trigger**: Send `/export-session [path]` or `/export [path]` as a chat message
- **Output**: Self-contained HTML file with embedded CSS, JS, and vendor libs (marked.js, highlight.js)
- **Features**:
  - Sidebar with session tree navigation
  - Filter modes: Default / No-tools / User / Labeled / All
  - Dark theme (`pi-mono`)
  - Deep-linking via `?leafId=` and `?targetId=` URL params
  - Markdown rendering with syntax highlighting
- **Limitation**: Static snapshot — no live updates
- **Output path**: `openclaw-session-<sessionId[0:8]>-<timestamp>.html` in workspace dir

Source: `src/auto-reply/reply/commands-export-session.ts`, `src/auto-reply/reply/export-html/` (template.html, template.css, template.js)

---

## 3. Gateway WebSocket API for Transcripts

Any custom viewer can use these Gateway methods:

| Method | Direction | Purpose |
|---|---|---|
| `sessions.list` | Request → Response | List all sessions with metadata |
| `sessions.preview` | Request → Response | Last N messages from multiple sessions (for session list UI) |
| `chat.history` | Request → Response | Full message history for one session (bounded: 12,000 chars/text, 128 KB/message) |
| `chat.send` | Request → Event stream | Send a message, receive streaming response |
| `chat.inject` | Request → Response | Append assistant note without triggering agent run |
| `chat.abort` | Request → Response | Abort active agent run |
| `chat` | Event (server → client) | Live message stream (new messages, streaming tokens) |
| `agent` | Event (server → client) | Tool call cards, status updates |
| `presence` | Event (server → client) | Agent online/typing/idle status |
| `health` | Event (server → client) | Health state broadcasts |

**Authentication**: Gateway token via `?token=<OPENCLAW_GATEWAY_TOKEN>` query param or `Authorization` header.

Source: `docs/gateway/protocol.md`, `src/gateway/server-methods/chat.ts`, `src/gateway/server-methods/sessions.ts`

### Transcript Event Bus (Internal)

For plugins running inside the Gateway process:

```typescript
import { onSessionTranscriptUpdate } from "src/sessions/transcript-events";
onSessionTranscriptUpdate((sessionFile) => { /* re-read and render */ });
```

Fires after every `appendAssistantMessageToSessionTranscript()` write.

Source: `src/sessions/transcript-events.ts`, `src/sessions/transcript.ts:162`

---

## 4. Community Projects

### Projects That Show Conversation Content

| Project | GitHub | Live? | Multi-Agent? | Key Feature |
|---|---|---|---|---|
| **PinchChat** | [MarlBurroW/pinchchat](https://github.com/MarlBurroW/pinchchat) | Yes | Split-view (2 sessions) | Best per-agent transcript viewer: tool call visualization, reasoning blocks, token-by-token streaming, Markdown export |
| **OpenClaw Deck** | [kellyclaudeai/openclaw-deck](https://github.com/kellyclaudeai/openclaw-deck) | Yes | Up to 7 agents side-by-side | Closest to "mission control" — columns per agent, interactive, themes |
| **OpenClaw Studio** | [grp06/openclaw-studio](https://github.com/grp06/openclaw-studio) | Yes | Single session | Chat streaming + tool call + thinking trace rendering |
| **Clawtrol** | npm: `clawtrol` | Yes | Unclear | "Session viewer with live chat" listed as feature |

All of these are **interactive chat UIs** that connect to the Gateway WebSocket. None are purely passive/read-only observers.

### Deep Evaluation: PinchChat vs. OpenClaw Deck

Both were evaluated in detail for deployment on Mimas (Fedora 43, OpenClaw gateway running on host, Docker CE available).

#### PinchChat

**Repo:** [MarlBurroW/pinchchat](https://github.com/MarlBurroW/pinchchat) — "A sleek, dark-themed webchat UI for OpenClaw"

| Dimension | Detail |
|---|---|
| **Tech stack** | React 19 + TypeScript, Vite 7, Tailwind CSS v4, Radix UI |
| **Rendering** | react-markdown + remark-gfm + rehype-highlight |
| **Architecture** | Pure frontend SPA — no backend, no database. Docker runs Nginx serving static `dist/` |
| **Docker** | `ghcr.io/marlburrow/pinchchat:latest` — port 80 in container, health check included |
| **Config** | `VITE_GATEWAY_WS_URL` (build-time, pre-fills login screen), gateway token entered at runtime |
| **Maturity** | 374 commits, 24 stars, 2 contributors, MIT, last push 2026-02-22 (active) |
| **Read-only** | No — input area always rendered |

**Features:**
- Real-time token streaming via WebSocket
- Multi-session sidebar with drag-and-drop reordering
- Split-view mode (2 sessions side by side)
- Tool call rendering: colored badges, expandable parameters and results
- Thinking/reasoning blocks (collapsible)
- Message search (Ctrl+F), raw JSON viewer for debugging
- Export conversations as Markdown
- Inline image rendering with lightbox
- Token usage progress bars per session
- Channel icons (Discord, Telegram, cron, etc.)
- PWA support with offline caching
- 4 themes (Dark, Light, OLED Black, System), 6 accent colors
- Keyboard shortcuts, ARIA accessibility, i18n (en, fr)

**Deployment (Docker Compose):**

```yaml
services:
  pinchchat:
    image: ghcr.io/marlburrow/pinchchat:latest
    ports:
      - "3001:80"
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:80/"]
      interval: 30s
      timeout: 3s
      retries: 3
```

#### OpenClaw Deck

**Repo:** [kellyclaudeai/openclaw-deck](https://github.com/kellyclaudeai/openclaw-deck)

| Dimension | Detail |
|---|---|
| **Tech stack** | React + TypeScript, Vite 6, Zustand, react-markdown + highlight.js |
| **Architecture** | Pure frontend SPA — no backend, no database |
| **Docker** | None — no Dockerfile in repo. Must build static files and serve via Nginx manually |
| **Config** | `VITE_GATEWAY_URL` + `VITE_GATEWAY_TOKEN` (both build-time) |
| **Maturity** | 211 stars, 30 forks, 4 issues, MIT, last push 2026-02-18 (very young) |
| **Read-only** | No — input field always active |

**Features:**
- 7 color-coded agent columns with independent scrolling
- Per-agent model selection at creation time
- Full markdown rendering with syntax highlighting
- 8 dark themes (Midnight, Ocean Depths, Forest Night, etc.)
- Keyboard shortcuts (Tab/Shift+Tab for column switch, Cmd+1–9)
- **No tool call rendering**
- **No thinking block display**
- **No export functionality**

#### Head-to-Head Comparison

| Dimension | **PinchChat** | **OpenClaw Deck** |
|---|---|---|
| Multi-agent view | Split-view (2) + sidebar | 7 columns side-by-side |
| Tool call rendering | Yes — colored badges, expandable | No |
| Thinking blocks | Yes — collapsible | No |
| Docker image | Yes — official GHCR | No — manual build required |
| Markdown export | Yes | No |
| Message search | Yes (Ctrl+F) | No |
| Themes | 4 (incl. Light + OLED) | 8 (dark only) |
| Maturity | 374 commits, actively maintained | Very young (Feb 2026) |
| Stars | 24 | 211 |

**Verdict: PinchChat is the stronger choice.** Tool call and thinking block rendering are essential for b4arena agents — that is 90% of the conversation content. OpenClaw Deck's 7-column layout is visually appealing but without tool/thinking rendering, the transcripts would be mostly empty or unreadable.

#### Gateway Connection Architecture

Both projects connect **from the browser directly to the OpenClaw Gateway WebSocket** — there is no backend proxy. This has deployment implications:

```
User's Browser ──WebSocket──→ OpenClaw Gateway (:18789)
       │
       └── HTTP ──→ PinchChat Nginx (:3001, static files only)
```

The Gateway port (18789) must be reachable from wherever the browser runs. On a LAN setup (laptop → Mimas), this means:
1. Firewall must allow port 18789 from LAN
2. `VITE_GATEWAY_WS_URL` must point to the LAN IP (e.g., `ws://192.168.1.11:18789`)
3. Since the URL is baked at build-time, either enter it manually on the login screen or build a custom image

**Recommended: Nginx reverse proxy** on Mimas serving both PinchChat and the Gateway WS under one origin:

```
https://mimas.local/         → PinchChat (container :80)
https://mimas.local/ws       → OpenClaw Gateway WS (:18789)
```

This avoids mixed-content issues with HTTPS and simplifies the `VITE_GATEWAY_WS_URL` to a relative path.

### Projects That Show Metrics Only (Not Transcripts)

| Project | GitHub | Notes |
|---|---|---|
| **Mission Control** | [crshdn/mission-control](https://github.com/crshdn/mission-control) | Task/Kanban board, event feed — no message content |
| **ClawDeck** | [clawdeckio/clawdeck](https://github.com/clawdeckio/clawdeck) | Task management, activity feed — no transcripts |
| **openclaw-dashboard** | [tugcantopaloglu/openclaw-dashboard](https://github.com/tugcantopaloglu/openclaw-dashboard) | API cost, system health, memory browser — no transcripts |

### General AI Agent Observability (Cross-Platform)

| Tool | Shows Conversation Content? | Notes |
|---|---|---|
| **AgentOps** ([agentops.ai](https://agentops.ai/)) | Yes — exact prompts and completions per LLM call | Cloud SaaS, requires SDK instrumentation, session replay feature |
| **Langfuse** ([langfuse.com](https://langfuse.com/)) | Yes — full prompt/completion per generation span | Open-source, self-hostable. See also [chat-logs-viewer](https://github.com/sib-swiss/chat-logs-viewer) built on Langfuse |
| **LangSmith** ([langchain.com/langsmith](https://www.langchain.com/langsmith)) | Yes — exact messages per turn | LangChain/LangGraph only |
| **Google ADK Web UI** | Yes — events, state, step-by-step execution | Google Agent Development Kit only |

### Generic LLM Conversation Log Viewers

These render exported chat logs without a live connection — potentially adaptable to OpenClaw's JSONL:

| Tool | GitHub | Notes |
|---|---|---|
| **LLM Conversations Viewer** | [TomzxCode/llm-conversations-viewer](https://github.com/TomzxCode/llm-conversations-viewer) | 100% client-side, loads ChatGPT/Claude JSON exports, real-time search, Markdown rendering |
| **LLM Chat Explorer** | [levysoft/llm-chat-explorer](https://github.com/levysoft/llm-chat-explorer) | Upload and browse exported conversations, in-browser |
| **chat-logs-viewer** | [sib-swiss/chat-logs-viewer](https://github.com/sib-swiss/chat-logs-viewer) | SolidJS app for Langfuse JSONL files — structurally similar to OpenClaw's format |
| **agno agent-ui** | [agno-agi/agent-ui](https://github.com/agno-agi/agent-ui) | Modern chat UI with reasoning steps, tool calls, streaming — framework-agnostic |

---

## 5. The Gap: Passive Multi-Agent Transcript Feed

### What Exists vs. What's Missing

```
What exists today:
┌─────────────────────────────────────────────┐
│  Built-in Control UI    → 1 session, live   │
│  PinchChat              → 2 sessions, live  │
│  OpenClaw Deck          → 7 agents, live    │  ← closest
│  /export-session        → 1 session, static │
│  AgentOps / Langfuse    → spans, not chat   │
└─────────────────────────────────────────────┘

What's missing:
┌─────────────────────────────────────────────┐
│  Passive multi-agent transcript feed        │
│  • Read-only (no interaction with agents)   │
│  • All agents, all sessions, one view       │
│  • Live-updating (streaming or polling)     │
│  • Reads like a chat log / blog timeline    │
│  • Filterable by agent, time, session       │
│  • No Gateway connection required (opt.)    │
└─────────────────────────────────────────────┘
```

### Why OpenClaw Deck Is Close But Not Enough

OpenClaw Deck shows up to 7 agents side-by-side, which is the closest to the "mission control" vision. However:

1. It is **interactive** (sends messages), not a passive observer
2. It requires **active session selection** — doesn't auto-discover all sessions
3. It connects via **Gateway WebSocket** — requires a running Gateway per instance
4. It does not provide a **timeline/feed view** across agents

---

## 6. Implementation Approaches

### A) Gateway WebSocket Client (Recommended for Live)

Build a read-only web client that connects to the Gateway and renders all active sessions.

```
Browser ──WebSocket──→ Gateway (:18789)
  │
  ├─ sessions.list → discover all sessions
  ├─ sessions.preview → get recent messages per session
  ├─ chat.history (per session) → full transcript
  └─ subscribe to `chat` + `agent` events → live updates
```

**Pros**: Official API, live streaming, includes tool calls and thinking blocks.
**Cons**: Requires running Gateway, bounded message sizes (12,000 chars/text field).

### B) JSONL File Watcher (Simplest for Local/Server)

Watch the session directories on disk and re-render on change.

```
fs.watch("~/.openclaw/agents/*/sessions/*.jsonl")
  │
  ├─ On change: re-read file, linearize DAG, render to HTML/Markdown
  └─ Serve via simple HTTP server with SSE for live updates
```

**Pros**: No Gateway needed, works on Mimas directly, full untruncated content.
**Cons**: Must handle DAG linearization (use `commands-export-session.ts` as reference), no streaming granularity (only sees completed writes), file permission considerations.

### C) Periodic HTML Export (Lowest Effort)

Automate the existing `/export-session` mechanism via a script.

```bash
#!/bin/bash
# Poll all active sessions, export each to a shared directory
for session in $(openclaw gateway call sessions.list --json | jq -r '.[] | .sessionKey'); do
  openclaw gateway call chat.inject --params "{\"sessionKey\":\"$session\",\"text\":\"/export-session /var/www/transcripts/$session.html\"}"
done
```

**Pros**: Reuses existing, well-tested HTML export with full rendering.
**Cons**: Not truly live (polling interval), triggers agent activity (`chat.inject`), one file per session.

### D) Custom OpenClaw Plugin (Most Integrated)

Build a plugin that hooks into the transcript event bus and renders on every write.

```typescript
import { onDiagnosticEvent } from "openclaw/plugin-sdk";

onDiagnosticEvent((evt) => {
  if (evt.type === "message.processed") {
    // Read session, render, push to external viewer
  }
});
```

**Pros**: Most integrated, real-time, access to all internal state.
**Cons**: Highest effort, must maintain with OpenClaw upgrades, runs inside Gateway process.

### Comparison

| Approach | Effort | Live? | Needs Gateway? | Multi-Agent? |
|---|---|---|---|---|
| **A) WS Client** | Medium | Yes (streaming) | Yes | Yes |
| **B) File Watcher** | Low | Yes (on write) | No | Yes |
| **C) HTML Export** | Low | No (polling) | Yes | Yes |
| **D) Plugin** | High | Yes (real-time) | Yes (runs inside) | Yes |

For the b4arena use case (agents on Mimas), **Approach B** is most practical — the JSONL files are on disk, no Gateway dependency, and a simple SSE-based web server can push updates to a browser.

---

## 7. Recommended Next Steps

1. **Deploy PinchChat on Mimas** — Docker-ready, best feature set for b4arena agents (tool calls, thinking blocks). Add an Ansible task to `infra/` for the container + an Nginx reverse proxy for same-origin WebSocket access.
2. **If passive/read-only viewing is needed later** — prototype Approach B (JSONL file watcher + SSE) as a lightweight complement. PinchChat covers 80% of the use case but is interactive, not a passive observer.
3. **Consider Langfuse** as a complementary solution for long-term audit trails — it captures conversation content via OTel spans (if configured with the Collector attribute mapping from `exploration/deployment/langfuse-audit-trail.md`), providing a searchable, persistent archive alongside the live viewer.

---

## References

### OpenClaw Source Files

| File | Relevance |
|---|---|
| `src/auto-reply/reply/commands-export-session.ts` | HTML export logic, DAG linearization |
| `src/auto-reply/reply/export-html/template.html` | Self-contained HTML shell |
| `src/auto-reply/reply/export-html/template.css` | Chat rendering styles (pi-mono dark theme) |
| `src/auto-reply/reply/export-html/template.js` | Client-side JS: marked.js rendering, tree navigation, filtering |
| `src/tui/components/chat-log.ts` | Terminal chat log renderer |
| `src/tui/gateway-chat.ts` | TUI Gateway client (chat.history + WS events) |
| `src/sessions/transcript-events.ts` | In-process transcript event bus |
| `src/sessions/transcript.ts` | Transcript write logic |
| `src/gateway/server-methods/chat.ts` | Gateway chat method handlers |
| `src/gateway/server-methods/sessions.ts` | Gateway session method handlers |
| `src/gateway/protocol/schema.ts` | Gateway WS protocol schema |
| `src/utils/transcript-tools.ts` | Utilities for inspecting transcript messages |

### OpenClaw Documentation

| Doc | Relevance |
|---|---|
| `docs/reference/session-management-compaction.md` | JSONL format, compaction, entry types |
| `docs/web/control-ui.md` | Built-in browser chat UI |
| `docs/web/tui.md` | Terminal chat UI |
| `docs/web/webchat.md` | macOS/iOS WebChat |
| `docs/gateway/protocol.md` | Gateway WebSocket protocol (all methods) |
| `docs/cli/sessions.md` | CLI session commands |

### Community Projects

| Project | URL |
|---|---|
| PinchChat | [github.com/MarlBurroW/pinchchat](https://github.com/MarlBurroW/pinchchat) |
| OpenClaw Deck | [github.com/kellyclaudeai/openclaw-deck](https://github.com/kellyclaudeai/openclaw-deck) |
| OpenClaw Studio | [github.com/grp06/openclaw-studio](https://github.com/grp06/openclaw-studio) |
| LLM Conversations Viewer | [github.com/TomzxCode/llm-conversations-viewer](https://github.com/TomzxCode/llm-conversations-viewer) |
| chat-logs-viewer | [github.com/sib-swiss/chat-logs-viewer](https://github.com/sib-swiss/chat-logs-viewer) |
| agno agent-ui | [github.com/agno-agi/agent-ui](https://github.com/agno-agi/agent-ui) |

### Prior Exploration

| Doc | Relevance |
|---|---|
| `exploration/deployment/best-practices.md` | Monitoring section: health checks, logging, OTel metrics/spans |
| `exploration/deployment/langfuse-audit-trail.md` | Langfuse as audit trail, OTel integration, attribute mapping |
| `exploration/architecture/architecture-overview.md` | Gateway architecture, session management overview |
