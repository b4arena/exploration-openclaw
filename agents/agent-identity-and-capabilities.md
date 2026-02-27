# Agent Identity, Org Topology, and Capability Expression

> Design research: how b4arena agents declare who they are, what they can do,
> and how that information flows through infrastructure, routing, and peer discovery.
>
> Sources: `b4arena/infra/`, `b4arena/arena/framework/`, `b4arena/ludus/agents/`,
> `b4arena/meta/skills/`. Not derived from OpenClaw source — this is org design
> research layered on top of OpenClaw's agent and skill primitives.

---

## 1. The Problem

OpenClaw provides the runtime (gateway, sandbox, session, skill injection). It does
not prescribe how an organisation structures its agents, groups them, expresses their
capabilities, or routes work between them across multiple hosts.

The b4arena workspace runs agents on two hosts (`mimas`, `rpi5`) with plans to add more. As the
agent roster grows, three gaps emerge:

1. **No org layer** — agents are registered per-host with no grouping above "all agents
   on this host". There is no machine-readable concept of a wing, team, or functional
   domain.

2. **Capabilities are prose** — what an agent can do lives in `SOUL.md` narrative.
   Routing and dispatch tooling cannot consume prose without an LLM.

3. **Identity is fragmented** — cryptographic identity (GPG), deployment placement
   (host + workspace remote), and behavioral spec (SOUL.md) live in three separate
   places with no shared schema linking them.

---

## 2. Core Mental Model

### Wings — organisational units

A **wing** is a functional grouping of related roles. Wings define reporting lines and
communication patterns; they do not define hard execution boundaries.

Defined in `infra/inventory/group_vars/all/wings.yaml`:

| Wing | Purpose | Coordinator |
|---|---|---|
| `leadership` | Human ↔ agent bridge, cross-wing coordination | `chief-of-staff` |
| `product` | Roadmap, user stories, UX | `product-manager` |
| `engineering` | Architecture, development, infrastructure | `engineering-manager` |
| `quality` | Test coverage, release gate | `qa-engineer` |
| `domain` | Domain expertise, documentation | `analyst` |
| `ludus` | Internal tooling, CI, multi-agent coordination | `orchestrator` |

Cross-wing escalations always flow through `leadership` (CoS). No wing talks to
another wing directly — work items cross via the CoS routing layer.

### Hosts — Borg cube compute nodes

`mimas` and `rpi5` are **compute nodes**, not organisational units. Any wing's agents
can run on any node. The org topology (wings) is entirely independent of the compute
topology (hosts).

```
Org layer (wings):      leadership — product — engineering — quality — domain — ludus
                             ↕ all cross through leadership (CoS)

Compute layer (nodes):  mimas ←→ rpi5 ←→ (future nodes)
                             any agent from any wing on any node
```

Conflating these two layers was the original design flaw — agents were "pinned" to
hosts by their deployment identity with no way to express that the same role could
run elsewhere.

### The 4-layer agent identity

```
Layer 1 — Cryptographic   (infra/group_vars/all/agent_identity_<slug>.sops.yml)
  GPG key pair, SOPS-encrypted. Signing and encryption identity.

Layer 2 — Deployment      (infra/host_vars/<host>/main.yml)
  openclaw_agent_name, host placement, workspace remote, wing, role,
  cross_wing flag, escalates_to. Links org identity to compute node.

Layer 3 — Behavioral      (ludus/agents/<name>/SOUL.md + ROLE.yaml)
  SOUL.md: prose persona, values, communication style — injected at session start.
  ROLE.yaml: structured declaration consumed by tooling (see §3).

Layer 4 — Capability      (ludus/skills/<name>/SKILL.md)
  Per-skill specs referenced by ROLE.yaml. Self-describing: includes
  infrastructure requirements (bins, env vars) tooling can provision.
```

Layers 1–2 live in `infra/` (infrastructure concerns). Layers 3–4 live in the
ludus repo (behavioural concerns). They are linked by the agent name slug.

---

## 3. ROLE.yaml — Machine-Readable Identity

`SOUL.md` is for the agent to read. `ROLE.yaml` is for tooling to read. They are
independent layers with independent change cycles.

**Schema** (defined in `ludus/framework/agents.md`):

```yaml
# Required
wing: <wing-name>           # Must match a key in infra/inventory/group_vars/all/wings.yaml
role: <role-name>           # Must match a role slug in that wing's roles list
escalates_to: <target>      # "human" | "<wing>:<role>"
cross_wing: <bool>          # Accepts work items from other wings?

# Capability declaration
skills: []                  # Invocable — each must resolve to a SKILL.md file
competencies: []            # Soft — broader abilities documented in SOUL.md, not invocable
tools: []                   # CLI tools available in sandbox

# Deployment hints
site_requirements: []       # Empty = any node; or list specific hostnames
```

**Implemented examples** — `ludus/agents/`:

| Agent | Wing | Role | cross_wing | Skills |
|---|---|---|---|---|
| `main` | ludus | orchestrator | true | beads-coordination |
| `b4-dev` | ludus | developer | false | pr-creation, beads-coordination, worktree-workflow |
| `b4-eng-mgr` | ludus | engineering-manager | false | task-triage, beads-coordination |

`wings.yaml` is the authoritative source. If a `wing` or `role` value in `ROLE.yaml`
does not match an entry in `wings.yaml`, it is an error.

---

## 4. Capability Expression — Two Tiers

A flat `skills:` list conflates two distinct things:

| Tier | Name | Definition | Invocable? | Backed by |
|---|---|---|---|---|
| 1 | **Skill** | Discrete, packaged capability | Yes — by name | `SKILL.md` must exist |
| 2 | **Competency** | Broader contextual ability | No | Documented in `SOUL.md` |

`skills:` entries must resolve to a `SKILL.md` file. If no SKILL.md exists for a
named skill, it belongs in `competencies:` instead. This invariant is enforceable
by tooling at deploy time.

### The skill derivation chain

Giving an agent the beads capability illustrates how the chain works:

```
ROLE.yaml
  skills: [intercom]
       ↓ resolves to
ludus/skills/intercom/SKILL.md
  frontmatter:
    requires:
      bins: [bd]
      env: [BEADS_DIR]
  body: full operational guide injected into session context
       ↓ consumed by
OpenClaw provisioning:
  → installs `bd` binary in sandbox container
  → injects BEADS_DIR + BD_ACTOR env vars
  → injects SKILL.md body into agent system prompt
```

The SKILL.md is **self-describing**: its frontmatter declares the infrastructure
requirements (binaries, env vars) that OpenClaw needs to provision. ROLE.yaml only
needs to reference the skill by name — no duplication of requirements.

This is the correct pattern for any capability that requires tooling:
write the SKILL.md first (operational spec + `requires` block), then reference it
in ROLE.yaml. The agent, the infrastructure, and other agents all learn what they
need from the single SKILL.md source.

---

## 5. Capability Discovery — Static vs Dynamic

Two discovery modes serve different consumers:

| Mode | Mechanism | Token cost | Consumer |
|---|---|---|---|
| Static | Read `ROLE.yaml` or capabilities registry | Zero | Routing, dispatch, deploy tooling |
| Dynamic | Agent answers via beads comment | LLM tokens | Complex capability negotiation |

The missing piece for static discovery is an **org-level capabilities registry**
auto-generated at deploy time from all `ROLE.yaml` files:

```yaml
# org/capabilities-registry.yaml  (generated — do not edit by hand)
agents:
  - name: main
    wing: ludus
    role: orchestrator
    cross_wing: true
    skills: [intercom]
    tools: [bd, telegram]
    host: mimas
  - name: b4-dev
    wing: ludus
    role: developer
    cross_wing: false
    skills: [intercom, pr-creation, worktree-workflow]
    tools: [gh, git, bd, claude-code]
    host: mimas
```

Generation command: `just gen-registry` — reads all `ROLE.yaml` files across agent
directories and validates skill references against SKILL.md existence.

---

## 6. AGENTS.md — Routing Guide, Not Capability Dump

`AGENTS.md` (present in every repo/agent dir, consumed at session start) is
frequently confused with capability documentation. They answer different questions:

| Artifact | Question answered | Authored by |
|---|---|---|
| `ROLE.yaml` | What can this agent do? (machine) | Human, maintained |
| `SOUL.md` | Who am I? (agent session) | Human, maintained |
| `AGENTS.md` | Who are my peers and when do I call them? (agent session) | Human — **not generated** |
| capabilities registry | Which agents have skill X on node Y? (tooling) | Generated from ROLE.yaml |

---

The "when to involve" column in an AGENTS.md team roster is routing judgment — it
cannot be derived from a skills list. A generated table would be factually correct
but operationally useless to an agent.

**Example of the gap:**

Generated from registry:
```
| main | orchestrator | Skills: intercom | Tools: bd, telegram |
```

Hand-authored AGENTS.md:
```
| main | Orchestrator | Route ALL cross-wing escalations here first.
|      |              | Never bypass to reach the human directly.   |
```

The factual fields (name, role, skills) can optionally be refreshed by tooling.
The "when to involve" annotation must be hand-authored and is the document's
primary value.

`AGENTS.md` stays hand-authored. The capabilities registry serves the machine-readable
use case. These are separate artifacts with separate purposes.

---

## 7. Deployment Identity — Extended Schema

`openclaw_agent_identities` in `infra/host_vars/<host>/main.yml` now carries
the full 4-layer identity:

```yaml
openclaw_agent_identities:
  - agent: "b4-dev"
    # Org identity (Layer 2 — new fields)
    wing: "ludus"
    role: "developer"
    cross_wing: false
    escalates_to: "ludus:engineering-manager"
    # Cryptographic identity (Layer 1)
    gpg_slug: "b4_dev"
    gpg_fingerprint: "<fpr>"
    gpg_email: "b4-dev@b4arena.net"
    # Deployment (Layer 2 — existing)
    workspace_remote: "https://github.com/b4arena/b4-dev-workspace.git"
    workspace_branch: "main"
```

The `create-agent-identity` skill (Phase 1b) now prompts for `wing`, `role`,
`cross_wing`, and `escalates_to` and writes them into both the host inventory
entry and the ROLE.yaml file.

---

## 8. Open Questions

- **Registry generation tooling** — `just gen-registry` is proposed but not
  implemented. What validates skill name → SKILL.md resolution at CI time?

- **Ludus agent directories** — `ludus/framework/agents.md` defines agents in
  prose. They need per-agent directories with `SOUL.md` + `ROLE.yaml` before
  they can be provisioned. Blocked on splitting the monolithic `agents.md`.

- **Cross-site intercom** — beads intercom is currently per-host. Cross-site
  work items (an engineering agent on `rpi5` picking up work from a `mimas`
  bead) require either a shared intercom instance or a federation/bridge layer.
  Shared intercom with wing-prefixed label namespacing (`ludus:dev`, `leadership:cos`)
  is the lower-complexity starting point.

- **Skill filter inheritance** — OpenClaw passes the parent agent's skill snapshot
  to subagents (see `subagent-skills.md`). ROLE.yaml `skills:` provides the
  per-agent filter list for this mechanism, but the provisioning integration
  (reading ROLE.yaml to set the agent's skill filter at registration time) is
  not yet implemented.
