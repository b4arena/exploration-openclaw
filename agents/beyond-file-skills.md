# Beyond File-Based Skills

Non-standard content formats and workflow runtimes in OpenClaw that go beyond the standard `SKILL.md` approach.

## Overview

OpenClaw supports three distinct execution models for skills/workflows, forming a spectrum from static instructions to fully deterministic pipelines:

| Format | File Extension | Interpreter | Deterministic? |
|---|---|---|---|
| SKILL.md | `.md` | Gateway (prompt injection) | No — LLM decides |
| OpenProse | `.prose` | LLM-as-VM (compiler.md) | No — LLM orchestrates |
| Lobster | `.lobster` | External `lobster` CLI | Yes — shell commands |
| LLM Task | (tool call) | Single constrained LLM call | Partially — schema-bound |

**Key architectural insight:** Neither `.prose` nor `.lobster` files are parsed by the Node.js gateway. OpenProse is interpreted by the LLM itself (the model _is_ the VM), while Lobster files are passed as a path argument to an external CLI binary. Both formats live entirely outside the gateway process (`src/agents/skills/workspace.ts`).

## 1. OpenProse — `.prose` Files

A declarative, markdown-first DSL for orchestrating multi-agent AI workflows. The LLM loads a ~101KB grammar specification (`compiler.md`) and becomes the runtime interpreter.

**Plugin:** `extensions/open-prose/`

### Activation

- Slash command: `/prose`
- Uses `command-dispatch: tool` frontmatter to bypass the model and route directly to the OpenProse tool (`extensions/open-prose/skills/prose/SKILL.md`)
- This dispatch mechanism is implemented in `src/agents/skills/workspace.ts:723-768`

### Syntax

```prose
agent researcher:
  model: sonnet
  skills: ["web-search"]

parallel:
  findings = session: researcher
    prompt: "Research {topic}"

loop until **approved**:
  session "Write draft"

items | map: session "Process {item}"
```

Source: `extensions/open-prose/skills/prose/examples/roadmap/syntax/open-prose-syntax.prose`

### Key Constructs

- **`agent`** — define named agents with model, skills, and persona
- **`session`** — spawn a sub-agent via `sessions_spawn` (mapped to Task tool)
- **`parallel`** — run multiple sessions concurrently
- **`loop until`** — iterate until a condition (often human approval) is met
- **`items | map`** — process collections through sessions
- **`read` / `write`** — file I/O
- **`web_fetch`** — URL fetching

### State Backends

| Backend | Storage | Notes |
|---|---|---|
| `filesystem` (default) | `.prose/runs/{id}/` | Persistent across restarts |
| `in-context` | Conversation memory | Transient, lost on context reset |
| `sqlite` (experimental) | Local DB | Requires `sqlite3` binary |
| `postgres` (experimental) | Remote DB | Requires `psql` + `OPENPROSE_POSTGRES_URL` |

### Remote Programs

Programs can be fetched from a registry: `/prose run handle/slug` resolves to `https://p.prose.md/<handle>/<slug>`.

### Key Files

- `extensions/open-prose/skills/prose/SKILL.md` — activation rules, routing
- `extensions/open-prose/skills/prose/compiler.md` — full grammar (~101KB)
- `extensions/open-prose/skills/prose/examples/` — 37+ example programs
- `extensions/open-prose/openclaw.plugin.json` — plugin manifest

## 2. Lobster — `.lobster` Files

A typed workflow runtime for deterministic, multi-step shell pipelines with explicit approval checkpoints and resumable state.

**Plugin:** `extensions/lobster/`

### Activation

- Must be explicitly allowlisted: `tools.alsoAllow: ["lobster"]`
- Runs via external `lobster` CLI binary as subprocess (`LOBSTER_MODE=tool`)
- Source: `extensions/lobster/src/lobster-tool.ts`

### File Format (YAML)

```yaml
name: inbox-triage
args:
  tag:
    default: "family"
steps:
  - id: collect
    command: inbox list --json
  - id: approve
    command: inbox apply --approve
    stdin: $categorize.stdout
    approval: required
  - id: execute
    command: inbox apply --execute
    condition: $approve.approved
```

Source: `docs/tools/lobster.md:128`

### Key Features

- **Steps** are shell commands, chained via `stdin`/`stdout` references (`$step_id.stdout`)
- **Approval gates** pause execution and return a resume token
- **Conditions** control step execution based on previous step results
- **Resumable** via tokens: `lobster resume --token <token> --approve yes|no`

### Tool Parameters

| Parameter | Values | Default |
|---|---|---|
| `action` | `run`, `resume` | — |
| `pipeline` | Path to `.lobster` file | — |
| `timeoutMs` | milliseconds | 20,000 |
| `maxStdoutBytes` | bytes | 512,000 |

### Output Envelope

```json
{
  "ok": true,
  "status": "ok|needs_approval|cancelled",
  "output": [...],
  "requiresApproval": {
    "prompt": "...",
    "items": [...],
    "resumeToken": "..."
  }
}
```

## 3. LLM Task — Structured JSON Tool Calls

A plugin tool for running a single LLM call with a JSON Schema constraint, returning validated output.

**Plugin:** `extensions/llm-task/`
**Docs:** `docs/tools/llm-task.md`

Designed to be embedded inside `.lobster` pipelines, bridging the gap between deterministic shell steps and AI reasoning. Where Lobster steps are fully deterministic shell commands, LLM Task injects a single schema-bound LLM call into the pipeline.

## 4. Command Dispatch — The Bridge Mechanism

The `command-dispatch` frontmatter key in `SKILL.md` files enables skills to bypass the LLM and route slash commands directly to a plugin tool. This is the mechanism that connects the standard skill format to the non-standard runtimes.

```yaml
---
command-dispatch: tool
command-tool: <tool_name>
command-arg-mode: raw
---
```

Source: `src/agents/skills/workspace.ts:723-768`

This is how `/prose` works — the slash command is intercepted before reaching the model and dispatched directly to the OpenProse tool.

## When to Use What

| Use Case | Format | Why |
|---|---|---|
| Static agent instructions | SKILL.md | Simple, no runtime needed |
| Multi-agent creative workflows | OpenProse | LLM orchestration, parallel agents, iterative loops |
| Deterministic automation pipelines | Lobster | Reproducible, auditable, approval gates |
| Single structured LLM step in a pipeline | LLM Task | Schema-validated output inside Lobster |
| Direct tool routing from slash commands | Command Dispatch | Bypass model, zero latency |
