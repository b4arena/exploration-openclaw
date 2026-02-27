# b4arena

Multi-agent agentic company — AI agents that plan, build, test, and ship an eSports data platform, steered by a human who spends max 1 hour/day on decisions.

## The Structure

The architecture follows the Roman Colosseum model: two inseparable buildings connected by a tunnel.

**The Colosseum** (`arena/`) — the organizational framework: constitution, operating model, role definitions, and the rules that govern how agents collaborate.

**The Ludus Magnus** (`ludus/`) — where agents are made: SOULs, GPG identities, deployment configs, and the skills registry. Agents enter the arena through the *crypta* — session-start injection of their SOUL and ROLE.

**The crypta** — the tunnel between them. When SOUL.md and ROLE.yaml are loaded at session-start, the agent is no longer a definition in a file — it is an active participant in the framework.

## The Agents

13 agents across 5 wings, all reporting through Apex (Chief of Staff) to the human Editor:

```
              [ Editor: Human, 1h/day ]
                          │
                 [ Apex — Leadership Wing ]
                (Chief of Staff, runs the Ludus)
                          │
      ┌───────────┬────────┼────────┬──────────┐
      │           │        │        │          │
   Product   Engineering  Quality  Domain   Leadership
   ──────    ──────────   ───────  ──────
   Priya     Atlas        Probe    Senna
   Maren     Rio                   Quill
             Vite
             Forge
             Muse
             Hertz
             Helm
```

## Exploration

Our research into agent architectures, coordination patterns, and deployment strategies is public:

- [exploration-openclaw](https://github.com/b4forge/exploration-openclaw) — architecture deep-dives, agent communication patterns, and deployment research
