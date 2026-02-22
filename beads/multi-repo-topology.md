# Beads Multi-Repo Topology

Research into how the Beads protocol handles multi-repository workspaces and what topology options exist for a setup with `openclaw/`, `infra/`, `skills/`, and `exploration/` as separate git repositories.

**Sources:**
- `docs/MULTI_REPO_AGENTS.md` — primary agent-facing reference
- `docs/MULTI_REPO_MIGRATION.md` — human setup guide with interactive wizards
- `docs/ADVANCED.md` — database redirects, BEADS_DIR, separate beads repo
- `docs/WORKTREES.md` — worktree topology and shared-database architecture
- `CHANGELOG.md` — feature history for multi-repo routing
- Gas Town `README.md` / `docs/overview.md` — rig-based multi-project orchestration

All paths above are relative to the `steveyegge/beads` or `steveyegge/gastown` repositories.

---

## 1. Core Multi-Repo Primitives

### 1.1 `source_repo` Field

Every issue carries a `source_repo` field in its JSONL record indicating which repository owns it:

```jsonl
{"id":"bd-abc","source_repo":".","title":"Core issue"}
{"id":"bd-xyz","source_repo":"~/.beads-planning","title":"Planning issue"}
{"id":"infra-q3r","source_repo":"/workspace/infra","title":"Infra change"}
```

Values: `.` (current repo), `~/.beads-planning` (contributor planning), or absolute path.

Filter by source in queries:
```bash
bd list --json | jq '.[] | select(.source_repo == "/workspace/infra")'
```

### 1.2 Multi-Repo Hydration (`repos.additional`)

Beads aggregates issues from multiple repos into a single unified SQLite view:

```bash
bd config set repos.primary "."
bd config set repos.additional "/workspace/infra,/workspace/skills/my-skill"
```

After hydration, `bd list`, `bd ready`, `bd dep tree` all operate across all configured repos. The `source_repo` field preserves provenance — issues are never copied, only read into a local SQLite cache.

**How hydration works:**
1. Reads JSONL from each configured repo
2. Imports into unified SQLite at `.beads/beads.db`
3. Maintains `source_repo` for provenance
4. Routes new issues back to correct JSONL files on creation

### 1.3 Auto-Routing

```bash
bd config set routing.mode auto          # detect role
bd config set routing.maintainer "."     # where maintainer issues go
bd config set routing.contributor "~/.beads-planning"  # contributor fallback

# Or explicit (always use a fixed repo)
bd config set routing.mode explicit
bd config set routing.default "."

# Per-command override
bd create "infra change" -p 1 --repo /workspace/infra
```

Role detection order: (1) `git config beads.role`, (2) SSH vs HTTPS remote URL, (3) fallback = contributor.

Auto-routing is **disabled by default** since CHANGELOG entry `#1177` — explicit opt-in required.

### 1.4 Discovered-Issue Inheritance

Issues created with `--deps discovered-from:PARENT-ID` automatically inherit the parent's `source_repo`. This is intentional: a bug discovered while working in `openclaw/` stays attributed to `openclaw/`. Override with `--repo`:

```bash
bd create "infra cert expiry" -p 1 --deps discovered-from:oc-abc --repo /workspace/infra
```

### 1.5 Cross-Repo Dependencies

Dependencies can span repos. `bd dep add impl-42 plan-10 --type blocks` links issues from different databases once hydrated. Blocking is resolved across the full hydrated view.

### 1.6 Database Redirects

For multi-agent setups with multiple clones of the *same* repo, a `.beads/redirect` file points secondary clones at a shared database:

```bash
echo "../main-clone/.beads" > .beads/redirect
```

**Limitations:** Single-level only (no chaining). This is for same-repo multi-clone scenarios, not cross-project.

### 1.7 BEADS_DIR — External Beads Repository

The `BEADS_DIR` environment variable points `bd` at a `.beads` directory outside the current git repo. This enables a fully standalone beads repository:

```bash
export BEADS_DIR=~/my-project-beads/.beads
bd create "task" -t task   # stored in ~/my-project-beads, not in current repo
bd sync                    # commits to ~/my-project-beads git repo
```

`bd sync` detects "External BEADS_DIR" and targets git operations at the beads repo, never touching the code repo.

---

## 2. MCP Server Routing

A **single** `beads-mcp` instance handles all projects:

```json
{
  "beads": {
    "command": "beads-mcp",
    "args": []
  }
}
```

The MCP server:
- Detects workspace from the agent's current working directory
- Routes to the correct per-project daemon (`.beads/bd.sock`)
- Manages a connection pool keyed by canonicalized path
- Resolves symlinks to prevent duplicate connections
- Treats git submodules with their own `.beads/` as separate projects

**Do not use multiple MCP server instances** — this causes incorrect server selection by agents.

---

## 3. `bd ready` Across Repos

`bd ready` operates on the fully hydrated database. Once `repos.additional` is configured, `bd ready` shows unblocked work from all repos, respecting cross-repo blocking relationships. Filter by repo:

```bash
bd ready --json | jq '.[] | select(.source_repo == ".")'
```

---

## 4. Gas Town Rig Model

Gas Town (`steveyegge/gastown`) treats each git repository as a **rig**:

```bash
gt rig add openclaw https://github.com/org/openclaw.git
gt rig add infra    https://github.com/org/infra.git
```

Each rig has its own crew, polecats (worker agents), and beads ledger. The **Mayor** (central coordinator) orchestrates across rigs via **convoys** — bundles of beads assigned across rigs:

```bash
gt convoy create "Deploy feature X" oc-abc12 infra-def34 --notify
gt sling oc-abc12 openclaw   # assign bead to openclaw rig
gt sling infra-def34 infra   # assign bead to infra rig
```

**Cross-rig work patterns:**
- **Worktrees**: agent from one rig checks out and commits in another rig
- **Dispatch**: task assigned to target rig's team; naming convention `gastown/crew/joe` preserves identity

**Per-rig `.beads/`**: Each rig has its own `.beads/` directory (per the rig-per-repo architecture). The Mayor aggregates visibility via the Gas Town town root (`hq-` prefix) and cross-rig routing via `routes.jsonl`.

---

## 5. `routes.jsonl` and Prefix Routing

Gas Town introduces `routes.jsonl` at the town root for prefix-based routing:

```json
{"prefix":"oc","path":"./openclaw"}
{"prefix":"infra","path":"./infra"}
{"prefix":"hq","path":"."}
```

With prefix routing, `bd show infra-q3r` from any rig resolves to the `infra/` repo's database. `bd close` and `bd update` also use prefix routing so cross-rig operations work without `--repo` flags.

---

## 6. Topology Options for Our Workspace

### Option A: Per-Repo `.beads/` with Hydration (Recommended for Teams)

```
workspace/
├── openclaw/    ← .beads/ here (maintainer mode, committed to repo)
├── infra/       ← .beads/ here (maintainer mode, committed to repo)
├── skills/      ← .beads/ per skill repo (if needed)
└── exploration/ ← no .beads/ (docs-only repo)
```

**Configuration** (in `openclaw/`):
```bash
bd config set repos.primary "."
bd config set repos.additional "/workspace/infra,/workspace/skills/my-skill"
```

**Pros:**
- Git ledger per repo — issues travel with the code they describe
- Team members pulling `infra/` get its full issue history
- `bd list` from `openclaw/` shows everything when hydrated

**Cons:**
- Agents must be cwd-aware (MCP auto-routes, but agents must use correct cwd)
- Cross-repo epics require manual linking via `bd dep add`
- `.beads/issues.jsonl` needs coordination in PRs

### Option B: Standalone Shared Beads Repo

```
workspace/
├── openclaw/    ← no .beads/
├── infra/       ← no .beads/
├── skills/      ← no .beads/
├── exploration/ ← no .beads/
└── beads/       ← dedicated git repo, all issues here
    └── .beads/
        ├── beads.db
        └── issues.jsonl
```

**Setup:**
```bash
cd /workspace/beads && git init && bd init --prefix ws
# In any other repo:
export BEADS_DIR=/workspace/beads/.beads
```

**Or via direnv** (per workspace root):
```bash
echo 'export BEADS_DIR=/workspace/beads/.beads' > /workspace/.envrc
direnv allow /workspace
```

**Pros:**
- Clean separation: code history not polluted by issue tracking
- All agents share one database regardless of cwd
- Single `bd sync` commits all issues to one place
- Works well if the repos are private/internal

**Cons:**
- Issues are not co-located with the code they describe — harder for external contributors
- Requires `BEADS_DIR` to be set in every shell / agent session
- No per-repo `bd list` — everything is in one flat namespace (use `source_repo` tags)
- Standalone repo must be kept separate from `exploration/` (different concerns)

### Option C: Gas Town Town with Multiple Rigs

```
~/gt/                         ← Gas Town town root
├── openclaw/                 ← rig (git clone of openclaw)
│   └── .beads/
├── infra/                    ← rig (git clone of infra)
│   └── .beads/
├── routes.jsonl              ← prefix routing table
└── .beads/                   ← town-level HQ beads (hq- prefix)
    └── issues.jsonl
```

**Setup:**
```bash
gt install ~/gt --git
gt rig add openclaw https://github.com/org/openclaw.git
gt rig add infra    https://github.com/org/infra.git
```

**Pros:**
- Native multi-agent orchestration built in (Mayor, Polecats, Convoys)
- Cross-rig work (cross-repo epics) tracked via convoys
- `routes.jsonl` enables transparent cross-rig `bd show/close/update`
- Identity and attribution preserved across rigs

**Cons:**
- Gas Town is a separate heavy dependency (adds mayor/deacon/witness/refinery roles)
- Rigs are clones managed by Gas Town — conflicts with our existing workspace layout
- Overkill if the goal is just cross-repo issue visibility without full orchestration

### Option D: Workspace-Root `.beads/` (Monorepo Workaround)

```
workspace/          ← bd init here
├── .beads/         ← single shared database at workspace root
├── openclaw/       ← no .beads/
├── infra/          ← no .beads/
└── exploration/    ← no .beads/
```

**Setup:**
```bash
cd /workspace && bd init --prefix ws
```

`bd` walks up the directory tree (like git) to find `.beads/`. All repos share the workspace-root database.

**Pros:**
- Simplest setup — one `bd init`, everything visible
- No `BEADS_DIR` needed
- Cross-repo epics trivially in one database

**Cons:**
- Issues are not committed to the repo they describe (they go to `workspace/`)
- `workspace/` is the `exploration/` repo — mixing issue tracking into docs repo is semantically wrong
- If `openclaw/` is worked on standalone (outside this workspace), it has no `.beads/`
- Violates "git ledger" principle — issues not co-located with code

---

## 7. Practical Scenarios

### Scenario A: Agent in `openclaw/` creates bead requiring `infra/` changes

```bash
# In openclaw/ (primary)
bd create "Add TLS cert rotation" -p 1 -t epic  # → oc-abc

# Infra sub-task, routed to infra/
bd create "Provision cert in Vault" -p 1 --deps blocks:oc-abc --repo /workspace/infra
# → infra-xyz (source_repo = "/workspace/infra")

# View from openclaw/ (hydrated)
bd dep tree oc-abc  # Shows oc-abc blocked by infra-xyz
```

### Scenario B: Cross-repo epic spanning `openclaw/` and `infra/`

**Option 1:** Parent lives in `openclaw/`, children in each repo with `blocks` deps.

**Option 2:** Parent lives in the standalone `beads/` repo or workspace-root `.beads/` — neutral ground.

**Option 3:** Gas Town convoy bundles both beads into one tracked work unit.

Recommendation: **Option 1** — parent in the repo that owns the feature outcome (`openclaw/`), children linked cross-repo.

### Scenario C: Agent needs to see all open work across all repos

With **Option A** (per-repo + hydration):
```bash
# From openclaw/ with repos.additional set
bd ready --json   # aggregated view, all repos
```

With **Option B** (standalone beads repo):
```bash
# From any cwd
BEADS_DIR=/workspace/beads/.beads bd ready --json
```

With **Option D** (workspace root):
```bash
cd /workspace && bd ready --json   # or any subdirectory
```

---

## 8. Decision: Option B — Standalone Beads Repo

**Decided: Option B (standalone shared Beads repo with `BEADS_DIR` via direnv)**

### Context: The Software Forge Model

The workspace is a **software forge** ("Softwareschmiede") coordinated by OpenClaw. Multiple roles (agents) work across multiple projects:

| Role | Repo Checkout? | Needs Beads? |
|---|---|---|
| Product Manager | No | Yes — creates epics, prioritizes |
| Architect | Sometimes (read-only) | Yes — design decisions, reviews |
| Engineering Manager | No | Yes — status overview, assignments |
| Developer | Yes (1-2 repos) | Yes — tasks, communication |
| DevOps | Yes (infra) | Yes — deployment tasks |
| QA | Yes (read-only) | Yes — bug reports, test results |

**Key insight:** Roles like PM and Engineering Manager have no code repo checkout. Per-repo `.beads/` with hydration (Option A) requires a cwd inside a repo — this fundamentally doesn't work for roles without repo access.

### Beads vs. GitHub Issues: Separate Worlds

Beads is **internal Slack** — agent-to-agent and agent-human coordination and communication. It does not replace a ticket system.

| System | Purpose | Lifecycle |
|---|---|---|
| **Beads** | Internal coordination, task dispatch, agent chat | Ephemeral — beads can be cleaned up after completion |
| **GitHub Issues** | Official project tracking, external documentation | Permanent — part of the project record |

No automatic sync between the two. Agents reference GitHub Issues in beads manually where needed (e.g., "see GH#42").

### Rationale

1. **Roles without repos** — PM, Eng. Manager need Beads access without any code checkout
2. **Single polling point** — Tier 1 watcher polls one location, zero cwd management
3. **Clean separation** — code history not polluted by coordination chatter
4. **Zero scaling cost** — new projects don't require Beads config changes (just use labels)
5. **Ephemeral nature** — Beads are conversations, not artifacts; they don't need to travel with code

### Workspace Layout

```
/workspace/                     ← Software forge root
├── beads/                      ← Standalone git repo (the nervous system)
│   └── .beads/
│       ├── beads.db
│       └── issues.jsonl
│
├── projects/                   ← Code repos (only roles that need them)
│   ├── project-alpha/          ← GitHub repo with Issues
│   ├── project-beta/
│   └── infra/
│
└── .envrc                      ← export BEADS_DIR=/workspace/beads/.beads
```

### Minimal Setup

```bash
# Create the beads repo
mkdir -p /workspace/beads && cd /workspace/beads
git init && bd init --prefix ws

# Set up direnv at workspace root
echo 'export BEADS_DIR=/workspace/beads/.beads' > /workspace/.envrc
direnv allow /workspace

# Single MCP server in claude_desktop_config.json
# { "beads": { "command": "beads-mcp", "args": [] } }

# Verify from any directory
cd /workspace/projects/project-alpha
bd where  # → /workspace/beads/.beads
```

### Tier 1 Watcher Routing (Labels)

```bash
# Watcher groups by label, wakes matching agent:
# labels: ["project-alpha", "frontend"]  → frontend-dev agent
# labels: ["epic", "project-beta"]       → pm agent
# labels: ["deploy", "infra"]            → devops agent
# labels: []  (no label)                 → eng-manager agent (triage)
```

---

## 9. Key Commands Reference

```bash
# Config
bd config set repos.additional "~/repo1,~/repo2"
bd config set routing.mode auto
bd config set routing.maintainer "."
bd config get repos.additional
bd info --json   # shows full config + daemon status

# Cross-repo routing
bd create "issue" -p 1 --repo /workspace/infra
bd show infra-xyz         # resolved via prefix routing
bd dep add oc-abc infra-xyz --type blocks

# Visibility
bd where                  # which database is active
bd ready --json           # unblocked work across all hydrated repos
bd list --json | jq '.[] | select(.source_repo == "/workspace/infra")'

# External beads repo
export BEADS_DIR=/workspace/beads/.beads
bd create "cross-repo task" -p 1
bd sync  # commits to beads repo only
```
