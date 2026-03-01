# b4arena

Multi-agent agentic company — AI agents that plan, build, test, and ship an eSports data platform, steered by humans who each spend max 1 hour/day on decisions.

## The Structure

The architecture follows the Roman Colosseum model: two inseparable buildings connected by a tunnel.

**The Colosseum** (`arena/`) — the organizational framework: constitution, operating model, role definitions, and the rules that govern how agents collaborate.

**The Ludus Magnus** (`ludus/`) — where agents are made: SOULs, GPG identities, deployment configs, and the skills registry. Agents enter the arena through the *crypta* — session-start injection of their SOUL and ROLE.

**The crypta** — the tunnel between them. When SOUL.md and ROLE.yaml are loaded at session-start, the agent is no longer a definition in a file — it is an active participant in the framework.

## The Agents

Each human runs an independent node with their own Apex (Chief of Staff) and agent fleet. Nodes coordinate through GitHub.

```
  Human A                            Human B
      │                                 │
      ▼                                 ▼
  [ Apex — CoS ]                   [ Apex — CoS ]
      │                                 │
      ├── Product Wing                  ├── Product Wing
      ├── Engineering Wing              ├── Engineering Wing
      ├── Quality Wing                  ├── Quality Wing
      └── Domain Wing                   └── Domain Wing
      │                                 │
      └────────── GitHub ──────────────┘
               Issues + PRs
            (shared work repos)
```

13 agents across 5 wings per node: Priya (PM), Maren (UX), Atlas (Architect), Rio (EM), Vite, Forge, Muse, Hertz, Helm (Engineering), Probe (QA), Senna, Quill (Domain).

## Exploration

Our research into agent architectures, coordination patterns, and deployment strategies is public:

- [exploration-openclaw](https://github.com/b4arena/exploration-openclaw) — architecture deep-dives, agent communication patterns, and deployment research
